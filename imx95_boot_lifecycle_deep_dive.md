# NXP i.MX95 EVK — Boot Lifecycle & Build Ecosystem: Deep-Dive Technical Reference

> **Target Platform:** NXP i.MX95 (i.MX9592) EVK · ARMv9-A (Cortex-A55 cluster) + Cortex-M33 (System Manager) + Cortex-M7 (optional RTOS domain)  
> **Android Release:** AOSP Android 15 (API 35) / Android Automotive OS  
> **Toolchain:** LLVM/Clang 17 (Android), GCC 12.x arm-none-eabi (M33/M7 firmware)  
> **Build System:** AOSP `m` / Soong + NXP BSP Yocto (kirkstone/scarthgap layer)

---

## Bảng Mục Lục

1. [Toàn Cảnh Chuỗi Khởi Động](#1-toàn-cảnh-chuỗi-khởi-động)
   - 1.1 [Tổng Quan Kiến Trúc SoC & Exception Levels](#11-tổng-quan-kiến-trúc-soc--exception-levels)
   - 1.2 [Giai Đoạn 0: Power-On Reset & BootROM](#12-giai-đoạn-0-power-on-reset--bootrom)
   - 1.3 [Giai Đoạn 1: SPL + DDR PHY Init](#13-giai-đoạn-1-spl--ddr-phy-init)
   - 1.4 [Giai Đoạn 2: ATF BL31 & PSCI/SMC Setup](#14-giai-đoạn-2-atf-bl31--pscismc-setup)
   - 1.5 [Giai Đoạn 3: EdgeLock Enclave (ELE) & M33 System Manager](#15-giai-đoạn-3-edgelock-enclave-ele--m33-system-manager)
   - 1.6 [Giai Đoạn 4: U-Boot Proper](#16-giai-đoạn-4-u-boot-proper)
   - 1.7 [Giai Đoạn 5: Linux Kernel Boot](#17-giai-đoạn-5-linux-kernel-boot)
   - 1.8 [Giai Đoạn 6: Android Init (Phase 1 & 2) + SELinux](#18-giai-đoạn-6-android-init-phase-1--2--selinux)
   - 1.9 [Giai Đoạn 7: Zygote → System Server → Launcher](#19-giai-đoạn-7-zygote--system-server--launcher)
2. [Chi Điểm Source Code & Log Minh Họa](#2-chi-điểm-source-code--log-minh-họa)
3. [Quá Trình Biên Dịch Phân Mảnh (Modular Build)](#3-quá-trình-biên-dịch-phân-mảnh-modular-build)
4. [Output Artifacts & Cơ Chế Flashing](#4-output-artifacts--cơ-chế-flashing)

---

## 1. Toàn Cảnh Chuỗi Khởi Động

### 1.1 Tổng Quan Kiến Trúc SoC & Exception Levels

i.MX95 (SM8695-based, TSMC N4P) tích hợp kiến trúc HMP (Heterogeneous Multi-Processing) với nhiều execution domain tách biệt:

```
┌─────────────────────────────────────────────────────────────────────┐
│                         NXP i.MX95 SoC                              │
│                                                                      │
│  ┌─────────────────────┐   ┌──────────────────────────────────────┐ │
│  │  Cortex-A55 Cluster │   │       Cortex-M33 (System Manager)   │ │
│  │  (6x cores @ 1.8GHz)│   │  - AUTOSAR-like SM firmware         │ │
│  │                     │   │  - Power/Clock/Voltage management    │ │
│  │  EL3: ATF BL31      │   │  - SCMI server (System Control &    │ │
│  │  EL2: Hypervisor/   │   │    Management Interface)            │ │
│  │       KVM (optional)│   │  - Runs from TCM (64KB) / OCRAM     │ │
│  │  EL1: Linux Kernel  │   └──────────────────────────────────────┘ │
│  │  EL0: Userspace     │                                             │
│  └─────────────────────┘   ┌──────────────────────────────────────┐ │
│                             │    EdgeLock Enclave (ELE / S400)    │ │
│  ┌─────────────────────┐   │  - ARMv8-M Cortex-M33 (dedicated)  │ │
│  │  Cortex-M7 (opt.)   │   │  - Root of Trust, HAB v5 successor  │ │
│  │  FreeRTOS / Zephyr  │   │  - Secure key storage, TRNG        │ │
│  │  MCUX SDK           │   │  - OTP/eFuse management            │ │
│  └─────────────────────┘   └──────────────────────────────────────┘ │
│                                                                      │
│  ┌─────────────────────────────────────────────────────────────────┐ │
│  │  Memory Subsystem: LPDDR5 (32-bit / 64-bit) via DDR PHY       │ │
│  │  Storage: eMMC 5.1 (HS400) / FlexSPI (OSPI NOR/NAND)         │ │
│  └─────────────────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────────────┘
```

**Exception Level Transition Map:**

```
Power-On Reset
    │
    ▼
[ELE Firmware] ─────────────────────────────────────── (Secure Enclave, isolated)
    │
    ▼
[BootROM] (EL3, AArch64)
    │  Loads → authenticates SPL from eMMC boot0/OSPI
    ▼
[SPL / imx-SPL] (EL3, AArch64)
    │  Init DDR PHY → Loads ATF BL31 + U-Boot Proper + OP-TEE (optional)
    ▼
[ATF BL31] (EL3, AArch64) ─── permanent resident, handles SMC calls
    │  PSCI setup → hands off to U-Boot at EL2 (or EL1 if no hypervisor)
    ▼
[U-Boot Proper] (EL1/EL2, AArch64)
    │  Loads kernel Image + DTB → executes booti
    ▼
[Linux Kernel] (EL1, AArch64) ─── ATF remains at EL3 as SMC handler
    │  Decompresses → MMU on → subsystem init → init_task
    ▼
[Android Init PID=1] (EL0, AArch64)
    │  Mounts partitions → loads SELinux → starts services
    ▼
[Zygote → System Server → Launcher] (EL0)
```

---

### 1.2 Giai Đoạn 0: Power-On Reset & BootROM

#### Hardware Reset Vector

Khi VDD_MAIN được cấp, Power Management IC (PMIC PF09) assert `POR_B` sau khi voltage rails ổn định. i.MX95 BootROM được map tại địa chỉ cố định:

```
BootROM Base Address: 0x0000_0000 (aliased from internal ROM)
Reset Vector (AArch64): 0x0000_0000 → ldr x0, =rom_entry
BootROM Size: 96KB (internal mask ROM)
```

#### Boot Device Selection (BOOT_MODE pins)

```
BOOT_MODE[1:0] = 2'b10  → Internal Boot (từ fuse/eFuse BOOT_CFG)
BOOT_MODE[1:0] = 2'b00  → Boot from fuses (production)
BOOT_MODE[1:0] = 2'b01  → Serial Downloader (USB OTG → imx_usb_loader)

BOOT_CFG fuse map (OTP bank 1, word 3):
  Bits [3:0]   = Boot device: 0x0=eMMC, 0x1=SD, 0x2=FlexSPI NOR
  Bits [7:4]   = eMMC bus width: 0x0=1-bit, 0x1=4-bit, 0x2=8-bit
  Bit  [8]     = eMMC boot partition: 0=boot0, 1=boot1
  Bits [11:9]  = FlexSPI clock: 000=30MHz, 001=50MHz, 010=100MHz
```

#### BootROM Boot Container (i.MX9 Boot Image Format)

i.MX95 sử dụng **Boot Container** format mới (không còn IVT/DCD của i.MX8), được định nghĩa trong NXP AN13694:

```
┌──────────────────────────────────────────────────────────┐
│  Boot Container (eMMC boot0, offset 0x0)                 │
│                                                           │
│  ┌─────────────────────────────┐  Offset 0x000           │
│  │  Container Header           │                          │
│  │  tag: 0x87 (CONTAINER_TAG)  │                          │
│  │  version: 0x00              │                          │
│  │  flags: HAB_flags           │                          │
│  │  num_images: N              │                          │
│  │  sig_offset: ptr_to_sign    │                          │
│  └─────────────────────────────┘                          │
│  ┌─────────────────────────────┐                          │
│  │  Image Entry[0]: SPL        │                          │
│  │  load_addr: 0x2049_0000     │  (OCRAM)                 │
│  │  entry_point: 0x2049_0000   │                          │
│  │  hab_flags: 0x1 (encrypted?)│                          │
│  │  hab_hash: SHA384 digest    │                          │
│  └─────────────────────────────┘                          │
│  ┌─────────────────────────────┐                          │
│  │  Image Entry[1]: ELE FW     │                          │
│  │  load_addr: 0x0            │  (ELE internal SRAM)     │
│  └─────────────────────────────┘                          │
│  ┌─────────────────────────────┐                          │
│  │  SRK Table (4x SRK hashes) │  Super Root Key          │
│  │  SRK1_hash[63:0]           │                          │
│  │  SRK2_hash[63:0]           │                          │
│  │  SRK3_hash[63:0]           │                          │
│  │  SRK4_hash[63:0]           │                          │
│  └─────────────────────────────┘                          │
│  ┌─────────────────────────────┐                          │
│  │  Signature (ECDSA P-384 or  │                          │
│  │  RSA-4096) over header +    │                          │
│  │  image entries              │                          │
│  └─────────────────────────────┘                          │
└──────────────────────────────────────────────────────────┘
```

#### Xác Thực Bảo Mật: EdgeLock Enclave (ELE) vs HAB

i.MX95 thay thế HAB (High Assurance Boot) của i.MX8 bằng **EdgeLock Enclave (S400 subsystem)**:

```
i.MX8:  BootROM → HAB4 → Authenticate với SRK fuses (SHA256)
i.MX95: BootROM → ELE FW → Authenticate với SRK fuses (SHA384/SHA512)
                         → Key Management Unit (KMU)
                         → OTFAD (On-The-Fly AES Decryption, nếu encrypted boot)
```

**ELE Authentication Flow:**

```
1. BootROM nạp ELE firmware vào ELE internal SRAM (từ Boot Container)
2. ELE khởi động độc lập, thực hiện self-test (TRNG, crypto engine)
3. BootROM gửi IPC message đến ELE: "authenticate image at addr X"
4. ELE đọc SRK hash từ eFuse bank (burned vào OTP)
5. ELE verify ECDSA/RSA signature của Boot Container header
6. ELE verify hash của từng image entry (SHA384 over image data)
7. Nếu pass: ELE trả lời BootROM "authenticated", set secure state
8. Nếu fail: ELE ghi security violation vào SNVS_HPSR register,
             SoC có thể brick (nếu SEC_CONFIG[1]=1 trong fuse)
```

**Register liên quan:**

```
SNVS_HPSR (HP Status Register): 0x4049_0014
  Bit[15:14] = SSM_STATE: 00=Init, 01=HardFail, 10=Soft Fail,
                           11=Init Intermediate, 100=Check,
                           1000=Non-Secure, 1011=Trusted, 1111=Secure

ELE_STATUS register (via MU - Messaging Unit):
  Accessible at: 0x4700_0000 (S3MUA - Secure MU for ELE)
```

#### Log BootROM (UART1, 115200 8N1):

```
# i.MX95 BootROM không in ra UART trong production mode.
# Chỉ in trong Serial Downloader mode hoặc debug ROM:

HAB status check (sau khi U-Boot load, gọi hab_status):
```

```
U-Boot> hab_status

Secure Boot Configuration Status
---------------------------------
ELE FW Version:     0x07 0x02 (v7.2)
ELE Status:         0x00000000
Lifecycle Status:   0xC (CLOSED - Secure Boot enabled)

HAB/ELE Event Count: 0

hab_auth_status: OK (no violations)
```

---

### 1.3 Giai Đoạn 1: SPL + DDR PHY Init

#### SPL Load & Execution

BootROM nạp SPL vào OCRAM (On-Chip RAM) tại `0x2049_0000`. SPL là một subset của U-Boot được biên dịch với `CONFIG_SPL=y`.

**Source tree:**

```
u-boot/
├── arch/arm/mach-imx/imx9/
│   ├── soc.c              ← imx9_soc_init(), clock_init()
│   └── clock_root.c       ← configure_clock_root() via CCM_ROOT registers
├── board/freescale/imx95_evk/
│   ├── spl.c              ← board_early_init_f(), spl_dram_init()
│   └── imx95_evk.c        ← board_init(), board_late_init()
├── drivers/ddr/imx/
│   ├── ddr_init.c         ← ddr_init() wrapper
│   └── lpddr5/
│       ├── ddr_phy_train.c ← CTL/PHY firmware training sequence
│       └── imx95_evk_lpddr5_timing.c ← Board-specific timing parameters
└── common/spl/
    └── spl.c              ← spl_load_image(), board_boot_order()
```

**Hàm cốt lõi SPL execution flow:**

```c
/* arch/arm/lib/crt0_64.S */
ENTRY(_start)
    /* EL3 entry, setup stack pointer */
    adr     x0, __bss_start
    adr     x1, __bss_end
    bl      board_init_f_alloc_reserve   /* Reserve GD (global data) */

/* arch/arm/lib/board.c */
void board_init_f(ulong dummy)
{
    /* 1. Setup GD (global_data struct at top of SRAM) */
    gd = (gd_t *)CONFIG_SYS_INIT_SP_ADDR;
    
    /* 2. Early serial init (UART1) */
    preloader_console_init();           /* calls serial_init() */
    
    /* 3. SoC-specific clock/power init */
    board_early_init_f();               /* → imx9_soc_init() */
    
    /* 4. DDR initialization */
    spl_dram_init();
}

/* board/freescale/imx95_evk/spl.c */
void spl_dram_init(void)
{
    struct dram_timing_info *dram_timing = &dram_timing_b0;  /* LPDDR5 timing */
    
    /* Configure DDR Controller (DDRC) */
    ddr_init(dram_timing);
    
    /* DDR PHY Training: IMEM/DMEM firmware load → PIE execution */
    /* DDRC base: 0x4E30_0000 */
    /* DDR PHY base: 0x4E2E_0000 */
}
```

#### DDR PHY Training Chi Tiết

i.MX95 sử dụng Synopsys DWC LPDDR5 Controller + PHY. Training sequence:

```c
/* drivers/ddr/imx/lpddr5/ddr_phy_train.c */

int ddr_phy_train(struct dram_timing_info *dtiming)
{
    /* Step 1: Load 1D PHY Firmware (IMEM) */
    /* IMEM base: DDR_PHY_BASE + 0x50000 */
    ddr_phy_load_firmware(PHY_FIRMWARE_1D_IMEM, imem_1d, IMEM_1D_SIZE);
    
    /* Step 2: Load 1D PHY Data (DMEM) - timing parameters */
    ddr_phy_load_firmware(PHY_FIRMWARE_1D_DMEM, dmem_1d, DMEM_1D_SIZE);
    
    /* Step 3: Execute 1D training (Write Leveling, Rd/Wr DQ deskew) */
    ddr_phy_start_training();           /* PIE[0] = 0x1 → trigger */
    ddr_phy_wait_training_done();       /* Poll PIE[0x00d0] == 0x07 */
    
    /* Step 4: Load 2D PHY Firmware (Read/Write Eye training) */
    ddr_phy_load_firmware(PHY_FIRMWARE_2D_IMEM, imem_2d, IMEM_2D_SIZE);
    ddr_phy_load_firmware(PHY_FIRMWARE_2D_DMEM, dmem_2d, DMEM_2D_SIZE);
    
    /* Step 5: Execute 2D training */
    ddr_phy_start_training();
    ddr_phy_wait_training_done();
    
    /* Step 6: Load PIE (PHY Init Engine) production code */
    ddr_phy_load_firmware(PHY_FIRMWARE_PIE, pie_data, PIE_SIZE);
    
    return 0;
}

/* Register access pattern: */
/* PHY CSR (Control/Status Registers) accessed via 16-bit APB interface */
/* Addr mapping: PHY_BASE + (msg_block_offset << 1) */
```

**DDR PHY Timing Parameters (board-specific):**

```c
/* drivers/ddr/imx/lpddr5/imx95_evk_lpddr5_timing.c */
struct dram_timing_info dram_timing_b0 = {
    .ddrc_cfg = {
        /* DDRC_MSTR: 0x4E300000 */
        { 0x4E300000, 0xA1080010 },  /* MSTR: lpddr5, BL=16, geardown */
        /* DDRC_RFSHTMG: tREFI = 3904 (3.9us @ 1GHz) */
        { 0x4E300064, 0x006200F4 },
        /* DDRC_INIT3: MR1/MR2 LPDDR5 mode registers */
        { 0x4E3000DC, 0x00C40006 },
        /* ... 80+ register pairs for full timing config */
    },
    .ddrphy_cfg = {
        /* Synopsys PHY CSR values post-training */
        { 0x0002_0010, 0x0001 },     /* MemResetL */
        { 0x0002_0060, 0x02A5 },     /* ATestCsMode */
        /* ... */
    },
    .fsp_msg = {
        /* Frequency Set Point messages (1D/2D training inputs) */
        [0] = {
            .drate = 6400,           /* LPDDR5-6400: 3200 MT/s per pin × 2 */
            .MR1 = 0x44,             /* BL=16, WR preamble */
            .MR2 = 0x36,             /* RL=36, WL=18 */
            .MR3 = 0xF1,             /* PDDS=7, DBI-RD, DBI-WR off */
            .MR11 = 0x04,            /* CA ODT=80ohm */
            .MR13 = 0x00,
            .MR22 = 0x16,            /* ODTE-CS, ODTD-CA */
        },
    },
};
```

**SPL Serial Log:**

```
U-Boot SPL 2024.04-lf-6.6.23-2.0.0+gc9a2441 (Mar 15 2025 - 10:22:31 +0000)
DDRINFO: start DRAM init
DDRINFO: LPDDR5 training config: 6400 MT/s
DDRINFO: 1D training start...
DDRINFO: 1D training done, phyinit complete
DDRINFO: 2D training start...
DDRINFO: 2D training done
DDRINFO: init_freq set to 3200000000
Normal Boot
Trying to boot from MMC1 Boot Partition 1

image offset 0x8000, pagesize 0x200, ivt offset 0x0
NOTICE:  BL31: v2.10.0(release):lf-6.6.23-2.0.0-0-gd6c5394
NOTICE:  BL31: Built : 10:22:28, Mar 15 2025
```

---

### 1.4 Giai Đoạn 2: ATF BL31 & PSCI/SMC Setup

#### ATF (ARM Trusted Firmware) trên i.MX95

SPL nạp ATF BL31 vào OCRAM tại địa chỉ dành riêng. BL31 chạy vĩnh viễn ở EL3 và xử lý SMC (Secure Monitor Calls).

**Source tree:**

```
trusted-firmware-a/
├── plat/imx/imx95/
│   ├── platform.mk         ← Build configuration, BL31 memory map
│   ├── imx95_bl31_setup.c  ← bl31_platform_setup(), imx95_pwr_domain_init()
│   ├── imx95_psci.c        ← imx95_pwr_domain_on/off/suspend()
│   └── imx95_scmi.c        ← SCMI transport over MU (Messaging Unit)
├── include/plat/imx/
│   └── imx95_def.h         ← Memory map, MU base addresses
├── lib/psci/
│   ├── psci_main.c         ← psci_cpu_on(), psci_system_suspend()
│   └── psci_stat.c
└── services/std_svc/
    └── std_svc_setup.c     ← SMC dispatcher: PSCI / SCMI / OP-TEE
```

**BL31 Memory Map trên i.MX95:**

```c
/* plat/imx/imx95/include/platform_def.h */

/* BL31 resides in OCRAM Secure region */
#define BL31_BASE               UL(0x204E0000)
#define BL31_LIMIT              UL(0x20500000)  /* 128KB reserved */

/* EL3 Runtime Firmware stack */
#define BL31_STACK_BASE         (BL31_LIMIT - 0x1000)

/* Non-secure BL33 (U-Boot) entry point */
#define PLAT_NS_IMAGE_OFFSET    UL(0x80200000)  /* DDR: U-Boot entry */

/* PSCI CPU ON entry (warm reset vector) */
#define PLAT_WARM_RESET_ADDR    UL(0x204E0040)
```

**SMC Interface (SMCCC ABI):**

```c
/* services/std_svc/std_svc_setup.c */
/* SMC Function ID format: 
   Bits[31]: 0=ARM32 SMC, 1=ARM64 SMC
   Bits[30]: 0=Fast call, 1=Yielding call (legacy)
   Bits[29:24]: Service range
     0x00 = ARM Architecture calls
     0x01 = CPU Service (PSCI)
     0x02 = SiP Service (NXP-specific) ← i.MX95 custom
     0x04 = Standard Service (SCMI over SMC)
*/

/* PSCI Function IDs (ARM DEN0022E): */
#define PSCI_CPU_SUSPEND_AARCH64    0xC4000001
#define PSCI_CPU_OFF                0x84000002
#define PSCI_CPU_ON_AARCH64         0xC4000003
#define PSCI_AFFINITY_INFO_AARCH64  0xC4000004
#define PSCI_SYSTEM_OFF             0x84000008
#define PSCI_SYSTEM_RESET           0x84000009
#define PSCI_SYSTEM_RESET2_AARCH64  0xC4000012

/* NXP SiP SMC (platform-specific, range 0xC2000000-0xC200FFFF): */
#define IMX_SIP_GPC             0xC2000000  /* GPC (General Power Controller) */
#define IMX_SIP_CPUIDLE         0xC2000001
#define IMX_SIP_DDR_DVFS        0xC2000004  /* DDR frequency scaling */
#define IMX_SIP_NOC_PRIORITY    0xC2000008
```

**PSCI CPU_ON Flow (khi Linux kernel boot secondary cores):**

```c
/* plat/imx/imx95/imx95_psci.c */
int imx95_pwr_domain_on(u_register_t mpidr)
{
    unsigned int core_id = MPIDR_AFFLVL0_VAL(mpidr);  /* Cortex-A55 core 0-5 */
    
    /* 1. Ghi warm boot entry vào SRC (System Reset Controller) */
    /* SRC_GPR base: 0x44460000 */
    mmio_write_32(SRC_GPR(core_id), (uint32_t)imx_mailbox_entry);
    
    /* 2. Gọi M33 System Manager qua SCMI để power up core domain */
    scmi_core_start(core_id);
    
    /* 3. Release core từ trạng thái WFE/reset hold */
    /* GPC_CPU_PD (General Power Controller) */
    mmio_setbits_32(GPC_CPU_PD_BASE + core_id * 0x800, GPC_PD_CTRL_OFF_M);
    
    return PSCI_E_SUCCESS;
}
```

**ATF Serial Log:**

```
NOTICE:  BL31: v2.10.0(release):imx95-bl31-2024.10
NOTICE:  BL31: Built : 10:22:28, Mar 15 2025
INFO:    GICv4 with ITS detected, VLPI supported
INFO:    GIC redistributor base=0x48060000
INFO:    BL31: Initializing runtime services
INFO:    BL31: cortex_a55: CPU workaround for erratum 1530923 was applied
INFO:    BL31: cortex_a55: CPU workaround for erratum 2658417 was applied
INFO:    EL3: SCMI channel initialized, transport: MU3
INFO:    PSCI: Suspending system, target power state:
INFO:    BL31: Preparing for EL3 exit to normal world
INFO:    Entry point address = 0x80200000
INFO:    SPSR = 0x3c9
```

---

### 1.5 Giai Đoạn 3: EdgeLock Enclave (ELE) & M33 System Manager

#### EdgeLock Enclave (ELE / S400)

ELE là một processor độc lập (Cortex-M33 core riêng biệt với Cortex-A55) với mục đích bảo mật. Firmware của ELE được NXP cung cấp dưới dạng binary blob.

```
ELE Firmware location trong build:
  firmware/imx/ele/
  └── mx95a0-ahab-container.img  ← ELE firmware binary (NXP signed)

ELE được BootROM nạp TRƯỚC khi SPL chạy.
ELE firmware binary được đặt trong Boot Container Image.

ELE Services (via MU - Messaging Unit tại 0x4700_0000):
  - ELE_GET_INFO          : Firmware version, lifecycle
  - ELE_GENERATE_KEY      : Key generation (ECDSA, RSA, AES)
  - ELE_LOAD_KEY          : Load key from keystore
  - ELE_ENCRYPT/DECRYPT   : AES-GCM, ChaCha20
  - ELE_SIGN/VERIFY       : ECDSA P-256/P-384
  - ELE_HASH              : SHA256/384/512
  - ELE_ATTEST            : Attestation token generation
  - ELE_GET_RANDOM        : True RNG output
  - ELE_OTP_WRITE         : One-Time Programmable write (irreversible)
  - ELE_SAB_INIT          : Security Audit Block init
```

**SCMI over MU (M33 System Manager):**

i.MX95 sử dụng Cortex-M33 riêng (không phải ELE) làm **System Manager (SM)** chạy firmware AUTOSAR-inspired để quản lý resources.

```
SM Firmware source: 
  github.com/nxp-imx/imx-sm  (MCUXpresso SDK based)
  
SM Communication Protocol: SCMI (System Control & Management Interface)
  Transport: Messaging Unit (MU) - shared memory doorbell mechanism
  MU Base addresses:
    MU1 (A55 ↔ M33): 0x44230000 (non-secure) / 0x44240000 (secure)
    MU2: 0x44250000
    MU3 (ATF ↔ SM): 0x44260000
    MU5 (ELE ↔ SM): 0x44270000

SCMI Protocols used by Linux/ATF:
  0x10: Base protocol
  0x11: Power Domain Management
  0x12: System Power Management
  0x13: Performance (DVFS)
  0x14: Clock Management
  0x15: Sensor
  0x17: Voltage Domain
  0x80: NXP BBM (Battery Backed Module - RTC/Watchdog)
  0x81: NXP MISC (misc platform services)
```

**SM Firmware Memory Layout:**

```
/* M33 executes from: */
ITCM base:  0x0FFE_0000  (64KB, instruction TCM)
DTCM base:  0x2000_0000  (64KB, data TCM)
OCRAM:      0x2004_0000  (SM shared with A55 for SCMI buffers)

/* SCMI Shared Memory (shmem): */
MU1 SHMEM base: 0x2004_F000  (4KB, A55 ↔ SM channel)
```

**M33 SM Firmware Load (SPL side):**

```c
/* board/freescale/imx95_evk/spl.c */
void load_m33_image(void)
{
    /* M33 firmware đã được include trong flash.bin và nằm ở
       partition riêng hoặc embedded trong boot container */
    
    /* Load M33 fw vào ITCM qua SRC (System Reset Controller) */
    /* SRC_M33: 0x44460800 */
    
    /* 1. Assert M33 reset */
    mmio_setbits_32(SRC_M33_CTRL, SRC_CTRL_SW_RESET);
    
    /* 2. Copy .text vào ITCM, .data vào DTCM */
    memcpy((void *)M33_ITCM_BASE, m33_fw_text, m33_fw_text_size);
    memcpy((void *)M33_DTCM_BASE, m33_fw_data, m33_fw_data_size);
    
    /* 3. Set M33 boot address */
    mmio_write_32(SRC_M33_CTRL + 0x04, M33_ITCM_BASE);
    
    /* 4. De-assert reset → M33 starts executing */
    mmio_clrbits_32(SRC_M33_CTRL, SRC_CTRL_SW_RESET);
}
```

---

### 1.6 Giai Đoạn 4: U-Boot Proper

#### U-Boot Relocation & Board Init

Sau khi ATF trả điều khiển về EL1 (hoặc EL2), U-Boot Proper bắt đầu thực thi tại `0x8020_0000` trong DDR.

```
u-boot/
├── arch/arm/lib/
│   ├── crt0_64.S           ← ENTRY(_start), EL setup, call board_init_f
│   └── board.c             ← board_init_f(), initcall_run_list()
├── common/
│   ├── board_r.c           ← board_init_r(), initr_*() functions
│   ├── bootm.c             ← do_bootm(), bootm_load_os()
│   └── image.c             ← image_get_os(), boot container parsing
├── cmd/
│   ├── booti.c             ← do_booti() → raw Linux Image boot
│   └── bootflow.c          ← bootflow scan/boot (distroboot)
├── board/freescale/imx95_evk/
│   ├── imx95_evk.c         ← board_init(), board_late_init()
│   └── imx95_evk_ddr_timing.h
└── drivers/
    ├── mmc/fsl_esdhc_imx.c ← eMMC/SD driver (HS400 mode)
    └── net/fec_mxc.c       ← FEC Ethernet (for TFTP boot)
```

**initcall序列 (board_init_f):**

```c
/* common/board_f.c */
static const init_fnc_t init_sequence_f[] = {
    setup_mon_len,
    initf_malloc,
    log_init,
    initf_bootstage,
    event_init,
    setup_spl_handoff,      /* Nhận handoff info từ SPL */
    arch_cpu_init,          /* arch/arm/mach-imx/imx9/soc.c */
    mach_cpu_init,
    initf_dm,               /* Driver Model init (minimal) */
    board_early_init_f,     /* UART, clock, power config */
    timer_init,             /* ARM generic timer, CNTFRQ */
    board_init_f_init_reserve,
    env_init,               /* Locate ENV: eMMC offset 0x400000 */
    init_baud_rate,         /* 115200 bps */
    serial_init,
    console_init_f,
    display_options,        /* Print U-Boot banner */
    display_text_info,
    checkcpu,               /* "CPU: i.MX95 rev1.0 at xxxMHz" */
    print_cpuinfo,
    show_board_info,        /* "Board: i.MX95 EVK" */
    misc_init_f,
    INIT_FUNC_WATCHDOG_INIT
    announce_dram_init,
    dram_init,              /* Get DDR size from DDRC registers */
    dram_init_banksize,
    setup_relocaddr,        /* Calculate relocation address (top DDR) */
    reserve_round_4k,
    setup_machine,
    reserve_global_data,
    reserve_fdt,            /* Reserve DTB space */
    reserve_bootstage,
    reserve_bloblist,
    setup_dram_config,
    show_dram_config,       /* "DRAM: 8 GiB" */
    setup_reloc,
    NULL,
};
```

**DTB Loading & bootargs Setup:**

```bash
# U-Boot environment (stored in eMMC at mmcblk0, offset 0x400000):

mmcdev=1
mmcpart=1
bootpart=1:1  # boot partition on eMMC

# Distroboot scan sequence:
boot_targets=mmc0 mmc1 usb0 pxe dhcp

# bootcmd for Android:
bootcmd=run android_bootcmd

android_bootcmd=
  mmc dev ${mmcdev};
  setenv bootargs "androidboot.hardware=nxp_evk95 \
    androidboot.dtbo_idx=0 \
    androidboot.verifiedbootstate=orange \
    androidboot.serialno=${serial#} \
    earlycon=ec_imx6q,0x44380000,115200 \
    console=ttymxc1,115200 \
    init=/init \
    loglevel=4 \
    androidboot.selinux=permissive";
  
  # Load boot.img (GKI format)
  part start mmc ${mmcdev} boot boot_start;
  part size  mmc ${mmcdev} boot boot_size;
  mmc read ${loadaddr} ${boot_start} ${boot_size};
  
  # Android boot image parse & jump
  bootm ${loadaddr};
```

**`do_booti()` → Kernel Handoff:**

```c
/* cmd/booti.c */
static int do_booti(struct cmd_tbl *cmdtp, int flag, int argc, char *const argv[])
{
    struct bootm_headers images = { 0 };
    
    /* Parse raw Linux Image header (Magic: 0x644D5241 "ARM\x64") */
    ret = booti_setup(images.ep, &fake_header, &images, argc > 2);
    
    /* Setup FDT (Flattened Device Tree) */
    /* DTB address passed as second argument or from U-Boot DM */
    images.ft_addr = (char *)dtb_addr;
    
    /* Final handoff: */
    /* x0 = FDT blob address */
    /* x1 = 0 (reserved) */
    /* x2 = 0 (reserved) */
    /* x3 = 0 (reserved) */
    /* PC = kernel Image entry point (TEXT_OFFSET = 0 in v5.8+) */
    kernel_entry(0, 0, 0, images.ft_addr);  /* jump! */
}
```

**U-Boot Serial Log:**

```
U-Boot 2024.04-lf-6.6.23-2.0.0+gc9a2441 (Mar 15 2025 - 10:22:31 +0000)

CPU:   NXP i.MX95 (0x95850010) rev1.1 at 1800 MHz
Model: NXP i.MX95 EVK board
DRAM:  8 GiB (LPDDR5-6400)
Core:  149 devices, 32 uclasses, devicetree: board
WDT:   Not starting watchdog@42490000
MMC:   FSL_SDHC: 0, FSL_SDHC: 1, FSL_SDHC: 2
Loading Environment from MMC... OK
In:    serial@44380000
Out:   serial@44380000
Err:   serial@44380000
Net:   eth0: ethernet@4ac10000 [PRIME]
Hit any key to stop autoboot:  0
switch to partitions #0, OK
mmc1(part 0) is current device
Scanning for Android bootflow...
** Android image booting: kernel@0x80200000, fdt@0x83000000, ramdisk@0x84000000 **
Kernel command line: "androidboot.hardware=nxp_evk95 ... earlycon=ec_imx6q,0x44380000,115200"
## Booting Android Image at 0x80200000 ...
   Image Name:   
   Image Type:   AArch64 Linux Kernel Image (not compressed)
   Data Size:    29360128 Bytes = 28 MiB
   Load Address: 00000000
   Entry Point:  00000000
Starting kernel ...
```

---

### 1.7 Giai Đoạn 5: Linux Kernel Boot

#### Entry Point: `head.S` → `start_kernel()`

```
arch/arm64/kernel/
├── head.S              ← Absolute entry point (_start / _head)
├── head-common.S       ← shared macros (el2_setup, set_cpu_boot_mode_flag)
├── vmlinux.lds.S       ← Linker script (Text offset, sections layout)
├── setup.c             ← setup_arch() → early_fixmap, memblock, DT parse
├── smp.c               ← smp_init(), secondary_start_kernel()
└── process.c           ← cpu_idle_poll()

init/
├── main.c              ← start_kernel() → rest_init()
└── initramfs.c         ← populate_rootfs() (nếu initrd/ramdisk)
```

**`arch/arm64/kernel/head.S` - Chi tiết luồng thực thi:**

```asm
/* arch/arm64/kernel/head.S */

    /* Linux Kernel Image Header (64 bytes): */
    .quad   0x14000010      /* b primary_entry (branch over magic) */
    .quad   0               /* text_offset = 0 (v5.8+ ABI) */
    .quad   _kernel_size_le /* image_size */
    .quad   TEXT_OFFSET     /* flags */
    .quad   0               /* reserved */
    .quad   0               /* reserved */
    .quad   0               /* reserved */
    .ascii  "ARM\x64"       /* magic = 0x644D5241 */
    .long   0               /* reserved (PE/COFF offset) */

primary_entry:
    bl      preserve_boot_args      /* Lưu x0(DTB), x1-x3 vào boot_args[] */
    bl      init_kernel_el          /* Setup EL1 → SCTLR_EL1, HCR_EL2 */
    adrp    x23, __PHYS_OFFSET
    bl      set_cpu_boot_mode_flag  /* __boot_cpu_mode = BOOT_CPU_MODE_EL1/EL2 */
    
    /* Map kernel sections: .text, .rodata, .data, .bss */
    bl      __create_page_tables   /* Setup initial page tables (TTBR0/TTBR1) */
    
    /*
     * Switch on MMU:
     * SCTLR_EL1[M] = 1
     * TTBR0_EL1 = VA:PA identity map (trampoline)
     * TTBR1_EL1 = kernel VA map (0xFFFF_0000_0000_0000+)
     */
    bl      __primary_switch       /* Enable MMU, jump to __primary_switched */

__primary_switched:
    /* Now running with MMU ON, kernel virtual addresses valid */
    adr_l   x4, init_task          /* init_task (PID 0, swapper) struct */
    init_cpu_task x4, x5, x6      /* Setup SP = init_task.stack top */
    
    /* Zero BSS */
    adr_l   x0, __bss_start
    adr_l   x1, __bss_stop
    bl      __pi_memset
    
    /* Jump to C code */
    bl      start_kernel           /* init/main.c */
```

**`start_kernel()` Execution Chain:**

```c
/* init/main.c */
void __init __no_sanitize_address start_kernel(void)
{
    /* EL1, MMU on, caches on (partially) */
    
    set_task_stack_end_magic(&init_task);   /* Canary for stack overflow */
    smp_setup_processor_id();              /* CPU0 affinity */
    
    pr_notice("%s", linux_banner);
    /* Prints: "Linux version 6.6.23-android15-8-..." */
    
    early_fixmap_init();                   /* FixMap VA region setup */
    early_ioremap_init();
    
    setup_arch(&command_line);             /* arch/arm64/kernel/setup.c */
    /* → unflatten_device_tree()           Parse DTB → device_node tree */
    /* → arm64_memblock_init()             Setup memblock allocator */
    /* → paging_init()                     Setup full page tables */
    /* → acpi_table_init() / of_scan_flat_dt() */
    
    build_all_zonelists(NULL);
    page_alloc_init();
    
    pr_notice("Kernel command line: %s\n", saved_command_line);
    
    parse_early_param();                   /* Parse "earlycon=", "loglevel=" etc */
    
    trap_init();                           /* Exception vectors (vectors label in entry.S) */
    mm_init();                             /* Buddy allocator, SLUB/SLAB init */
    
    sched_init();                          /* Scheduler, runqueues, CFS */
    
    rcu_init();                            /* Read-Copy-Update */
    
    early_irq_init();
    init_IRQ();                            /* GICv3/GICv4 init (plat/arm/gic-v3.c) */
    
    tick_init();
    rcu_init_nohz();
    init_timers();
    hrtimers_init();
    softirq_init();
    timekeeping_init();                    /* Clocksource: ARM arch timer, 52MHz */
    
    /* Boot CPU secondary cores (PSCI CPU_ON) */
    smp_prepare_cpus(setup_max_cpus);
    
    rest_init();                           /* ← Critical: spawn PID 1 */
}

void __init __noreturn rest_init(void)
{
    /* Spawn kernel_init thread (will become PID 1) */
    pid = kernel_thread(kernel_init, NULL, CLONE_FS);
    
    /* Spawn kthreadd (PID 2, kernel thread manager) */
    pid = kernel_thread(kthreadd, NULL, CLONE_FS | CLONE_FILES);
    
    /* PID 0 (swapper/idle) enters idle loop */
    cpu_startup_entry(CPUHP_ONLINE);       /* → do_idle() */
}

static int __init kernel_init(void *unused)
{
    /* Run all initcalls: core_initcall → postcore → arch → subsys → 
                          fs → device → late */
    do_initcalls();
    
    /* Mount rootfs (ramdisk / system partition) */
    /* For Android: ramdisk is the first_stage_ramdisk from vendor_boot.img */
    if (ramdisk_execute_command) {
        /* execute /init from ramdisk */
        ret = run_init_process(ramdisk_execute_command);
    }
    
    /* Fallback: try /sbin/init, /etc/init, /bin/init, /bin/sh */
    try_to_run_init_process("/init");      /* → Android Init! */
}
```

**i.MX95 Specific Platform Initcalls:**

```c
/* drivers/clk/imx/clk-imx95.c */
static int __init imx95_clk_init(void)
{
    /* Register CCM (Clock Controller Module) clock providers */
    /* CCM base: 0x44450000 */
    /* 300+ clock nodes: PLL, root clock, clock gates */
    of_clk_add_provider(np, of_clk_src_onecell_get, &clk_data);
}
arch_initcall(imx95_clk_init);

/* drivers/pinctrl/freescale/pinctrl-imx95.c */
static int imx95_pinctrl_probe(struct platform_device *pdev)
{
    /* IOMUXC base: 0x443C0000 */
    /* 350+ pin multiplexing registers */
    imx_pinctrl_probe(pdev, &imx95_pinctrl_info);
}

/* sound/soc/fsl/fsl_sai.c */
static int fsl_sai_probe(struct platform_device *pdev)
{
    /* SAI1-8 (Synchronous Audio Interface) */
    /* SAI3 base (TAS5828 path): 0x443B0000 */
    devm_snd_soc_register_component(&pdev->dev,
        &fsl_component, fsl_sai_dai, ARRAY_SIZE(fsl_sai_dai));
}
```

**dmesg Log (kernel boot, i.MX95 specific):**

```
[    0.000000] Booting Linux on physical CPU 0x0000000000 [0x411fd050]
[    0.000000] Linux version 6.6.23-android15-8-00001-g3abc1234 (gcc version 12.3.0, GNU ld 2.38)
[    0.000000] Machine model: NXP i.MX95 19x19 EVK board
[    0.000000] earlycon: ec_imx6q0 at MMIO 0x0000000044380000 (options '115200')
[    0.000000] printk: bootconsole [ec_imx6q0] enabled
[    0.000000] efi: UEFI not found.
[    0.000000] NUMA: No NUMA configuration found
[    0.000000] NUMA: Faking a node at [mem 0x0000000080000000-0x000000027fffffff]
[    0.000000] Zone ranges:
[    0.000000]   DMA      [mem 0x0000000080000000-0x00000000bfffffff]
[    0.000000]   DMA32    [mem 0x00000000c0000000-0x00000000ffffffff]
[    0.000000]   Normal   [mem 0x0000000100000000-0x000000027fffffff]
[    0.000000] Movable zone start for each node
[    0.000000] Early memory node ranges
[    0.000000]   node   0: [mem 0x0000000080000000-0x000000027fffffff]
[    0.000000] Initmem setup node 0 [mem 0x0000000080000000-0x000000027fffffff]
[    0.000000] cma: Reserved 256 MiB at 0x0000000094000000
[    0.000000] psci: probing for conduit method from DT.
[    0.000000] psci: PSCIv1.1 detected in firmware.
[    0.000000] psci: Using standard PSCI v0.2 function IDs
[    0.000000] psci: Trusted OS migration not required
[    0.000000] psci: SMC Calling Convention v1.4
[    0.000000] GICv4: Distributor has no Range Selector support
[    0.000000] GIC: Using split EOI/Deactivate mode
[    0.000000] arch_timer: cp15 timer(s) running at 24.00MHz (virt).
[    0.000000] clocksource: arch_sys_counter: mask: 0xffffffffffffff max_cycles: 0x588fe9dc0, max_idle_ns: 440795202592 ns
[    0.000000] sched_clock: 56 bits at 24MHz, resolution 41ns, wraps every 4398046511097ns
[    0.045000] Console: colour dummy device 80x25
[    0.045000] Calibrating delay loop (skipped), value calculated using timer frequency.. 48.00 BogoMIPS
[    0.046000] pid_max: default: 32768 minimum: 301
[    0.047000] Mount-cache hash table entries: 16384 (order: 5, 131072 bytes, linear)
[    0.139000] rcu: Hierarchical SRCU implementation.
[    0.201000] EFI services will not be available.
[    0.202000] smp: Bringing up secondary CPUs ...
[    0.203000] Detected PIPT I-cache on CPU1
[    0.203000] CPU1: Booted secondary processor 0x0000000001 [0x411fd050]
[    0.204000] CPU2: Booted secondary processor 0x0000000002 [0x411fd050]
[    0.205000] CPU3: Booted secondary processor 0x0000000003 [0x411fd050]
[    0.206000] CPU4: Booted secondary processor 0x0000000004 [0x411fd050]
[    0.207000] CPU5: Booted secondary processor 0x0000000005 [0x411fd050]
[    0.207000] smp: Brought up 1 node, 6 CPUs
[    0.207000] SMP: Total of 6 processors activated.
[    0.208000] CPU features: detected: ARMv8.2 LVA
[    0.208000] CPU features: detected: Spectre-v2
[    0.208000] CPU features: detected: Spectre-BHB
[    0.550000] imx-scmi-misc scmi_dev.3: NXP SM Platform: i.MX95 A0 (0x00950000)
[    0.551000] imx-scmi-misc scmi_dev.3: SM firmware: 2025.03.15
[    0.620000] imx8mp-pinctrl 443c0000.pinctrl: initialized IMX pinctrl driver
[    0.720000] fsl-sai 443b0000.sai: SAI3 driver probed (TAS5828 path)
[    0.725000] fsl-sai 44370000.sai: SAI1 driver probed
[    1.100000] input: gpio-keys as /devices/platform/gpio-keys/input/input0
[    1.245000] nxp-fec 4ac10000.ethernet eth0: registered PHC device 0
```

---

### 1.8 Giai Đoạn 6: Android Init (Phase 1 & 2) + SELinux

#### First Stage Init

Android Init chia làm 2 giai đoạn (first_stage và second_stage). Kernel exec `/init` từ ramdisk (first_stage_ramdisk trong `vendor_boot.img`).

```
system/core/init/
├── main.cpp                ← main() - dispatch to FirstStageMain or SecondStageMain
├── first_stage_init.cpp    ← FirstStageMain()
├── first_stage_mount.cpp   ← DoFirstStageMount() - mount super, etc.
├── init.cpp                ← SecondStageMain()
├── init_parser.cpp         ← ParseConfig() - *.rc file parser
├── service.cpp             ← Service::Start()
├── builtins.cpp            ← Built-in commands: mount, exec, etc.
└── selinux.cpp             ← LoadSelinuxPolicy(), SelinuxSetupKernelLogging()
```

**First Stage Init Flow:**

```cpp
/* system/core/init/main.cpp */
int main(int argc, char** argv) {
    /* argv[0] determines mode */
    if (!strcmp(basename(argv[0]), "ueventd")) return ueventd_main(argc, argv);
    if (argc > 1) {
        if (!strcmp(argv[1], "subcontext")) return SubcontextMain(argc, argv, ...);
        if (!strcmp(argv[1], "selinux_setup")) return SetupSelinux(argv);
        if (!strcmp(argv[1], "second_stage")) return SecondStageMain(argc, argv);
    }
    return FirstStageMain(argc, argv);  /* Default: first stage */
}

/* system/core/init/first_stage_init.cpp */
int FirstStageMain(int argc, char** argv) {
    /* 1. Mount essential tmpfs/devtmpfs */
    CHECKCALL(mount("tmpfs",  "/dev",  "tmpfs", ...));
    CHECKCALL(mount("devpts", "/dev/pts", "devpts", ...));
    CHECKCALL(mount("proc",   "/proc", "proc", ...));
    CHECKCALL(mount("sysfs",  "/sys",  "sysfs", ...));
    CHECKCALL(mount("selinuxfs", "/sys/fs/selinux", "selinuxfs", ...));
    
    /* 2. Setup logging to kmsg */
    SetStdioToDevNull(argv);
    InitKernelLogging(argv);
    
    LOG(INFO) << "init first stage started!";
    
    /* 3. Load kernel modules (first stage) */
    LoadKernelModules(IsRecoveryMode() ? ...);
    
    /* 4. Mount APEX (Android 10+): early APEX for first_stage */
    /* APEX mounts provide /apex/com.android.runtime → art, libc++ etc. */
    if (!DoFirstStageMounting()) {
        LOG(FATAL) << "Failed to mount partitions";
    }
    
    /* 5. re-exec as selinux_setup stage */
    const char* path = "/system/bin/init";
    const char* args[] = { path, "selinux_setup", nullptr };
    execv(path, const_cast<char**>(args));
}
```

**SELinux Policy Loading:**

```cpp
/* system/core/init/selinux.cpp */
int SetupSelinux(char** argv) {
    /* Load SELinux policy into kernel */
    LoadSelinuxPolicy(policy);
    
    /* Policy sources: precompiled (GKI) or built from source */
    /* Location on device: /vendor/etc/selinux/precompiled_sepolicy */
    /* Or: compile from /system/etc/selinux + /vendor/etc/selinux */
    
    selinux_android_set_sehandle(sehandle);
    
    /* Switch to enforcing/permissive based on cmdline */
    /* androidboot.selinux=permissive → permissive mode */
    
    /* re-exec as second_stage */
    const char* args[] = { "/system/bin/init", "second_stage", nullptr };
    execv(args[0], const_cast<char**>(args));
}
```

**Second Stage Init + .rc parsing:**

```cpp
/* system/core/init/init.cpp */
int SecondStageMain(int argc, char** argv) {
    /* 1. Setup property system */
    PropertyInit();
    
    /* 2. Setup epoll for event loop */
    Epoll epoll;
    
    /* 3. Parse init.rc and device-specific .rc files */
    /* Load order: */
    /*   /system/etc/init/hw/init.rc (main init.rc) */
    /*   /system/etc/init/*.rc */
    /*   /vendor/etc/init/*.rc  ← OEM services (HALs, daemons) */
    /*   /odm/etc/init/*.rc */
    ActionManager& am = ActionManager::GetInstance();
    ServiceList& sm = ServiceList::GetInstance();
    LoadBootScripts(am, sm);
    
    /* 4. Execute "early-init" trigger */
    am.QueueEventTrigger("early-init");
    /* → ueventd starts → /dev populated with device nodes */
    
    /* 5. Execute "init" trigger */
    am.QueueEventTrigger("init");
    /* → Mount /data, /cache, etc. */
    /* → Start vold (Volume Daemon) */
    /* → Start keymaster HAL */
    
    /* 6. Execute "late-init" trigger → "boot" trigger */
    am.QueueEventTrigger("late-init");
    /* → "class_start core" → start core services */
    /* → "class_start main" → start main services */
    
    /* 7. Event loop */
    while (true) {
        auto epoll_timeout = std::optional<std::chrono::milliseconds>{};
        am.ExecuteOneCommand();
        if (!IsShuttingDown()) {
            auto next_process_action_time = HandleProcessActions();
        }
        epoll.Wait(epoll_timeout);
    }
}
```

**Key init.rc Service Definitions (Android Automotive):**

```rc
# system/core/rootdir/init.rc (excerpt)

on init
    # Start essential services
    start ueventd
    start logd
    start servicemanager

on late-init
    trigger early-fs
    trigger fs
    trigger post-fs
    trigger late-fs
    trigger post-fs-data
    trigger load_persist_props_action
    trigger start-hal-task
    trigger zygote-start

on zygote-start && property:ro.crypto.state=encrypted && property:ro.crypto.type=file
    wait_for_prop odsign.verification.done 1
    exec_start wait_for_keymaster
    start zygote
    start zygote_secondary

# vendor/etc/init/android.hardware.audio.service.rc
service vendor.audio-hal /vendor/bin/hw/android.hardware.audio.service
    class hal
    user audioserver
    group audio camera drmrpc inet media mediadrm net_bt net_bt_admin net_bw_acct wakelock context_hub
    capabilities BLOCK_SUSPEND SYS_NICE
    ioprio rt 4
    task_profiles ProcessCapacityHigh HighPerformance
    onrestart restart audioserver
    socket audio_us_hal_uds stream 0660 audioserver audioserver

# vendor/etc/init/android.hardware.automotive.audiocontrol-service.rc
service audiocontrol-default /vendor/bin/hw/android.hardware.automotive.audiocontrol-service.nxp
    class hal
    user audioserver
    group audio
```

**logcat Output (Init Phase):**

```
01-01 00:00:01.123  1  1 I init    : init first stage started!
01-01 00:00:01.234  1  1 I init    : Loaded kernel module: /lib/modules/fsl_enet.ko
01-01 00:00:01.345  1  1 I init    : Loaded kernel module: /lib/modules/imx-sdma.ko
01-01 00:00:01.456  1  1 I init    : First stage mount success
01-01 00:00:01.567  1  1 I init    : [libfs_avb] Waiting for metadata...
01-01 00:00:02.123  1  1 I init    : Loading SELinux policy
01-01 00:00:02.345  1  1 I init    : SELinux policy loaded, security_setenforce(1)
01-01 00:00:02.456  1  1 I init    : init second stage started!
01-01 00:00:02.567  1  1 I init    : Parsing file /system/etc/init/hw/init.rc...
01-01 00:00:02.678  1  1 I init    : Parsing file /vendor/etc/init/hw/init.imx95evk.rc...
01-01 00:00:02.789  1  1 I servicemanager: ServiceManager: 0 services registered.
01-01 00:00:03.100  1  1 I init    : starting service 'zygote'...
```

---

### 1.9 Giai Đoạn 7: Zygote → System Server → Launcher

#### Zygote Fork Model

```
frameworks/base/
├── core/jni/
│   └── AndroidRuntime.cpp          ← AndroidRuntime::start()
├── cmds/app_process/
│   └── app_main.cpp                ← main() → AppRuntime::onZygoteInit()
└── core/java/com/android/internal/os/
    ├── ZygoteInit.java             ← ZygoteInit.main()
    ├── ZygoteServer.java           ← ZygoteServer.runSelectLoop()
    └── Zygote.java                 ← Zygote.forkAndSpecialize()
```

**Zygote Startup:**

```cpp
/* frameworks/base/cmds/app_process/app_main.cpp */
int main(int argc, char* const argv[]) {
    /* argv: app_process /system/bin --zygote --start-system-server */
    
    AppRuntime runtime(argv[0], computeArgBlockSize(argc, argv));
    
    bool zygote = false;
    bool startSystemServer = false;
    
    while (i < argc) {
        if (strcmp(argv[i], "--zygote") == 0) zygote = true;
        if (strcmp(argv[i], "--start-system-server") == 0) startSystemServer = true;
    }
    
    if (zygote) {
        runtime.start("com.android.internal.os.ZygoteInit",
                      args,          /* "--start-system-server" */
                      zygote);
    }
}

/* frameworks/base/core/jni/AndroidRuntime.cpp */
void AndroidRuntime::start(const char* className, ...) {
    /* 1. Start JVM (ART) */
    JNI_CreateJavaVM(&mJavaVM, &env, &initArgs);
    
    /* 2. Register JNI methods for Android framework */
    register_jni_procs(gRegJNI, NELEM(gRegJNI), env);
    
    /* 3. Call ZygoteInit.main() via JNI */
    jmethodID startMeth = env->GetStaticMethodID(startClass, "main", "([Ljava/lang/String;)V");
    env->CallStaticVoidMethod(startClass, startMeth, strArray);
}
```

**ZygoteInit.java - Pre-loading & System Server Fork:**

```java
/* frameworks/base/core/java/com/android/internal/os/ZygoteInit.java */
public static void main(String argv[]) {
    /* 1. Preload classes & resources (CRITICAL for fork speed) */
    preload(bootTimingsTraceLog);
    /* → preloadClasses()   : Load ~6000 classes from /system/etc/preloaded-classes */
    /* → preloadResources() : Load framework-res.apk drawables, layouts */
    /* → nativePreloadAppProcessHALs() : Load audio/graphics HAL handles */
    
    /* 2. Create Zygote socket server */
    zygoteServer = new ZygoteServer(isPrimaryZygote);
    /* Socket: /dev/socket/zygote (listened by system_server for app forks) */
    
    /* 3. GC before forking (reduce copy-on-write pages) */
    gcAndFinalize();
    
    /* 4. Fork SystemServer */
    if (startSystemServer) {
        forkSystemServer(abiList, zygoteSocketName, zygoteServer);
    }
    
    /* 5. Zygote event loop: wait for fork requests */
    caller = zygoteServer.runSelectLoop(abiList);
    caller.run();
}

private static Runnable forkSystemServer(...) {
    /* System server args */
    String args[] = {
        "--setuid=1000",          /* system uid */
        "--setgid=1000",
        "--setgroups=1001,1002,1003,...",
        "--capabilities=...",
        "--nice-name=system_server",
        "--runtime-args",
        "--target-sdk-version=10000",
        "com.android.server.SystemServer",
    };
    
    /* Fork! */
    pid = Zygote.forkSystemServer(uid, gid, gids, runtimeFlags, ...);
    
    if (pid == 0) {
        /* Child: execute SystemServer.main() */
        return handleSystemServerProcess(parsedArgs);
    }
    return null;  /* Parent Zygote continues */
}
```

**SystemServer → Service Initialization:**

```java
/* frameworks/base/services/java/com/android/server/SystemServer.java */
public static void main(String[] args) {
    new SystemServer().run();
}

private void run() {
    /* 1. Init Looper for main thread */
    Looper.prepareMainLooper();
    
    /* 2. Load native services library */
    System.loadLibrary("android_servers");
    
    /* 3. Init system context */
    createSystemContext();
    
    /* 4. Start critical services */
    startBootstrapServices(t);
    /* → ActivityManagerService (AMS) */
    /* → PackageManagerService (PMS) - parse all APKs */
    /* → PowerManagerService */
    /* → RecoverySystemService */
    /* → LightsService */
    
    /* 5. Start core services */
    startCoreServices(t);
    /* → BatteryService */
    /* → UsageStatsService */
    /* → WebViewUpdateService */
    
    /* 6. Start other services */
    startOtherServices(t);
    /* → WindowManagerService */
    /* → InputManagerService */
    /* → NetworkManagementService */
    /* → AudioService (AudioPolicyService wrapper) */
    /* → CarService (Android Automotive!) */
    
    /* 7. Signal AMS that system is ready */
    ActivityManagerService.self().systemReady(() -> {
        startSystemUi(context, windowManagerF);
    });
    
    Looper.loop();
}
```

**CarService (Android Automotive):**

```java
/* packages/services/Car/service/src/com/android/car/CarService.java */
/* Started from SystemServer via:
   context.startService(new Intent(context, CarService.class)) */

public void onCreate() {
    mCarServiceProxy = new CarServiceProxy(...);
    
    /* Init CarAudioService, CarPowerManagementService, etc. */
    ICarImpl carImpl = new ICarImpl(this, vehicleHal, ...);
    carImpl.init();
    
    /* Connect to Vehicle HAL (AIDL: IVehicle) */
    VehicleHal vehicleHal = new VehicleHal(context);
    vehicleHal.init();
}
```

**logcat Output (System Boot Completion):**

```
01-01 00:00:04.100   1  1 I Zygote  : ...preloaded 6512 classes in 1547ms.
01-01 00:00:04.200 471 471 I SystemServer: Entered the Android system server!
01-01 00:00:04.300 471 471 I ActivityManager: AMS starting...
01-01 00:00:04.800 471 471 I PackageManager: /data/app scanned in 512ms
01-01 00:00:05.100 471 471 I AudioService: Initializing AudioService
01-01 00:00:05.200 471 471 I CarAudioService: Setting up audio dynamic routing
01-01 00:00:05.300 471 471 I CarAudioService: ro.android.car.audio.enableaudiopatch=true
01-01 00:00:05.400 471 471 I CarAudioService: Calling setPortGain() for zone 0
01-01 00:00:06.000 471 471 I ActivityManager: Boot is finished after 5123ms!
01-01 00:00:06.100 471 471 I Launcher: HomeActivity started
```

---

## 2. Chi Điểm Source Code & Log Minh Họa

### Summary Table: Source → Function → Log

| Giai Đoạn | Source File | Hàm Cốt Lõi | Log Identifier |
|---|---|---|---|
| BootROM | (mask ROM, no source) | (internal) | `hab_status` in U-Boot |
| SPL | `u-boot/board/freescale/imx95_evk/spl.c` | `spl_dram_init()` | `DDRINFO: 1D training done` |
| DDR PHY | `u-boot/drivers/ddr/imx/lpddr5/ddr_phy_train.c` | `ddr_phy_train()` | `DDRINFO: init_freq set to` |
| ATF BL31 | `trusted-firmware-a/plat/imx/imx95/imx95_bl31_setup.c` | `bl31_platform_setup()` | `NOTICE: BL31: v2.10.0` |
| ELE FW | (NXP blob) | (ELE internal) | `ELE FW Version: 0x07 0x02` |
| M33 SM | `imx-sm/` (MCUXpresso) | `SM_Init()` | `imx-scmi-misc: NXP SM Platform` |
| U-Boot | `u-boot/common/board_f.c` | `initcall_run_list()` | `CPU: NXP i.MX95 rev1.1` |
| Kernel Entry | `arch/arm64/kernel/head.S` | `primary_entry` | `Booting Linux on physical CPU` |
| start_kernel | `init/main.c` | `start_kernel()` | `Linux version 6.6.23` |
| init Phase 1 | `system/core/init/first_stage_init.cpp` | `FirstStageMain()` | `init first stage started!` |
| SELinux | `system/core/init/selinux.cpp` | `SetupSelinux()` | `Loading SELinux policy` |
| init Phase 2 | `system/core/init/init.cpp` | `SecondStageMain()` | `init second stage started!` |
| Zygote | `frameworks/base/core/jni/AndroidRuntime.cpp` | `AndroidRuntime::start()` | `preloaded 6512 classes` |
| SystemServer | `frameworks/base/services/java/com/android/server/SystemServer.java` | `run()` | `Entered the Android system server!` |
| CarService | `packages/services/Car/service/src/com/android/car/CarService.java` | `onCreate()` | `CarAudioService: Setting up audio` |
| Launcher | (SystemUI/Launcher3) | `HomeActivity.onCreate()` | `Launcher: HomeActivity started` |

### Additional Debugging Reference

**Boot Time Profiling:**

```bash
# Xem boot timing breakdown qua bootstat
adb shell adb shell bootstat -p

# Kernel initcall timing
adb shell dmesg | grep "initcall"
# Output: [    0.720000] initcall fsl_sai_driver_init+0x0/0x40 returned 0 after 1250 usecs

# Systemd-style boot chart (Android)
adb shell atrace --async_start -t 30 -b 16384 am wm view hal
adb shell atrace --async_stop
adb pull /data/local/tmp/atrace_data.ctrace
```

**Tracing Kernel → HAL → CarAudioService path:**

```bash
# 1. Check AudioFlinger mixer thread
adb shell dumpsys media.audio_flinger | grep -A10 "Output thread"

# 2. Check AudioPolicyService routing
adb shell dumpsys media.audio_policy | grep -E "output|route|port"

# 3. Check CarAudioService zone gain
adb shell dumpsys car_service | grep -i "gain\|port\|patch"

# 4. ALSA level: verify port gain applied
adb shell tinymix | grep -i "volume\|gain"
```

---

## 3. Quá Trình Biên Dịch Phân Mảnh (Modular Build)

### 3.1 Thiết Lập Môi Trường Build

```bash
# AOSP Build Setup
source build/envsetup.sh
lunch evk_95-ap3a-userdebug    # Target: evk_95, Android 14/15
# hoặc
lunch evk_imx95-userdebug      # NXP BSP variant

# Verify environment
echo $ANDROID_BUILD_TOP        # /path/to/aosp
echo $TARGET_PRODUCT           # evk_95
echo $TARGET_BUILD_VARIANT     # userdebug
echo $OUT                      # $ANDROID_BUILD_TOP/out/target/product/evk_95
```

### 3.2 Build Toàn Bộ (Full Build)

```bash
# Full AOSP build (first time: 3-6 hours on 64-core machine)
m -j$(nproc)

# Equivalent to:
make -j$(nproc) SOONG_ALLOW_MISSING_DEPENDENCIES=true

# With verbose output:
m -j$(nproc) showcommands 2>&1 | tee build.log
```

### 3.3 Build Riêng Bootloader (U-Boot + ATF + flash.bin)

```bash
# ---- Cách 1: AOSP build target ----
# Build chỉ bootloader artifacts
m bootloader
# Output: $OUT/bootloader.img  (= flash.bin được wrap)

# ---- Cách 2: Manual build (ngoài AOSP) ----

# Biến môi trường cho cross-compile:
export ARCH=arm64
export CROSS_COMPILE=aarch64-linux-android-  # hoặc aarch64-linux-gnu-
export CROSS_COMPILE_M33=arm-none-eabi-       # cho M33 firmware

# --- Build ATF (ARM Trusted Firmware) ---
cd trusted-firmware-a/
make PLAT=imx95 \
     BL33=/path/to/u-boot/u-boot-nodtb.bin \
     NEED_BL2=yes \
     IMX_BOOT_UART_BASE=0x44380000 \
     SPD=none \
     bl31
# Output: build/imx95/release/bl31.bin

# --- Build U-Boot ---
cd u-boot/
make imx95_evk_defconfig   # Load board defconfig
# hoặc
make imx95_evk_android_defconfig  # Android-specific (fastboot, AVB, etc.)

# Nếu cần chỉnh sửa config:
make menuconfig

# Build SPL + U-Boot proper
make -j$(nproc)
# Outputs:
#   spl/u-boot-spl.bin     (SPL binary)
#   u-boot-nodtb.bin       (U-Boot without DTB)
#   u-boot.dtb             (U-Boot device tree)
#   u-boot.bin             (= u-boot-nodtb.bin + u-boot.dtb)

# --- Assemble flash.bin với imx-mkimage ---
cd imx-mkimage/
cp /path/to/u-boot/spl/u-boot-spl.bin iMX95/
cp /path/to/u-boot/u-boot-nodtb.bin   iMX95/
cp /path/to/u-boot/u-boot.dtb         iMX95/
cp /path/to/atf/build/imx95/release/bl31.bin  iMX95/
cp /path/to/ele-firmware/mx95a0-ahab-container.img  iMX95/  # NXP ELE blob
cp /path/to/imx-sm/build/mx95/m33_image.bin  IsMX95/         # M33 SM firmware

make SOC=iMX95 \
     dtbs=evk-imx95.dtb \
     flash_singleboot_m33

# Output: iMX95/flash.bin
```

**imx-mkimage `Makefile` logic - hiểu cấu trúc flash.bin:**

```makefile
# imx-mkimage/iMX95/soc.mak (tham khảo)
flash_singleboot_m33: $(MKIMG) $(AHAB_IMG) m33_image.bin u-boot-spl.bin \
                      u-boot-nodtb.bin u-boot.dtb bl31.bin
    ./mkimage_imx8 \
        -soc IMX9 \
        -c \
        -ap u-boot-spl.bin a55 0x2049_0000 \
        -ap bl31.bin       a55 0x204E_0000 \
        -ap u-boot-nodtb.bin a55 0x8020_0000 \
        -p u-boot.dtb \
        -m33 m33_image.bin 0x201E_0000 \
        -ahab $(AHAB_IMG) \
        -out flash.bin
# mkimage_imx8 tạo Boot Container với:
#   - Container Header (tag 0x87)
#   - Image Entries: ELE FW, M33 FW, SPL, BL31, U-Boot
#   - SRK Table (nếu HAB enabled)
#   - ECDSA/RSA Signature (nếu signed build)
```

### 3.4 Build Riêng Linux Kernel

```bash
# ---- Cách 1: Android kernel build (GKI-based) ----
# NXP i.MX95 dùng GKI (Generic Kernel Image) trên Android 14/15

cd kernel/
# Vendor kernel tree: kernel/nxp/ hoặc từ Android Common Kernel
# https://android.googlesource.com/kernel/common (android14-6.1 / android15-6.6)

# Setup build environment
BUILD_CONFIG=kernel/nxp/build.config.imx95 build/build.sh

# Hoặc với Bazel (Android 13+):
tools/bazel build //common:imx95_dist

# ---- Cách 2: Manual kernel build ----
export ARCH=arm64
export CROSS_COMPILE=aarch64-linux-android-

make imx95_defconfig         # hoặc android_imx95_defconfig

# Tùy chỉnh config
make menuconfig              # TUI config editor
# hoặc
scripts/config --enable CONFIG_SND_SOC_FSL_SAI   # Enable SAI audio driver
scripts/config --enable CONFIG_SND_SOC_TAS5828M  # Enable TAS5828 codec

# Build kernel Image + modules
make -j$(nproc) Image modules

# Build Device Tree Blobs
make -j$(nproc) dtbs
# Output:
#   arch/arm64/boot/Image             (compressed kernel)
#   arch/arm64/boot/dts/freescale/imx95-19x19-evk.dtb
#   arch/arm64/boot/dts/freescale/imx95-19x19-evk-*.dtbo

# Install modules
make INSTALL_MOD_PATH=$OUT/vendor modules_install
# Output: $OUT/vendor/lib/modules/6.6.23/*.ko

# ---- Cách 3: Rebuild chỉ một kernel module ----
# Ví dụ: rebuild fsl_sai (SAI audio driver)
make -C /path/to/kernel M=/path/to/kernel/sound/soc/fsl modules
# Nhanh hơn nhiều so với full kernel build
```

**Build chỉ Device Tree (không rebuild kernel):**

```bash
# Cực kỳ hữu ích khi chỉ chỉnh sửa DTS/DTSI
make -j$(nproc) dtbs

# Verify DTS compilation
dtc -I dtb -O dts -o /tmp/check.dts \
    arch/arm64/boot/dts/freescale/imx95-19x19-evk.dtb

# Build riêng 1 DTB
make arch/arm64/boot/dts/freescale/imx95-19x19-evk.dtb

# Build DTBO (Device Tree Blob Overlay)
make arch/arm64/boot/dts/freescale/imx95-19x19-evk-audio.dtbo
```

### 3.5 Build Riêng Audio HAL

```bash
# Build chỉ Audio HAL (AIDL implementation)
cd $ANDROID_BUILD_TOP

# Build vendor HAL module
m android.hardware.audio.service.nxp
# Sources: vendor/nxp/nxp_android_multimedia/audio/

# Build AudioFlinger (framework)
m libaudioflinger

# Build AudioPolicyService
m libaudiopolicyservice

# Build CarAudioService (JAR)
m car-frameworks-service-module

# Build AudioControl HAL (Automotive)
m android.hardware.automotive.audiocontrol-service.nxp

# Push trực tiếp lên device (không cần full flash)
adb root
adb disable-verity
adb reboot
adb root

# Push HAL binary
adb push $OUT/vendor/bin/hw/android.hardware.audio.service.nxp \
         /vendor/bin/hw/

# Push HAL library
adb push $OUT/vendor/lib64/android.hardware.audio.core-V2-ndk.so \
         /vendor/lib64/

# Restart AudioServer
adb shell stop audioserver
adb shell start audioserver
```

### 3.6 Build Riêng Android Partition Images

```bash
# Build chỉ system image (sau khi thay đổi framework)
m systemimage
# Output: $OUT/system.img

# Build chỉ vendor image (sau khi thay đổi HAL/driver)
m vendorimage
# Output: $OUT/vendor.img

# Build boot image (kernel + ramdisk)
m bootimage
# Output: $OUT/boot.img

# Build vendor_boot image (vendor ramdisk + DTBs)
m vendorbootimage
# Output: $OUT/vendor_boot.img

# Build super image (chứa system + vendor + product + system_ext)
m superimage
# Output: $OUT/super.img

# Build tất cả images (không compile Java/C++)
m dist
```

### 3.7 Build trong Yocto (NXP BSP Yocto / meta-imx)

```bash
# Setup Yocto (kirkstone / scarthgap)
. setup-environment build-imx95

# Build full image
bitbake fsl-image-automotive

# Build riêng U-Boot
bitbake u-boot-imx

# Build riêng kernel
bitbake linux-imx

# Build riêng kernel module
bitbake linux-imx -c compile_kernelmodules -f

# Build với devshell (interactive)
bitbake u-boot-imx -c devshell
# → Opens shell in U-Boot source with build env set up

# Force rebuild artifact
bitbake linux-imx -c cleansstate && bitbake linux-imx

# Deploy artifacts
bitbake fsl-image-automotive -c image
# Output in: tmp/deploy/images/imx95-19x19-evk/
```

---

## 4. Output Artifacts & Cơ Chế Flashing

### 4.1 Tổng Quan Phân Vùng eMMC

```
eMMC Physical Layout (NXP i.MX95 EVK, 64GB UFS/eMMC):

┌──────────────────────────────────────────────────────────────────────┐
│  eMMC Boot Partition 0 (boot0, 4MB)                                  │
│  Offset 0x000: Boot Container (flash.bin)                            │
│    → Container Header                                                 │
│    → ELE Firmware (mx95a0-ahab-container.img)                       │
│    → M33 SM Firmware (m33_image.bin)                                 │
│    → SPL (u-boot-spl.bin, loads to OCRAM 0x2049_0000)              │
│    → ATF BL31 (bl31.bin, loads to OCRAM 0x204E_0000)               │
│    → U-Boot Proper (u-boot-nodtb.bin, loads to DDR 0x8020_0000)    │
│    → U-Boot DTB (u-boot.dtb)                                         │
├──────────────────────────────────────────────────────────────────────┤
│  eMMC Boot Partition 1 (boot1, 4MB) - redundant boot copy           │
├──────────────────────────────────────────────────────────────────────┤
│  eMMC User Area (main partition):                                    │
│                                                                      │
│  Offset 0x000000: GPT Header                                         │
│  Offset 0x004000: GPT Partition Table                                │
│                                                                      │
│  Part#  Name          Size     Type                                  │
│  ──────────────────────────────────────                              │
│  1      bootenv       512KB    FAT16 (U-Boot env)                   │
│  2      boot_a        64MB     Android Boot Image                    │
│  3      boot_b        64MB     Android Boot Image (B-slot)          │
│  4      vendor_boot_a 64MB     Vendor Boot Image                    │
│  5      vendor_boot_b 64MB     Vendor Boot Image (B-slot)          │
│  6      dtbo_a        8MB      DTBO image                            │
│  7      dtbo_b        8MB      DTBO image (B-slot)                  │
│  8      vbmeta_a      64KB     Verified Boot Metadata               │
│  9      vbmeta_b      64KB     Verified Boot Metadata (B-slot)      │
│  10     vbmeta_system_a 64KB  System vbmeta                         │
│  11     vbmeta_vendor_a 64KB  Vendor vbmeta                         │
│  12     super         4GB     Dynamic Partitions Super Image        │
│  13     metadata      16MB    Dynamic Partitions Metadata           │
│  14     userdata      ~50GB   Android /data                         │
│  15     cache         256MB   OTA cache                             │
└──────────────────────────────────────────────────────────────────────┘
```

### 4.2 `flash.bin` / `bootloader.img` và imx-mkimage

**Cấu trúc Binary flash.bin:**

```
flash.bin (ví dụ: ~2MB):

Byte 0x000  ┌─────────────────────────────────────┐
            │  Padding (0x400 bytes = 1KB)         │
            │  (eMMC CMD0 timing requirement)       │
Byte 0x400  ├─────────────────────────────────────┤
            │  Boot Container 1 (Primary)          │
            │  ┌──────────────────────────────┐    │
            │  │ Container Header (tag=0x87)  │    │
            │  │ version, flags, num_images   │    │
            │  │ sig_blk_offset               │    │
            │  └──────────────────────────────┘    │
            │  ┌──────────────────────────────┐    │
            │  │ Image Entry[0]: ELE FW       │    │
            │  │ offset, size, load_addr=0    │    │
            │  │ entry_point, hab_flags       │    │
            │  │ sha384_hash[48]              │    │
            │  └──────────────────────────────┘    │
            │  ┌──────────────────────────────┐    │
            │  │ Image Entry[1]: M33 SM FW    │    │
            │  │ load_addr: 0x201E_0000       │    │
            │  └──────────────────────────────┘    │
            │  ┌──────────────────────────────┐    │
            │  │ Image Entry[2]: SPL          │    │
            │  │ load_addr: 0x2049_0000       │    │
            │  └──────────────────────────────┘    │
            │  ┌──────────────────────────────┐    │
            │  │ Image Entry[3]: ATF BL31     │    │
            │  │ load_addr: 0x204E_0000       │    │
            │  └──────────────────────────────┘    │
            │  ┌──────────────────────────────┐    │
            │  │ Signature Block              │    │
            │  │ SRK Table (4x SHA384)        │    │
            │  │ ECDSA P-384 Signature        │    │
            │  └──────────────────────────────┘    │
Byte N      ├─────────────────────────────────────┤
            │  Boot Container 2 (U-Boot)           │
            │  Image Entry[0]: U-Boot Proper       │
            │  load_addr: 0x8020_0000              │
            │  Image Entry[1]: U-Boot DTB          │
            └─────────────────────────────────────┘
```

**Flash flash.bin vào eMMC boot0:**

```bash
# Từ U-Boot (qua SD card hoặc TFTP):
# Ghi flash.bin vào eMMC boot0 partition
mmc dev 1 1                              # Chọn eMMC (dev 1), boot partition 1 (boot0)
fatload mmc 0 ${loadaddr} flash.bin     # Load từ SD card
mmc write ${loadaddr} 0x0 0x800         # Ghi 0x800 blocks (= 256KB*4 = 1MB)
# Hoặc ghi toàn bộ:
setexpr blkcount ${filesize} + 0x1ff
setexpr blkcount ${blkcount} / 0x200
mmc write ${loadaddr} 0 ${blkcount}

# Từ Linux host (qua USB OTG / uuu):
uuu flash.bin    # NXP Universal Update Utility
# uuu tự nhận biết imx9 boot container format

# Thủ công qua dd:
# (Sau khi board boot vào Android)
adb push flash.bin /sdcard/flash.bin
adb shell su -c "dd if=/sdcard/flash.bin of=/dev/block/mmcblk0boot0 bs=1024 seek=1"
# seek=1 → skip first 1KB (optional padding tùy board)

# Hoặc dùng fastboot (nếu U-Boot bootloader unlocked):
fastboot flash bootloader flash.bin
```

### 4.3 `boot.img` (GKI Architecture)

Từ Android 13 trở đi, i.MX95 sử dụng **GKI (Generic Kernel Image)** boot.img v3:

```
boot.img v3 structure:
┌──────────────────────────────────────────────────────────┐
│  Boot Header v3 (4096 bytes)                             │
│  magic: "ANDROID!"                                        │
│  kernel_size: size of GKI kernel Image                   │
│  ramdisk_size: size of generic ramdisk (no vendor stuff) │
│  os_version: 0x0F000000 (Android 15)                    │
│  header_size: 4096                                        │
│  header_version: 3                                        │
│  cmdline: "androidboot.hardware=nxp_evk95 ..."           │
├──────────────────────────────────────────────────────────┤
│  Kernel (GKI kernel Image, page-aligned)                 │
│  → Generic kernel from Android Common Kernel            │
│  → NO vendor-specific drivers (all in modules/vendor_dlkm) │
├──────────────────────────────────────────────────────────┤
│  Generic Ramdisk (page-aligned)                          │
│  → Contains: /init (first_stage), /system/bin/...       │
│  → NO vendor modules here (moved to vendor_boot)        │
└──────────────────────────────────────────────────────────┘

# Build:
m bootimage
# Output: $OUT/boot.img

# Flash:
fastboot flash boot boot.img
```

### 4.4 `vendor_boot.img` và `dtbo.img`

**vendor_boot.img** (chứa vendor-specific ramdisk + DTBs):

```
vendor_boot.img v4 structure:
┌──────────────────────────────────────────────────────────┐
│  Vendor Boot Header v4 (4096 bytes)                      │
│  magic: "VNDRBOOT"                                        │
│  vendor_ramdisk_size: total size of vendor ramdisks     │
│  dtb_size: DTB blob size                                 │
│  dtb_offset: load address for DTB                       │
│  header_version: 4                                        │
│  vendor_cmdline: "earlycon=... console=ttymxc1..."       │
│  vendor_ramdisk_table_size: N vendor ramdisk fragments   │
├──────────────────────────────────────────────────────────┤
│  Vendor Ramdisk Table (multiple ramdisk fragments):      │
│  ├── Fragment 0: "default" ramdisk                       │
│  │   → /lib/modules/6.6.23/*.ko  (vendor kernel modules)│
│  │   → /lib/modules/: fsl_sai.ko, snd-soc-tas5828m.ko  │
│  │   → /first_stage_ramdisk/     (first stage init)     │
│  ├── Fragment 1: "dlkm" ramdisk (vendor_dlkm modules)   │
│  │   → /lib/modules/: vendor-specific .ko files         │
│  └── Fragment 2: platform-specific overlay               │
├──────────────────────────────────────────────────────────┤
│  DTB (Flattened Device Tree Binary)                      │
│  → arch/arm64/boot/dts/freescale/imx95-19x19-evk.dtb   │
└──────────────────────────────────────────────────────────┘

# Build:
m vendorbootimage
# Output: $OUT/vendor_boot.img

# Flash:
fastboot flash vendor_boot vendor_boot.img
```

**dtbo.img** (Device Tree Blob Overlay):

```
dtbo.img structure (AOSP dtbo table format):
┌──────────────────────────────────────────────────────────┐
│  DTBO Table Header                                        │
│  magic: 0xD7B7AB1E                                        │
│  total_size: image total size                             │
│  header_size: 32 bytes                                    │
│  dt_entry_size: 32 bytes per entry                       │
│  dt_entry_count: number of DTBOs                         │
│  dt_entries_offset: 32 (right after header)              │
├──────────────────────────────────────────────────────────┤
│  DTBO Entry[0]: base EVK overlay                         │
│  → imx95-19x19-evk-overlay.dtbo                          │
├──────────────────────────────────────────────────────────┤
│  DTBO Entry[1]: Audio overlay (TAS5828)                  │
│  → imx95-19x19-evk-audio-tas5828.dtbo                   │
├──────────────────────────────────────────────────────────┤
│  DTBO Entry[2]: DSI display overlay                      │
│  → imx95-19x19-evk-mipi-panel.dtbo                      │
│  ...                                                      │
└──────────────────────────────────────────────────────────┘

# Build DTBOs:
make -C kernel imx95-19x19-evk-audio.dtbo
# hoặc trong AOSP:
m dtboimage
# Output: $OUT/dtbo.img

# U-Boot selects DTBO via androidboot.dtbo_idx=0,1 in bootargs
# DTBOs merged với base DTB tại kernel boot time bởi libfdt

# Flash:
fastboot flash dtbo dtbo.img
```

### 4.5 `super.img` (Dynamic Partitions)

**Dynamic Partitions Architecture:**

```
super.img (4GB):
┌──────────────────────────────────────────────────────────┐
│  LP Metadata Header (Logical Partition metadata)         │
│  magic: 0x544C4D21 ("TLM!")                              │
│  metadata_version: 10.0                                  │
│  partition_entry_size: 128                               │
│  metadata_slot_count: 2 (A/B)                           │
├──────────────────────────────────────────────────────────┤
│  Partition Table:                                         │
│  ├── system_a: size=1.5GB, type=ext4/erofs, readonly    │
│  ├── system_b: size=1.5GB (empty in B slot)             │
│  ├── vendor_a: size=512MB, type=ext4/erofs              │
│  ├── vendor_b: size=512MB                               │
│  ├── product_a: size=256MB                              │
│  ├── product_b: size=256MB                              │
│  ├── system_ext_a: size=256MB                           │
│  └── system_ext_b: size=256MB                           │
├──────────────────────────────────────────────────────────┤
│  system_a partition (ext4 or erofs):                     │
│  → /system/framework/  (jar files, DEX)                 │
│  → /system/lib64/      (system libraries)               │
│  → /system/bin/        (system binaries, init)          │
├──────────────────────────────────────────────────────────┤
│  vendor_a partition (ext4):                              │
│  → /vendor/lib/hw/     (HAL implementations)            │
│  → /vendor/lib/modules/ (kernel modules)                │
│  → /vendor/etc/        (config files, audio policy XML) │
│  → /vendor/bin/hw/     (HAL service binaries)           │
│  → /vendor/firmware/   (firmware blobs, M7/M33 .elf)   │
├──────────────────────────────────────────────────────────┤
│  product_a, system_ext_a partitions...                   │
└──────────────────────────────────────────────────────────┘

# Build:
m superimage
# Output: $OUT/super.img

# Build thành từng file nhỏ (sparse format, nhanh hơn):
m systemimage vendorimage productimage

# Flash super.img:
fastboot wipe-super super_empty.img  # Wipe metadata
fastboot flash system system.img     # Flash từng logical partition
fastboot flash vendor vendor.img

# Hoặc flash toàn bộ super:
fastboot flash super super.img

# Công cụ check nội dung super.img:
lpunpack super.img /tmp/super_unpacked/
lpdump super.img  # In metadata
```

**Dynamic Partitions tools:**

```bash
# lpdump - đọc LP metadata từ device:
adb shell lpdump /dev/block/by-name/super

# Output:
# Metadata version: 10.0
# Metadata size: 65536 bytes
# Partitions:
#   Name: system_a   Group: default   Flags: READONLY
#     Extents:   0 .. 3145727 linear super (offset 2048)
#   Name: vendor_a   Group: default   Flags: READONLY
#     Extents: 3145728 .. 4194303 linear super (offset 3147776)

# dmctl - manage device mapper:
adb shell dmctl table system_a
# Output:
# 0-3145728: linear 253:8 2048
```

### 4.6 RTOS Artifacts: Cortex-M33 & Cortex-M7 ELF Files

#### M33 System Manager Firmware

```
Build output:
  imx-sm/build/mx95/m33_image.bin     ← Raw binary (stripped ELF)
  imx-sm/build/mx95/m33_image.elf     ← ELF với debug symbols

ELF sections:
  .text   : 0x0FFE_0000  (ITCM - Instruction TCM)
  .rodata : 0x0FFE_8000
  .data   : 0x2000_0000  (DTCM - Data TCM)
  .bss    : 0x2000_4000
  .heap   : 0x2000_8000

Memory footprint (typical):
  .text:   ~40KB
  .data:   ~8KB
  Total:   ~48KB (fits in 64KB ITCM)
```

**Vị trí đặt M33 firmware để system tự load:**

```
Trong flash.bin (Boot Container):
→ imx-mkimage đặt m33_image.bin như một Image Entry trong Boot Container
→ BootROM nạp M33 firmware vào ITCM TRƯỚC khi chạy SPL
→ M33 bắt đầu chạy song song với A55 cluster ngay từ đầu

Backup copy trong filesystem:
/vendor/firmware/imx/sm/mx95_sm.bin   ← copy lưu trên vendor partition
→ Dùng cho recovery hoặc runtime update (không phổ biến)
```

#### M7 FreeRTOS / Zephyr Firmware

```
Build output (MCUXpresso SDK / Zephyr):
  release/m7_rtos_app.elf     ← ELF với full symbols
  release/m7_rtos_app.bin     ← Raw binary

ELF memory layout (FreeRTOS on M7):
  .text   : 0x0000_0000  (M7 ITCM, 64KB) hoặc
            0x2000_0000  (SRAM_A, 256KB shared)
  .data   : 0x2002_0000  (SRAM_B, 256KB)
  .bss    : 0x2004_0000
  FreeRTOS heap: 0x2006_0000  (configTOTAL_HEAP_SIZE = 32768)

Cortex-M7 IPC với A55:
  Protocol: RPMsg-lite (over shared SRAM)
  Shared SRAM: 0x2010_0000 - 0x2018_0000 (OCRAM, 512KB)
  RPMsg ring buffer: 0x2010_0000 (vring0=TX, vring1=RX)
```

**Vị trí và cơ chế load M7 firmware từ Android:**

```bash
# 1. Đặt firmware vào vendor partition:
# vendor/etc/firmware/imx/m7/m7_rtos_app.bin
# (AOSP: device/nxp/imx95_evk/firmware/m7_rtos_app.bin)

# 2. Device Tree khai báo remoteproc:
# arch/arm64/boot/dts/freescale/imx95-19x19-evk.dts:
# imx95_cm7: remoteproc@0 {
#     compatible = "fsl,imx95-cm7";
#     fsl,startup-delay-ms = <500>;
#     clocks = <&clk IMX95_CLK_M7>;
#     firmware-name = "imx/m7/m7_rtos_app.bin";  ← relative to /lib/firmware
# };

# 3. Linux remoteproc driver tự load firmware:
# drivers/remoteproc/imx_rproc.c
# → devm_request_firmware("imx/m7/m7_rtos_app.bin")
# → firmware nằm ở /vendor/lib/firmware/imx/m7/m7_rtos_app.bin
# (symlink: /lib/firmware → /vendor/lib/firmware khi Android mount vendor)

# 4. Android service trigger (optional):
# /vendor/etc/init/rpmsg-m7.rc:
# service load-m7-firmware /vendor/bin/load_m7_fw
#     class hal
#     oneshot
#     user root

# 5. Control qua sysfs:
adb shell cat /sys/class/remoteproc/remoteproc0/state
# Output: "offline" / "running"

adb shell echo start > /sys/class/remoteproc/remoteproc0/state
# → triggers imx_rproc_start() → loads ELF segments into M7 TCM
# → de-asserts M7 core reset

adb shell dmesg | grep remoteproc
# [    3.421000] remoteproc remoteproc0: imx-rproc is available
# [    3.422000] remoteproc remoteproc0: powering up imx-m7
# [    3.425000] remoteproc remoteproc0: Booting fw image imx/m7/m7_rtos_app.bin
# [    3.430000] remoteproc remoteproc0: remote processor imx-m7 is now up
# [    3.431000] virtio_rpmsg_bus virtio0: rpmsg host is online
# [    3.432000] virtio_rpmsg_bus virtio0: creating channel rpmsg-audio addr 0x400
```

### 4.7 Flash Toàn Bộ (Full Flash Procedure)

**Sử dụng NXP `uuu` (Universal Update Utility):**

```bash
# Download uuu: https://github.com/nxp-imx/mfgtools

# Chuẩn bị script uuu (uuu_imx95_evk.lst):
# uuu_version 1.4.182
# SDP: boot -f flash.bin
# SDPV: delay 1000
# FB: ucmd setenv fastboot_dev mmc
# FB: ucmd setenv mmcdev 1
# FB: ucmd mmc dev ${mmcdev} 1
# FB: flash bootloader flash.bin
# FB: flash dtbo dtbo.img
# FB: flash vbmeta vbmeta.img
# FB: flash boot boot.img
# FB: flash vendor_boot vendor_boot.img
# FB: wipe-super super_empty.img
# FB: flash system system.img
# FB: flash vendor vendor.img
# FB: flash product product.img
# FB: flash userdata userdata.img
# FB: reset

# Execute:
sudo uuu uuu_imx95_evk.lst

# Interactive fastboot:
fastboot devices
fastboot getvar all

# One-liner full flash:
fastboot flashall -w   # Flash all partitions + wipe userdata
```

**Verify Flash thành công:**

```bash
# Sau khi boot:
adb shell getprop ro.build.fingerprint
# nxp/evk_95/evk_95:15/AP3A.240905.015/2025031500:userdebug/test-keys

adb shell cat /proc/device-tree/model
# NXP i.MX95 19x19 EVK board

# Verify boot partition đang active:
adb shell bootctl get-current-slot
# 0  (slot A active)

adb shell bootctl get-suffix 0
# _a

# Verify dynamic partitions:
adb shell dmctl table system_a | head -3

# Verify kernel version:
adb shell uname -r
# 6.6.23-android15-8-00001-g3abc1234

# Verify M7 remoteproc:
adb shell cat /sys/class/remoteproc/remoteproc0/state
# running
```

---

*Document Version: 1.0 | Platform: NXP i.MX95 EVK | Android 15 (AOSP) | Kernel 6.6.x (GKI)*  
*Author: Senior Embedded Linux / AAOS Engineer | Classification: Internal Technical Reference*
