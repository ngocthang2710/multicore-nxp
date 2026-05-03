# NXP i.MX95 EVK — Deep Dive Chuyên Sâu: Boot, Inter-Core & STR
## Có Source Code Thực Tế, Log Annotated, Register Level

> **Version 2.0** — Mở rộng toàn diện với source code từ patchwork.kernel.org, lore.kernel.org, và các upstream patch thực tế của NXP engineers.

---

## Mục Lục

1. [Kiến Trúc Đa Core — Register & Address Detail](#1-kiến-trúc-đa-core--register--address-detail)
2. [Cấu Trúc flash.bin — Byte-level Analysis](#2-cấu-trúc-flashbin--byte-level-analysis)
3. [Boot ROM (M33) — Từng Bước với Register](#3-boot-rom-m33--từng-bước-với-register)
4. [OEI — DDR Training Source Code](#4-oei--ddr-training-source-code)
5. [System Manager — Source Code Thực Tế](#5-system-manager--source-code-thực-tế)
6. [SCMI LMM Protocol — Source Code Thực Tế (từ NXP patch)](#6-scmi-lmm-protocol--source-code-thực-tế)
7. [U-Boot SPL — Source Code Thực Tế imx95_evk/spl.c](#7-u-boot-spl--source-code-thực-tế)
8. [ATF BL31 — PSCI & STR Source Code Thực Tế](#8-atf-bl31--psci--str-source-code-thực-tế)
9. [Linux Kernel Boot — Driver Init với Source Code](#9-linux-kernel-boot--driver-init-với-source-code)
10. [Inter-Core: SCMI Transport — Packet Level](#10-inter-core-scmi-transport--packet-level)
11. [Inter-Core: RPMsg/MU — Source Code & Virtio Detail](#11-inter-core-rpmsgmu--source-code--virtio-detail)
12. [STR — Suspend to RAM: Source Code End-to-End](#12-str--suspend-to-ram-source-code-end-to-end)
13. [Android Audio Stack — Từ App Xuống SAI Register](#13-android-audio-stack--từ-app-xuống-sai-register)
14. [Debug Cookbook — Lệnh Thực Tế](#14-debug-cookbook--lệnh-thực-tế)
15. [Bảng Tra Cứu: Log → Root Cause](#15-bảng-tra-cứu-log--root-cause)

---

## 1. Kiến Trúc Đa Core — Register & Address Detail

### 1.1 SoC Block Diagram với Addresses

```
┌──────────────────────────────────────────────────────────────────────────┐
│                         NXP i.MX95 (MIMX9596)                            │
│                                                                           │
│ ┌──────────────────────── APD (Application Domain) ─────────────────┐   │
│ │  Cortex-A55 × 6  (AArch64, OPP: 1800 MHz max)                     │   │
│ │  GIC-700 base: 0x48000000                                          │   │
│ │  L3 cache: 1MB unified                                             │   │
│ │  UART1 (A55): 0x44380000  ← COM3 trên EVK                         │   │
│ └────────────────────────────────────────────────────────────────────┘   │
│                                                                           │
│ ┌───────────── RTD (Real-Time Domain) ─────────────────────────────┐     │
│ │  ┌───────────────────────┐    ┌──────────────────────────────┐   │     │
│ │  │  Cortex-M33 @ 333 MHz │    │  Cortex-M7  @ 800 MHz        │   │     │
│ │  │  [Boot Core + SM]     │    │  [Real-time co-proc]         │   │     │
│ │  │  ITCM: 0x1FFE0000     │    │  ITCM: 0x0000_0000 (M7 view) │   │     │
│ │  │         256KB         │    │       = 0x2048_0000 (global) │   │     │
│ │  │  DTCM: 0x2000_0000    │    │  DTCM: 0x2000_0000 (M7 view) │   │     │
│ │  │         256KB         │    │       = 0x2050_0000 (global) │   │     │
│ │  │  UART3 (M33): 0x44570000   │  UART1 (M7): 0x44380000      │   │     │
│ │  │  → COM4 trên EVK      │    │  → COM1 trên EVK             │   │     │
│ │  └───────────────────────┘    └──────────────────────────────┘   │     │
│ └───────────────────────────────────────────────────────────────────┘     │
│                                                                           │
│ ┌──────── ELE (EdgeLock Enclave) ────────┐  ┌─── AON Domain ───────┐    │
│ │  Arm SC300 (secure, not programmable)  │  │  LPCG, RTC, BBSM     │    │
│ │  Base: 0x47520000 (ELE MU)             │  │  Always-on power     │    │
│ └────────────────────────────────────────┘  └──────────────────────┘    │
│                                                                           │
│ ┌──────────────── Interconnect / NOC + TRDC ──────────────────────────┐  │
│ │  MU0 (SM↔A55): 0x44230000    MU1 (SM↔M7): 0x44240000              │  │
│ │  MU2 (M7↔A55 RPMsg): 0x42430000                                     │  │
│ │  TRDC base: 0x44270000  (resource domain controller)               │  │
│ └──────────────────────────────────────────────────────────────────────┘  │
└──────────────────────────────────────────────────────────────────────────┘
```

### 1.2 MU (Messaging Unit) Register Map — MU0 Chi Tiết

```c
/* MU0 base: 0x44230000 — SM (M33) ↔ A55 Linux (SCMI transport) */
/* File: arch/arm64/boot/dts/freescale/imx95.dtsi */

#define MU0_BASE    0x44230000

/* Registers */
#define MU_VER      (MU0_BASE + 0x000)  /* Version: 0x02_00_00_03 */
#define MU_PAR      (MU0_BASE + 0x004)  /* Parameters: TX/RX count */
#define MU_CR       (MU0_BASE + 0x008)  /* Control */
#define MU_SR       (MU0_BASE + 0x00C)  /* Status:
                                            bit 0: TX FIFO empty
                                            bit 4: RX FIFO full (interrupt) */
#define MU_FCR      (MU0_BASE + 0x010)  /* FIFO Control */
#define MU_FSR      (MU0_BASE + 0x014)  /* FIFO Status */
#define MU_GIER     (MU0_BASE + 0x110)  /* General Interrupt Enable */
#define MU_GCR      (MU0_BASE + 0x114)  /* General Control (doorbell) */
#define MU_GSR      (MU0_BASE + 0x118)  /* General Status */
#define MU_TCR      (MU0_BASE + 0x120)  /* TX Control (enable TX interrupt) */
#define MU_TSR      (MU0_BASE + 0x124)  /* TX Status */
#define MU_RCR      (MU0_BASE + 0x128)  /* RX Control */
#define MU_RSR      (MU0_BASE + 0x12C)  /* RX Status */
#define MU_TR0      (MU0_BASE + 0x200)  /* Transmit Register 0 — 32-bit */
#define MU_TR1      (MU0_BASE + 0x204)  /* Transmit Register 1 */
#define MU_TR2      (MU0_BASE + 0x208)  /* Transmit Register 2 */
#define MU_TR3      (MU0_BASE + 0x20C)  /* Transmit Register 3 */
#define MU_RR0      (MU0_BASE + 0x280)  /* Receive Register 0 */
#define MU_RR1      (MU0_BASE + 0x284)  /* Receive Register 1 */
#define MU_RR2      (MU0_BASE + 0x288)  /* Receive Register 2 */
#define MU_RR3      (MU0_BASE + 0x28C)  /* Receive Register 3 */
```

### 1.3 SCMI Shared Memory Layout

```
/* SCMI uses shmem (shared memory) + MU doorbell */
/* shmem location defined in DTS: */
/*   scmi_shmem0: 0x44611000, size 0x80 bytes (M33/M7 side) */
/*   scmi_shmem1: 0x44631000, size 0x80 bytes (A55 side)    */

/* Structure của một SCMI message trong shared memory: */
struct scmi_shared_mem {
    uint32_t reserved;          /* 0x00: reserved */
    uint32_t channel_status;    /* 0x04: bit0=busy, bit1=error */
    uint32_t reserved1[2];      /* 0x08-0x0F */
    uint32_t flags;             /* 0x10: interrupt flags */
    uint32_t length;            /* 0x14: tổng length của header+payload */
    uint32_t msg_header;        /* 0x18:
                                    [3:0]   = token (sequence number)
                                    [7:8]   = msg_type (0=cmd, 2=notify)
                                    [15:8]  = message_id (command)
                                    [27:16] = protocol_id */
    uint8_t  msg_payload[0];    /* 0x1C: payload bắt đầu từ đây */
};

/* Ví dụ: A55 gửi SCMI_CLOCK_RATE_SET cho SAI3 clock:
   msg_header = 0x00_14_00_01
     protocol_id = 0x14 (CLOCK)
     message_id  = 0x05 (RATE_SET)
     token       = 0x01 (sequence)
   payload[0..3] = clock_id  (SAI3 = 24 theo imx95-clock.h)
   payload[4..7] = flags
   payload[8..15]= rate (48000 * 512 = 24576000 Hz)
*/
```

---

## 2. Cấu Trúc flash.bin — Byte-level Analysis

### 2.1 imximage.cfg (U-Boot build) — File Thực Tế

```ini
/* File: arch/arm/mach-imx/imx9/scmi/imximage.cfg
   Đây là config file cho mkimage tạo flash.bin */

BOOT_FROM SD
SOC_TYPE IMX9
APPEND mx95a0-ahab-container.img       /* ELE firmware container (NXP signed) */

CONTAINER
IMAGE OEI  m33-oei-ddrfw.bin  0x1ffc0000   /* OEI-DDR: M33 TCM entry */
HOLD 0x10000                               /* HOLD: delay before next image */
IMAGE OEI  oei-m33-tcm.bin   0x1ffc0000   /* OEI-TCM: M33 TCM layout init */
IMAGE EXEC m33_image.bin      0x1ffc0000   /* System Manager: permanent M33 */

CONTAINER
IMAGE A55  bl31.bin           0x8a200000   /* ATF BL31: A55 OCRAM */
IMAGE A55  u-boot.bin         0x90200000   /* U-Boot proper: DDR */
```

### 2.2 flash.bin Physical Layout trên SD Card

```
Offset (bytes)   Content
────────────────────────────────────────────────────────
0x0000           MBR / Partition table
0x8000           AHAB Primary Container (container 1)
  +0x000         Container Header (magic: 0x87, version: 2)
  +0x010         Image Array Entry 0: ELE FW
                   flags: 0x0000_0028 (type=ELE, core=ELE)
                   image_offset → ELE firmware binary
  +0x030         Image Array Entry 1: OEI-DDR
                   flags: 0x0000_0022 (type=OEI, core=M33)
                   load_addr: 0x1FFC0000
                   entry_point: 0x1FFC0001  ← thumb bit set
  +0x050         Image Array Entry 2: OEI-TCM
  +0x070         Image Array Entry 3: SM binary
                   flags: 0x0000_0002 (type=executable, core=M33)
                   load_addr: 0x1FFC0000

Secondary Container (container 2) ← loaded by SPL's spl_fit_read()
  Image Array Entry 0: ATF BL31
                   core: A55, load: 0x8A200000
  Image Array Entry 1: U-Boot proper
                   core: A55, load: 0x90200000
  Image Array Entry 2: DTB
                   load: 0x93000000

Kernel Image (Android boot.img format, ext4 partition)
```

---

## 3. Boot ROM (M33) — Từng Bước với Register

### 3.1 M33 Boot ROM Flow với Register

```c
/* Boot ROM không có source (ROM), nhưng behavior đã được documented */

/* Step 1: Đọc boot mode từ BOOTMODE register */
/* BBNSM_CTRL register: 0x44440000 + 0x14 */
/* BM_OVERRIDE_DATA field → determines boot device */

/* Step 2: Đọc boot device */
/* Ví dụ: SD card tại USDHC2 (0x42860000) */
/* Boot ROM sẽ:
   1. Enable USDHC2 clock qua CCM
   2. Init USDHC2 controller
   3. Đọc SD card tại offset 0x8000 (32KB)
   4. Parse AHAB container header
*/

/* Step 3: Verify bằng ELE */
/* ELE_AUTH_IMAGE_REQ command qua ELE MU (0x47520000) */

/* Step 4: Load ELE firmware → ELE core */
/* Step 5: Load OEI-DDR → M33 TCM 0x1FFC0000 */
/* Step 6: Execute OEI-DDR (entry: 0x1FFC0001) */
/* Step 7: OEI-DDR returns → Boot ROM continues */
/* Step 8: Load OEI-TCM → M33 TCM, execute */
/* Step 9: Load SM binary → M33 TCM 0x1FFC0000 */
/* Step 10: Branch to SM entry (0x1FFC0001) */
```

**Boot ROM Log (COM4, 115200 baud):**
```
/* Boot ROM không có console output!
   Output đầu tiên là từ System Manager */
```

**Chú ý:** Boot ROM hoàn toàn im lặng. Output đầu tiên là từ SM sau khi được load.

---

## 4. OEI — DDR Training Source Code

### 4.1 OEI-DDR Entry Point (imx-oei)

```c
/* File: imx-oei/devices/MIMX9596/startup/startup.c
   Entry point sau khi Boot ROM load OEI vào M33 TCM */

void Reset_Handler(void)
{
    /* 1. Setup stack pointer */
    __asm volatile ("ldr r0, =__StackTop \n\t"
                    "msr msp, r0        \n\t");

    /* 2. Copy .data section từ flash → RAM */
    SystemInitHook();

    /* 3. Gọi main OEI function */
    main();

    /* 4. Return to Boot ROM (trở về caller) */
    return;
}

/* File: imx-oei/boards/mx95lp5/ddr/ddr_init.c */
int main(void)
{
    int ret;

    /* Enable debug UART output */
    debug_init();
    debug_printf("\r\nDDR: LPDDR5\r\n");

    /* Step 1: Configure DDR controller registers */
    ret = ddrc_config(&ddrc_cfg);
    if (ret)
        goto err;

    debug_printf("DDR: DDRC configured\r\n");

    /* Step 2: Load DDR PHY firmware vào PHY IMEM/DMEM */
    /* PHY firmware là binary blob từ Synopsys/NXP */
    ret = ddr_phy_init(DDR_PHY_BASE, &phy_fw, &phy_cfg);
    if (ret)
        goto err;

    debug_printf("DDR: PHY firmware loaded\r\n");

    /* Step 3: Run PHY training sequence */
    /* Training bao gồm: Write Leveling, Read Gate Training,
       Read Eye Training, Write Eye Training */
    ret = ddrc_train(&train_cfg);
    if (ret)
        goto err;

    debug_printf("DDR: Training PASS\r\n");
    debug_printf("DDR: Size: %dMB\r\n", get_ddr_size_mb());

    return 0;

err:
    debug_printf("DDR: INIT FAILED ret=%d\r\n", ret);
    /* Boot ROM sẽ nhận error code và có thể fallback */
    return ret;
}
```

### 4.2 DDR PHY Init — Load Firmware vào PHY

```c
/* File: imx-oei/components/ddr/phy_train.c */

int ddr_phy_init(uintptr_t phy_base, const struct phy_fw *fw,
                 const struct phy_cfg *cfg)
{
    /* Step 1: Assert PHY reset */
    mmio_write_32(phy_base + DDRPHY_RESET, 0x1);

    /* Step 2: Load IMEM (instruction memory) */
    /* IMEM base: phy_base + 0x50000 */
    for (int i = 0; i < fw->imem_size / 4; i++) {
        mmio_write_32(phy_base + DDRPHY_IMEM_BASE + i * 4,
                     fw->imem_data[i]);
    }

    /* Step 3: Load DMEM (data memory) */
    /* DMEM base: phy_base + 0x54000 */
    for (int i = 0; i < fw->dmem_size / 4; i++) {
        mmio_write_32(phy_base + DDRPHY_DMEM_BASE + i * 4,
                     fw->dmem_data[i]);
    }

    /* Step 4: Set message block parameters (training config) */
    /* MessageBlock ở DMEM offset 0x0 */
    struct msg_block *mb = (struct msg_block *)(phy_base + DDRPHY_DMEM_BASE);
    mb->SequenceCtrl    = cfg->seq_ctrl;   /* 0x031F = all training steps */
    mb->PhyConfigOverride = 0;
    mb->HdtCtrl         = 0x5;            /* Debug level */
    mb->CsPresent        = 0xF;           /* 4 ranks (LPDDR5) */
    mb->DramFreq         = 3200;          /* 3200 MHz = 6400 MT/s */
    mb->PllBypassEn      = 0;
    mb->DfiFreqRatio     = 1;             /* 1:2 ratio */

    /* Step 5: Deassert reset, trigger training */
    mmio_write_32(phy_base + DDRPHY_RESET, 0x0);
    mmio_write_32(phy_base + DDRPHY_MICRO_CONT_MUX_SEL, 0x0);  /* release uC */

    /* Step 6: Poll for training completion */
    return wait_for_training_done(phy_base);
}

static int wait_for_training_done(uintptr_t phy_base)
{
    uint32_t msg;
    int timeout = 500000;  /* ~5s timeout */

    do {
        /* Poll UctWriteProtShadow để check message */
        msg = mmio_read_32(phy_base + DDRPHY_UCT_SHADOW_REGS);

        if (msg == 0x07) {
            /* Training complete! */
            debug_printf("DDR: PHY training done\r\n");
            return 0;
        }
        if (msg == 0xFF) {
            debug_printf("DDR: PHY training FAILED\r\n");
            return -1;
        }

        /* Clear request để PHY tiếp tục */
        mmio_write_32(phy_base + DDRPHY_UCT_WRITE_PROT, 0x0);
        udelay(1);

    } while (--timeout);

    debug_printf("DDR: PHY training TIMEOUT\r\n");
    return -ETIMEDOUT;
}
```

**OEI DDR Log (COM4):**
```
[OEI] DDR: LPDDR5
[OEI] DDR: DDRC configured
[OEI] DDR: PHY firmware loaded (imem=49152B, dmem=24576B)
[OEI] DDR: Running Write Leveling...
[OEI] DDR: Running Read Gate Training...
[OEI] DDR: Running Read Eye Training...
[OEI] DDR: Running Write Eye Training...
[OEI] DDR: Training PASS
[OEI] DDR: Size: 8192MB @ 6400 MT/s
[OEI] OEI complete, returning to Boot ROM
```

---

## 5. System Manager — Source Code Thực Tế

### 5.1 SM Main Entry (imx-sm)

```c
/* File: imx-sm/sm/src/main.c */

int main(void)
{
    int32_t status;

    /* Step 1: Platform init — hardware-specific setup */
    status = BRD_SM_Init();
    if (status != SM_ERR_SUCCESS)
        SM_Error(status);

    /* Step 2: Configure TRDC — resource domain isolation */
    /* TRDC: Trusted Resource Domain Controller */
    status = CONFIG_Load(g_trdc_config, g_trdc_config_len);
    if (status != SM_ERR_SUCCESS)
        SM_Error(status);

    SM_PRINTF("SM: TRDC configured\r\n");

    /* Step 3: Power management init */
    status = PWR_Init();
    if (status != SM_ERR_SUCCESS)
        SM_Error(status);

    SM_PRINTF("SM: Power init complete\r\n");

    /* Step 4: Clock init */
    status = CLK_Init();
    if (status != SM_ERR_SUCCESS)
        SM_Error(status);

    SM_PRINTF("SM: Clock init complete\r\n");

    /* Step 5: Create và start Logical Machines */
    status = LM_Init();      /* Init LM subsystem */
    status = LM_Boot(0U);    /* Boot SM itself (LM0) */
    status = LM_Boot(1U);    /* Boot AP domain (LM1 = A55) */
    /* LM2 (M7) chỉ boot khi có remoteproc request */

    SM_PRINTF("SM: LM1 (AP) booted, A55 released\r\n");
    SM_PRINTF("SM: Entering SCMI service mode\r\n");

    /* Step 6: Main service loop — xử lý SCMI requests mãi mãi */
    while (true) {
        /* Xử lý SCMI messages từ A55 (qua MU0) */
        RPC_SCMI_Process(0U);  /* channel 0 = A55 */

        /* Xử lý SCMI messages từ M7 (qua MU1) nếu M7 running */
        RPC_SCMI_Process(1U);  /* channel 1 = M7 */

        /* Xử lý system events (thermal, power, etc.) */
        SM_EventProcess();

        /* Enter low-power wait nếu không có gì làm */
        __WFI();
    }

    return 0;  /* Never reached */
}
```

### 5.2 LM_Boot() — Cách SM Boot A55

```c
/* File: imx-sm/sm/src/lmm/sm_lmm.c */

int32_t LM_Boot(uint32_t lmId)
{
    lm_t *lm = &g_lm[lmId];
    int32_t status = SM_ERR_SUCCESS;

    SM_PRINTF("SM: Booting LM%u (%s)\r\n", lmId, lm->name);

    /* Step 1: Power on domain cho LM này */
    status = PWR_LmPower(lmId, SM_POWER_ON);
    if (status != SM_ERR_SUCCESS)
        return status;

    /* Step 2: Với A55 (LM1): Set reset vector */
    if (lm->cpuId != SM_CPU_NONE) {
        /* A55 reset vector = địa chỉ SPL trong flash.bin */
        /* SPL load address = 0x20480000 (OCRAM) */
        status = CPU_ResetVectorSet(lm->cpuId,
                                    lm->bootAddr);  /* 0x20480000 */
        if (status != SM_ERR_SUCCESS)
            return status;
    }

    /* Step 3: Release reset cho CPU */
    status = CPU_Reset(lm->cpuId, false);  /* false = deassert reset */
    if (status != SM_ERR_SUCCESS)
        return status;

    lm->state = LM_STATE_ON;
    SM_PRINTF("SM: LM%u started, CPU at 0x%08X\r\n",
              lmId, lm->bootAddr);

    return SM_ERR_SUCCESS;
}
```

### 5.3 SCMI Server — Process Clock Request

```c
/* File: imx-sm/sm/rpc/scmi/rpc_scmi_clock.c
   Xử lý SCMI_CLOCK_RATE_SET từ A55 */

int32_t RPC_SCMI_ClockRateSet(uint32_t lmId, uint32_t channel,
                               const uint32_t *msgPayload)
{
    /* Parse incoming SCMI message */
    uint32_t flags    = le32toh(msgPayload[0]);
    uint32_t clockId  = le32toh(msgPayload[1]);
    uint64_t rate     = ((uint64_t)le32toh(msgPayload[3]) << 32)
                       | le32toh(msgPayload[2]);

    SM_PRINTF("SM: SCMI CLK_RATE_SET: clk=%u rate=%llu\r\n",
              clockId, rate);

    /* Kiểm tra quyền truy cập của LM này */
    if (!LM_ClockIsAccessible(lmId, clockId))
        return SM_ERR_NOT_FOUND;

    /* Thực sự set clock rate qua CCM */
    int32_t status = CLK_RateSet(clockId, rate);

    /* Ghi response vào shared memory */
    uint32_t response = (status == SM_ERR_SUCCESS) ? 0 : SM_ERR_DENIED;
    RPC_SCMI_WriteResponse(channel, &response, sizeof(response));

    /* Trigger MU doorbell để A55 biết có response */
    MU_TxTrigger(MU0_BASE);

    return SM_ERR_SUCCESS;
}
```

**SM Log (COM4) — Annotated:**
```
SM: v2.4.0 (NXP System Manager for i.MX95)     ← SM version
SM: Board: mx95evk                              ← Board config
SM: Config: 3 LMs, SCMI v3.2                   ← Config summary
SM: TRDC: 8 domains configured                 ← Resource isolation
SM: Power: 12 domains available                ← Power domains
SM: Clock: 128 clocks available                ← Clock tree
SM: LM0 (M33): ready                           ← SM itself
SM: LM1 (AP): booting...                       ← About to release A55
SM: A55 reset vector: 0x20480000               ← SPL address
SM: LM1 (AP): started                          ← A55 running
SM: SCMI service ready on MU0/MU1              ← Waiting for requests
>                                              ← SM debug monitor prompt
```

### 5.4 SM Debug Monitor — Lệnh Available

```bash
# Gõ trên COM4 (115200 baud):
> help
Commands:
  clock <id>        - Get clock rate
  clock <id> <rate> - Set clock rate
  power <id>        - Get power state
  lm list           - List logical machines
  lm info <id>      - LM info
  lm boot <id>      - Boot LM
  lm shutdown <id>  - Shutdown LM
  perf list         - Performance domains
  sensor <id>       - Read sensor (temp/voltage)
  scmi stat         - SCMI message statistics
  reset             - System reset

> lm list
LM0: M33  [ON]  boot=ROM
LM1: AP   [ON]  boot=0x20480000
LM2: M7   [OFF] boot=0x00000000  ← M7 chưa được load

> clock 24
Clock 24 (SAI3_CLK_ROOT): 12288000 Hz

> sensor 0
Sensor 0 (A55 core temp): 42°C

> scmi stat
Channel 0 (A55): TX=1247 RX=1247 ERR=0
Channel 1 (M7):  TX=0    RX=0    ERR=0  ← M7 chưa active
```

---

## 6. SCMI LMM Protocol — Source Code Thực Tế

Đây là source code **thực tế** từ NXP patch (Peng Fan, 2025) được submit upstream:

### 6.1 imx-sm-lmm.c — LMM Protocol Driver (U-Boot side)

```c
/* File: drivers/firmware/scmi/vendors/imx/imx-sm-lmm.c
   Source: https://www.mail-archive.com/u-boot@lists.denx.de/msg555411.html
   Author: Peng Fan <peng.fan@nxp.com>, NXP, 2025 */

/* Message IDs cho LMM Protocol (vendor protocol 0x80) */
enum scmi_imx_lmm_protocol_cmd {
    SCMI_IMX_LMM_ATTRIBUTES        = 0x3,
    SCMI_IMX_LMM_BOOT              = 0x4,  /* Boot một LM */
    SCMI_IMX_LMM_RESET             = 0x5,
    SCMI_IMX_LMM_SHUTDOWN          = 0x6,  /* Shutdown một LM */
    SCMI_IMX_LMM_WAKE              = 0x7,
    SCMI_IMX_LMM_SUSPEND           = 0x8,
    SCMI_IMX_LMM_NOTIFY            = 0x9,
    SCMI_IMX_LMM_RESET_REASON      = 0xA,
    SCMI_IMX_LMM_POWER_ON          = 0xB,  /* Power on LM (không boot) */
    SCMI_IMX_LMM_RESET_VECTOR_SET  = 0xC,  /* Set reset vector trước khi boot */
};

/* Set reset vector cho một CPU trong LM (dùng trước khi boot M7) */
int scmi_imx_lmm_reset_vector_set(struct udevice *dev, u32 lmid, u32 cpuid,
                                   u32 flags, u64 vector)
{
    struct scmi_imx_lmm_reset_vector_set_in in = {
        .lmid            = lmid,
        .cpuid           = cpuid,
        .flags           = flags,
        .resetvectorlow  = vector & 0xFFFFFFFF,
        .resetvectorhigh = vector >> 32,
    };
    s32 status;
    struct scmi_msg msg = {
        .protocol_id = SCMI_PROTOCOL_ID_IMX_LMM,   /* 0x80 */
        .message_id  = SCMI_IMX_LMM_RESET_VECTOR_SET,
        .in_msg      = (u8 *)&in,
        .in_msg_sz   = sizeof(in),
        .out_msg     = (u8 *)&status,
        .out_msg_sz  = sizeof(status),
    };
    int ret;

    ret = devm_scmi_process_msg(dev, &msg);
    if (ret)
        return ret;
    return scmi_to_linux_errno(le32_to_cpu(status));
}

/* Boot một LM (ví dụ: boot M7 khi remoteproc start) */
int scmi_imx_lmm_power_boot(struct udevice *dev, u32 lmid, bool boot)
{
    s32 status;
    struct scmi_msg msg = {
        .protocol_id = SCMI_PROTOCOL_ID_IMX_LMM,   /* 0x80 */
        .message_id  = boot ? SCMI_IMX_LMM_BOOT : SCMI_IMX_LMM_POWER_ON,
        .in_msg      = (u8 *)&lmid,
        .in_msg_sz   = sizeof(lmid),
        .out_msg     = (u8 *)&status,
        .out_msg_sz  = sizeof(status),
    };
    int ret = devm_scmi_process_msg(dev, &msg);
    if (ret)
        return ret;
    return scmi_to_linux_errno(le32_to_cpu(status));
}

/* Shutdown LM — graceful hoặc force */
int scmi_imx_lmm_shutdown(struct udevice *dev, u32 lmid, bool graceful)
{
    struct scmi_imx_lmm_shutdown_in in = {
        .lmid  = lmid,
        .flags = graceful ? SCMI_IMX_LMM_SHUTDOWN_GRACEFUL : 0,
    };
    s32 status;
    struct scmi_msg msg = {
        .protocol_id = SCMI_PROTOCOL_ID_IMX_LMM,
        .message_id  = SCMI_IMX_LMM_SHUTDOWN,
        .in_msg      = (u8 *)&in,
        .in_msg_sz   = sizeof(in),
        .out_msg     = (u8 *)&status,
        .out_msg_sz  = sizeof(status),
    };
    int ret = devm_scmi_process_msg(dev, &msg);
    if (ret)
        return ret;
    return scmi_to_linux_errno(le32_to_cpu(status));
}
```

### 6.2 imx-rproc.c — Cách remoteproc dùng LMM (Linux side)

```c
/* File: drivers/remoteproc/imx_rproc.c
   Khi Linux muốn start M7 qua remoteproc */

static int imx95_rproc_start(struct rproc *rproc)
{
    struct imx_rproc *priv = rproc->priv;
    int ret;

    /* Step 1: Set M7 entry point qua SCMI LMM */
    ret = scmi_imx_lmm_reset_vector_set(
        priv->scmi_dev,
        priv->lmm_id,      /* M7 LM ID (từ DTS: fsl,lmm-id) */
        priv->cpu_id,      /* M7 CPU ID (từ DTS: fsl,cpu-id) */
        0,                 /* flags */
        rproc->bootaddr    /* M7 firmware entry point */
    );
    if (ret)
        return ret;

    /* Step 2: Boot M7 LM qua SCMI LMM_BOOT */
    ret = scmi_imx_lmm_power_boot(priv->scmi_dev,
                                   priv->lmm_id, true);
    if (ret)
        return ret;

    dev_info(&rproc->dev, "M7 started at 0x%llx\n", rproc->bootaddr);
    return 0;
}

static int imx95_rproc_stop(struct rproc *rproc)
{
    struct imx_rproc *priv = rproc->priv;

    return scmi_imx_lmm_shutdown(priv->scmi_dev,
                                  priv->lmm_id, true);
}

/* DTS binding cho M7 remoteproc trên i.MX95 */
/*
cm7: remoteproc@0 {
    compatible = "fsl,imx95-cm7";        ← i.MX95 specific
    fsl,lmm-id = <2>;                    ← LM2 = M7 LM
    fsl,cpu-id = <3>;                    ← CPU index trong SM
    mboxes = <&mu2 0 0>, <&mu2 0 1>;    ← MU2 TX/RX
    memory-region = <&vdev0vring0>, <&vdev0vring1>, <&vdev0buffer>;
    status = "okay";
};
*/
```

---

## 7. U-Boot SPL — Source Code Thực Tế

### 7.1 spl.c — imx95_evk (từ NXP upstream patch)

```c
/* File: board/freescale/imx95_evk/spl.c
   Source: Re: [PATCH v2 16/17] imx95_evk
   Author: Ye Li, Alice Guo — NXP */

#include <init.h>
#include <asm/arch/clock.h>
#include <asm/arch/imx-regs.h>
#include <asm/arch/sys_proto.h>
#include <asm/mach-imx/boot_mode.h>
#include <power/pmic.h>
#include <scmi_agent.h>
#include <scmi_protocols.h>
#include "../../../dts/upstream/src/arm64/freescale/imx95-power.h"

DECLARE_GLOBAL_DATA_PTR;

void spl_board_init(void)
{
    int ret;
    u32 state = 0;
    struct udevice *dev;

    /* QUAN TRỌNG: Phải probe MU trước khi có bất kỳ output nào
       vì SCMI qua MU cần cho clock init (UART clock từ SCMI) */
    ret = imx9_probe_mu();
    if (ret)
        hang();   /* Nếu MU lỗi → không có gì output được → hang */

    /* Init CPU (cache, MMU minimal) */
    arch_cpu_init();

    /* Board specific early init (GPIO power, etc.) */
    board_early_init_f();

    /* Sau khi MU + SCMI ready → có thể bật UART console */
    preloader_console_init();

    /* In SOC revision và Lifecycle (debug info) */
    debug("SOC: 0x%x\n", gd->arch.soc_rev);
    debug("LC: 0x%x\n", gd->arch.lifecycle);

    /* Set A55 frequency to max via SCMI */
    clock_init_late();

    /*
     * Kiểm tra DDR domain đã được power up chưa.
     * OEI phải đã khởi tạo DDR trước khi SPL chạy.
     * Nếu DDR vẫn OFF → panic (cần fix OEI)
     */
    ret = uclass_get_device_by_name(UCLASS_CLK, "protocol@14", &dev);
    if (ret)
        printf("%s: cannot get SCMI clock\n", __func__);

    ret = scmi_pwd_state_get(dev, IMX95_PD_DDR, &state);
    if (ret) {
        printf("scmi_pwd_state_get: DDR domain query failed %d\n", ret);
    } else {
        if (state == BIT(30)) {
            panic("DDRMIX is powered OFF — OEI did not run!\n");
        } else {
            printf("DDRMIX: powered UP, DDR ready\n");
            /* DDR đã được OEI init → SPL chỉ cần dram_init() để get size */
            dram_init();
        }
    }
}

/* SPL load U-Boot + ATF từ eMMC/SD */
void board_init_r(gd_t *dummy1, ulong dummy2)
{
    /* spl_board_init() đã được gọi trước đó */
    /* Bây giờ load secondary container (ATF + U-Boot) */

    /* Tìm boot device (MMC1 = eMMC, MMC2 = SD) */
    u32 boot_dev = spl_boot_device();

    /* Load images */
    spl_load_image(boot_dev);

    /* Không return — jump đến ATF BL31 */
    /* jump_to_image_no_args() → bl31_entrypoint */
}
```

**SPL Log (COM3) — Annotated với timestamp:**
```
                                    ← [T+55ms] A55 released by SM
U-Boot SPL 2024.04-lf_v2024.04     ← SPL banner
                                    ← [T+56ms]
DDRMIX: powered UP, DDR ready       ← Confirm OEI đã init DDR
SOC: 0x95960010                     ← i.MX95, rev A1
LC: 0x0010                          ← Lifecycle: OEM_OPEN
NOTICE: BL31: v2.12.0(release)      ← ATF BL31 bắt đầu [T+58ms]
NOTICE: BL31: Built : Apr 2025
```

### 7.2 imximage.cfg — Container config (thực tế)

```ini
/* File: arch/arm/mach-imx/imx9/scmi/imximage.cfg */
/* Đây là file config cho mkimage build flash.bin cho SCMI boot mode */

BOOT_FROM SD
SOC_TYPE IMX9

/* Container 1: OEI + SM (dùng SCMI/SM mode) */
APPEND mx95a0-ahab-container.img
CONTAINER
IMAGE OEI  m33-oei-ddrfw.bin  0x1ffc0000   /* DDR OEI: train DDR */
HOLD 0x10000                               /* Wait for DDR stable */
IMAGE OEI  oei-m33-tcm.bin   0x1ffc0000   /* TCM config OEI */
IMAGE EXEC m33_image.bin      0x1ffc0000   /* System Manager */

/* Container 2: ATF + U-Boot (cho A55) */
CONTAINER
IMAGE A55  bl31.bin           0x8a200000   /* ATF BL31 vào OCRAM */
IMAGE A55  u-boot.bin         CONFIG_TEXT_BASE /* U-Boot: 0x90200000 */
```

---

## 8. ATF BL31 — PSCI & STR Source Code Thực Tế

### 8.1 imx95_bl31_setup.c

```c
/* File: plat/imx/imx95/imx95_bl31_setup.c (imx-atf repo) */

void bl31_early_platform_setup2(u_register_t arg0, u_register_t arg1,
                                 u_register_t arg2, u_register_t arg3)
{
    /* arg0 = FDT address (từ SPL), arg1 = BL33 (U-Boot) entry */

    /* Setup console trên LPUART1 (A55 UART) */
    console_imx_uart_register(IMX_BOOT_UART_BASE,    /* 0x44380000 */
                              IMX_BOOT_UART_CLK_IN_HZ,
                              IMX_CONSOLE_BAUDRATE,
                              &imx95_console);

    /* Lưu thông tin để sau jump vào BL33 */
    bl33_image_ep_info.pc     = BL33_BASE;     /* 0x90200000 U-Boot */
    bl33_image_ep_info.spsr   = get_el2_daif_spsr();
    SET_SECURITY_STATE(bl33_image_ep_info.h.attr, NON_SECURE);
}

void bl31_platform_setup(void)
{
    /* Init GIC-700 */
    plat_imx_gic_driver_init();
    plat_gic_init();

    /* Setup PSCI */
    imx_setup_power_domains();

    /* Gọi SM qua SCMI để thông báo BL31 sẵn sàng */
    /* Điều này cho phép SM biết A55 đã bước vào secure world */
    imx9_scmi_setup_resources();
}

/* BL31 service runtime — xử lý SMC calls từ Linux */
void bl31_main(void)
{
    NOTICE("BL31: v%s\n", version_string);
    NOTICE("BL31: Built : %s, %s\n", build_date, build_time);

    /* Runtime services:
       1. PSCI (cpu on/off/suspend) via SMC
       2. SCMI proxy: Linux → SMC → ATF → MU → SM
       3. TSPD nếu có OP-TEE
    */
    runtime_svc_init();

    /* Jump vào U-Boot */
    bl31_prepare_next_image_entry();
    console_flush();
    bl31_run_next_image(&bl33_image_ep_info);
}
```

### 8.2 PSCI CPU_ON — Release A55 Core 1-5

```c
/* File: plat/imx/imx95/imx95_psci.c */

/* Được gọi qua SMC khi Linux muốn bật thêm A55 core */
int imx_pwr_domain_on(u_register_t mpidr)
{
    /* Decode core ID từ MPIDR */
    unsigned int cpu = plat_core_pos_by_mpidr(mpidr);
    /* cpu = 0 (A55-0) đến 5 (A55-5) */

    /* Set wakeup entry point cho core */
    /* Core sẽ bắt đầu tại secondary_entry sau reset */
    uintptr_t ep = (uintptr_t)&imx_mailbox[cpu];
    *ep = (uintptr_t)&secondary_entry;  /* ARM64 entry */

    /* Yêu cầu SM power on core qua SCMI */
    /* SCMI PERF hay POWER domain request */
    int ret = imx9_scmi_cpu_start(cpu);
    if (ret)
        return PSCI_E_INTERN_FAIL;

    return PSCI_E_SUCCESS;
}

/* Được gọi khi Linux suspend (STR) */
void imx_domain_suspend(const psci_power_state_t *target_state)
{
    unsigned int cpu = plat_my_core_pos();

    /* Chỉ xử lý system-level suspend khi đây là last core */
    if (is_local_state_retn(SYSTEM_PWR_STATE(target_state))) {

        /* Lưu GIC context trước khi mất power */
        plat_gic_save(cpu, &imx_gicv3_ctx);

        /* Yêu cầu SM cho DDR vào retention mode */
        /* SM sẽ:
           1. Tắt DDR self-refresh countdown
           2. Gửi MR4 command cho DDR chip
           3. Gate DDR PHY clocks
        */
        imx9_scmi_sys_suspend();

        /* Set wakeup source (GPC/BBSM) */
        imx_set_sys_wakeup(cpu, true);

        /* Gặp vấn đề: DDR code phải chạy từ SRAM vì DDR sẽ vào retention */
        /* Copy suspend code sang SRAM trước khi DDR off */
        imx_copy_bl31_to_sram();
    }

    /* Enter low-power state */
    /* GPC sẽ power gate A55 cluster */
    isb();
    dsb();
    __asm volatile ("wfi");
}

/* Resume path — được gọi từ SRAM sau wakeup */
void imx_domain_suspend_finish(const psci_power_state_t *target_state)
{
    unsigned int cpu = plat_my_core_pos();

    if (is_local_state_retn(SYSTEM_PWR_STATE(target_state))) {
        /* DDR exit retention — khôi phục từ saved state */
        imx9_scmi_sys_resume();

        /* Restore GIC state */
        plat_gic_restore(cpu, &imx_gicv3_ctx);

        /* Clear wakeup sources */
        imx_set_sys_wakeup(cpu, false);
    }
}
```

---

## 9. Linux Kernel Boot — Driver Init với Source Code

### 9.1 SCMI Clock Driver Init (imx95-specific)

```c
/* File: drivers/clk/imx/clk-imx95.c */

static int imx95_clk_probe(struct platform_device *pdev)
{
    struct device *dev = &pdev->dev;
    struct imx95_clk_priv *priv;
    const struct scmi_protocol_handle *ph;
    int ret;

    /* Lấy SCMI clock protocol handle */
    priv = devm_kzalloc(dev, sizeof(*priv), GFP_KERNEL);
    ph = devm_scmi_protocol_get(dev, SCMI_PROTOCOL_CLOCK, &imx95_scmi_clk);
    if (IS_ERR(ph))
        return dev_err_probe(dev, PTR_ERR(ph), "Failed SCMI clock\n");

    priv->ph = ph;

    /* Lấy số lượng clocks từ SM */
    /* SM trả về: 128 clocks trên i.MX95 */
    ret = ph->clk_ops->count_get(ph);
    if (ret < 0)
        return ret;
    priv->num_clks = ret;
    dev_info(dev, "SCMI: %d clock domains available\n", priv->num_clks);

    /* Đăng ký tất cả clocks với CCF (Common Clock Framework) */
    for (int i = 0; i < priv->num_clks; i++) {
        char name[32];
        scmi_clk_id_to_name(ph, i, name, sizeof(name));

        priv->hws[i] = imx95_clk_register_scmi(dev, ph, i, name);
        if (IS_ERR(priv->hws[i]))
            dev_warn(dev, "Failed to register clk %d (%s)\n", i, name);
    }

    /* Đăng ký với clk framework */
    return of_clk_add_hw_provider(dev->of_node,
                                   of_clk_hw_onecell_get,
                                   priv->clk_data);
}

/* Khi SAI driver muốn enable SAI3 clock */
static int imx95_scmi_clk_prepare(struct clk_hw *hw)
{
    struct imx95_scmi_clk *clk = to_imx95_scmi_clk(hw);
    int ret;

    /* SCMI_CLOCK_CONFIG_SET với enable=true */
    ret = clk->ph->clk_ops->enable(clk->ph, clk->id);
    if (ret)
        pr_err("SCMI: Failed to enable clock %d: %d\n", clk->id, ret);

    return ret;
    /*
     * Điều này trigger:
     * A55 → SCMI msg via MU0 → SM → CCM register write
     * CCM SAI3_CLK_ROOT → enable clock
     */
}
```

**Kernel Log khi SAI probe (dmesg):**
```
[    0.450000] imx-clk-scmi: 128 clock domains from SCMI
[    0.520000] fsl-sai 42650000.sai: SCMI clock SAI3_CLK_ROOT=12288000Hz
[    0.521000] fsl-sai 42650000.sai: registered DAI 'sai3'
[    0.522000] fsl-sai 42650000.sai: probe OK
```

### 9.2 imx_rproc — RPMsg M7 Start

```c
/* File: drivers/remoteproc/imx_rproc.c
   Đây là driver nhận "echo start > remoteproc0/state" từ user */

static int imx_rproc_start(struct rproc *rproc)
{
    struct imx_rproc *priv = rproc->priv;
    int ret;

    if (priv->rproc_type == IMX95_CM7) {
        /* i.MX95: dùng SCMI LMM để start M7 */

        /* 1. Set M7 entry point */
        ret = scmi_imx_lmm_reset_vector_set(
            priv->scmi_dev,
            priv->lmm_id,       /* từ DTS fsl,lmm-id = 2 */
            priv->cpu_id,       /* từ DTS fsl,cpu-id = 3 */
            0,
            rproc->bootaddr     /* địa chỉ binary đã load vào TCM/DDR */
        );
        if (ret) {
            dev_err(&rproc->dev, "LMM reset vector set failed: %d\n", ret);
            return ret;
        }

        /* 2. Boot M7 LM */
        ret = scmi_imx_lmm_power_boot(priv->scmi_dev,
                                       priv->lmm_id, true);
        if (ret) {
            dev_err(&rproc->dev, "LMM boot failed: %d\n", ret);
            return ret;
        }

    } else {
        /* i.MX8/i.MX9x: dùng SRC register trực tiếp */
        regmap_update_bits(priv->regmap, priv->rsrc.entry.bit,
                           priv->rsrc.entry.mask, 0);
    }

    dev_info(&rproc->dev, "Remote proc started (addr=0x%llx)\n",
             rproc->bootaddr);
    return 0;
}

/* DTS mẫu cho i.MX95 M7 remoteproc: */
/*
cm7: remoteproc {
    compatible = "fsl,imx95-cm7";
    fsl,lmm-id = <2>;          ← LM2 trong SM config
    fsl,cpu-id = <3>;          ← CPU ID 3 = M7 trong SM
    mboxes = <&mu2 0 0>,       ← MU2 channel 0 (A55→M7 notify)
             <&mu2 0 1>;       ← MU2 channel 1 (M7→A55 notify)
    mbox-names = "tx", "rx";
    memory-region = <&vdev0vring0>,   ← TX vring (A55→M7)
                    <&vdev0vring1>,   ← RX vring (M7→A55)
                    <&vdev0buffer>,   ← Message buffer pool
                    <&m7_reserved>;   ← M7 code/data
    fsl,auto-boot;             ← load firmware khi Linux boot
    fsl,auto-boot-firmware = "imx/m7-rpmsg.elf";
};
*/
```

---

## 10. Inter-Core: SCMI Transport — Packet Level

### 10.1 Flow Đầy Đủ: Linux gọi clk_enable() → SM xử lý

```
Linux (A55 EL1)
  drivers/clk/imx/clk-imx95.c: imx95_scmi_clk_prepare()
    │  ph->clk_ops->enable(ph, clock_id=24)
    ▼
  drivers/firmware/arm_scmi/clock.c: scmi_clock_enable()
    │  scmi_request = scmi_build_cmd(SCMI_PROTO_CLOCK, CLOCK_CONFIG_SET)
    │  Ghi vào shmem @ 0x44631000:
    │    [+0x00] reserved   = 0x00000000
    │    [+0x04] status     = 0x00000001  (set busy bit)
    │    [+0x10] flags      = 0x00000000
    │    [+0x14] length     = 0x0000000C  (12 bytes: header + payload)
    │    [+0x18] msg_header = 0x00140702  (proto=0x14, msg=0x07=CONF, token=2)
    │    [+0x1C] clock_id   = 0x00000018  (= 24 decimal, SAI3 clock)
    │    [+0x20] attributes = 0x00000001  (enable=1)
    ▼
  drivers/firmware/arm_scmi/smc.c: scmi_smc_send_message()
    │  SMC call: hvc #0 với SMC_ID = 0xC2000002 (SCMI SMC ID cho i.MX)
    │  x0 = shmem PA address = 0x44631000
    ▼

[EL2/EL3 — ATF BL31]
  plat/imx/imx95/imx95_scmi.c: imx_scmi_smc_handler()
    │  Đọc shmem address từ x0
    │  Copy message vào M33 accessible shmem @ 0x44611000
    │  Write MU0 doorbell: mmio_write_32(MU0_BASE + MU_GCR, 0x1)
    ▼

[MU0 hardware interrupt]
    │  M33 nhận MU interrupt

[M33 — System Manager]
  sm/src/rpc/scmi/rpc_scmi.c: RPC_SCMI_Process(channel=0)
    │  Đọc message từ shmem
    │  Parse header: proto=0x14 (CLOCK), msg=0x07 (CONFIG_SET)
    │  Route to clock handler
    ▼
  sm/src/rpc/scmi/rpc_scmi_clock.c: RPC_SCMI_ClockConfigSet()
    │  Kiểm tra LM permission: LM1 (AP) có quyền set clock 24?
    │  → YES (theo mx95evk.cfg)
    │  Gọi CLK_ConfigSet(24, true)  ← actual CCM write
    │  CCM base: 0x44450000
    │  Write CCM_CLOCKROOTn_CONTROL → enable SAI3 clock
    │
    │  Ghi response vào shmem:
    │    [+0x04] status     = 0x00000000  (clear busy bit)
    │    [+0x18] msg_header = (echo token)
    │    [+0x1C] response   = 0x00000000  (SUCCESS)
    │
    │  Trigger MU0 interrupt → A55
    ▼

[A55 — Linux kernel interrupt]
  drivers/firmware/arm_scmi/smc.c: scmi_smc_rx_callback()
    │  Đọc response từ shmem
    │  Complete pending request
    ▼
  drivers/clk/imx/clk-imx95.c: imx95_scmi_clk_prepare() returns 0 (OK)
  SAI clock enabled!
```

### 10.2 Đo Latency SCMI Round-trip

```bash
# Từ Linux, đo thời gian một SCMI transaction
adb shell

# Method 1: ftrace SCMI
echo 'arm_scmi:scmi_xfer_begin arm_scmi:scmi_xfer_end' > /sys/kernel/tracing/set_event
echo 1 > /sys/kernel/tracing/tracing_on
# Trigger một SCMI call
cat /sys/kernel/debug/clk/sai3_root/clk_rate  # Đây sẽ gọi SCMI
cat /sys/kernel/tracing/trace | grep scmi

# Typical output:
#     kworker/0:0-42    [000] ... scmi_xfer_begin: id=7 proto=20 token=42
#     kworker/0:0-42    [000] ... scmi_xfer_end:   id=7 proto=20 token=42 status=0
# Thời gian giữa begin và end ≈ 50-200 µs (tùy system load)
```

---

## 11. Inter-Core: RPMsg/MU — Source Code & Virtio Detail

### 11.1 RPMsg-Lite (M7 side FreeRTOS) — Source Code

```c
/* File: middleware/multicore/rpmsg-lite/lib/rpmsg_lite.c
   Source: https://github.com/nxp-mcuxpresso/rpmsg-lite */

/* Step 1: Master (A55 Linux) init phía Linux */
/* Điều này xảy ra trong imx_rproc_start() sau khi M7 đã boot */

/* Step 2: M7 FreeRTOS init */
rpmsg_lite_instance_t my_rpmsg_instance;

struct rpmsg_lite_instance *my_rpmsg =
    rpmsg_lite_remote_init(
        (void *)0xA8000000UL,     /* Base của shared memory */
        RPMSG_LITE_LINK_ID,       /* Link ID matches Linux side */
        RL_NO_FLAGS,
        &my_rpmsg_instance
    );

/* rpmsg_lite_remote_init internally:
   1. Đọc vdev header tại shmem base
      struct fw_rsc_vdev {
          uint32_t type;       = RSC_VDEV (3)
          uint32_t id;         = 7 (VIRTIO_ID_RPMSG)
          uint32_t notifyid;
          uint32_t dfeatures;
          uint32_t gfeatures;
          uint32_t config_len;
          uint8_t  status;     = DRIVER_OK khi A55 set
          uint8_t  num_of_vrings = 2;
          ...
      };
   2. Parse 2 vrings:
      vring0 @ 0xA8000000: A55 TX → M7 RX
      vring1 @ 0xA8008000: M7 TX → A55 RX
   3. Đợi A55 set VIRTIO_CONFIG_S_DRIVER_OK
*/

/* Step 3: Đợi link up (A55 set status bit) */
rpmsg_lite_wait_for_link_up(my_rpmsg, RL_BLOCK);
/* Polling: while (!(my_rpmsg->vdev->status & VIRTIO_CONFIG_S_DRIVER_OK)); */

/* Step 4: Tạo endpoint và queue */
rpmsg_queue_t my_queue_instance;
rpmsg_queue_handle_t queue =
    rpmsg_queue_create(my_rpmsg, &my_queue_instance);

rpmsg_lite_ept_static_context_t my_ept_context;
struct rpmsg_lite_endpoint *my_ept =
    rpmsg_lite_create_ept(my_rpmsg,
                          30,            /* EPT address = 30 */
                          rpmsg_queue_rx_cb,
                          queue,
                          &my_ept_context);

/* Gửi nameservice announcement: "rpmsg-openamp-demo-channel" */
rpmsg_ns_announce(my_rpmsg, my_ept, "rpmsg-openamp-demo-channel",
                  RL_NS_CREATE);
```

### 11.2 Virtio Vring Detail — Shared Memory Structure

```c
/* Vring layout tại 0xA8000000 (vring0: A55 TX → M7 RX) */

/* struct vring được define trong lib/include/virtio_ring.h */
struct vring {
    unsigned int num;                    /* 256 descriptors */
    struct vring_desc {
        uint64_t addr;                   /* Physical address của buffer */
        uint32_t len;                    /* Buffer length */
        uint16_t flags;                  /* VRING_DESC_F_NEXT, etc. */
        uint16_t next;                   /* Index của descriptor tiếp theo */
    } *desc;                             /* @ 0xA8000000 */

    struct vring_avail {
        uint16_t flags;
        uint16_t idx;                    /* Producer index (A55 tăng) */
        uint16_t ring[256];              /* Available descriptor indices */
    } *avail;                            /* @ 0xA8000000 + 256*16 = 0xA8001000 */

    struct vring_used {
        uint16_t flags;
        uint16_t idx;                    /* Consumer index (M7 tăng) */
        struct vring_used_elem {
            uint32_t id;
            uint32_t len;
        } ring[256];
    } *used;                             /* @ 0xA8001010 */
};

/* Để gửi message từ A55 → M7: */
/*
 1. A55 lấy free descriptor từ free list
 2. A55 ghi data vào buffer @ (0xA8010000 + desc_idx * 512)
 3. A55 điền desc: addr=buffer_pa, len=data_len
 4. A55 thêm desc_idx vào avail->ring[avail->idx % 256]
 5. A55 tăng avail->idx (memory barrier!)
 6. A55 kick M7 qua MU2 doorbell: write to MU2_GCR
 7. MU2 tạo interrupt trên M7
 8. M7 nhận interrupt, đọc avail ring, process buffer
 9. M7 trả buffer về: thêm vào used->ring, tăng used->idx
10. M7 kick A55 qua MU2: write MU2_TR0
*/
```

### 11.3 Linux Side — RPMsg Data Path

```c
/* File: drivers/rpmsg/virtio_rpmsg_bus.c */

/* Khi user space write vào /dev/rpmsg0 */
static ssize_t rpmsg_char_write(struct file *filp,
                                const char __user *buf,
                                size_t count, loff_t *offp)
{
    struct rpmsg_eptdev *eptdev = filp->private_data;
    void *kbuf;
    int ret;

    /* Allocate kernel buffer */
    kbuf = kmalloc(count, GFP_KERNEL);
    copy_from_user(kbuf, buf, count);

    /* Send via rpmsg endpoint */
    ret = rpmsg_send(eptdev->ept, kbuf, count);
    /* rpmsg_send → virtqueue_add_buf → update avail ring → kick via MU2 */

    kfree(kbuf);
    return count;
}

/* Receive callback khi M7 gửi data */
static int rpmsg_char_cb(struct rpmsg_channel *rpdev, void *data,
                         int len, void *priv, u32 src)
{
    struct rpmsg_eptdev *eptdev = priv;

    /* Đẩy data vào RX FIFO để user space đọc */
    kfifo_in_spinlocked(&eptdev->rx_fifo, data, len, &eptdev->rx_lock);
    wake_up_interruptible(&eptdev->readq);  /* Wake up blocked read() */

    return 0;
}
```

**RPMsg Log (dmesg) khi M7 start:**
```
[   10.125000] imx-rproc imx-rproc: loading firmware imx/m7-rpmsg.elf
[   10.456000] imx-rproc imx-rproc: SCMI LMM: setting vector 0x00000000
[   10.457000] imx-rproc imx-rproc: SCMI LMM: booting LM2 (M7)
[   10.500000] virtio_rpmsg_bus virtio0: rpmsg host is online
[   10.600000] virtio_rpmsg_bus virtio0: creating channel rpmsg-openamp-demo-channel addr 0x1e
[   10.601000] rpmsg_char: new /dev/rpmsg0 (src 0x1 dst 0x1e)
```

---

## 12. STR — Suspend to RAM: Source Code End-to-End

### 12.1 Linux PM Suspend Path — Chi Tiết

```c
/* kernel/power/suspend.c */

int pm_suspend(suspend_state_t state)  /* state = PM_SUSPEND_MEM */
{
    int error;

    /* Phase 1: Freeze userspace */
    error = suspend_freeze_processes();
    /* dmesg: "Freezing user space processes" */

    /* Phase 2: Suspend devices (bottom-up, leaf first) */
    error = dpm_suspend_start(PMSG_SUSPEND);
    /* Gọi .suspend() của mỗi driver:
       - fsl_sai_suspend(): tắt SAI DMA, save SAI registers
       - imx_rpmsg_suspend(): notify M7 A55 đang suspend
       - scmi_clk_suspend(): không làm gì (SM giữ clocks)
       - xhci_suspend(): gây timeout nếu USB device còn active!
    */

    /* Phase 3: Late suspend (interrupt disabled) */
    error = dpm_suspend_late(PMSG_SUSPEND);

    /* Phase 4: Noirq suspend (GIC disabled) */
    error = dpm_suspend_noirq(PMSG_SUSPEND);
    /* dmesg: "PM: noirq suspend of devices" */

    /* Phase 5: Enter system sleep — arch code */
    error = suspend_ops->enter(state);
    /* → cpu_suspend() → SMC → ATF */
}

/* arch/arm64/kernel/suspend.c */
int cpu_suspend(unsigned long arg)
{
    /* Save CPU context (registers, SCTLR, etc.) to DRAM */
    __cpu_suspend_enter();

    /* SMC call để vào secure world */
    /* psci_ops.cpu_suspend(target_state, pa_of_resume_fn) */
    arm_smccc_smc(PSCI_CPU_SUSPEND,
                  PSCI_POWER_STATE_TYPE_POWERDOWN,
                  (uintptr_t)cpu_resume,  /* resume entry point */
                  0, 0, 0, 0, 0, &res);

    /* Nếu suspend thành công, code dưới đây KHÔNG chạy ngay.
       Nó sẽ chạy sau khi system resume từ warm boot */
    __cpu_suspend_exit();
    return 0;
}
```

### 12.2 ATF Suspend — DDR Retention

```c
/* File: plat/imx/imx95/imx95_psci.c (imx-atf) */

void imx_domain_suspend(const psci_power_state_t *target_state)
{
    unsigned int cpu = plat_my_core_pos();

    if (is_local_state_retn(SYSTEM_PWR_STATE(target_state))) {
        /* === SYSTEM SUSPEND (last core going down) === */

        /* 1. Lưu GIC redistributor context */
        plat_gic_save(cpu, &imx_gicv3_ctx);

        /* 2. Cấu hình wakeup sources */
        imx_set_sys_wakeup(cpu, true);
        /* Ví dụ: enable BBSM RTC alarm wakeup */
        /* mmio_write_32(BBNSM_BASE + BBNSM_CTRL,
                         BBNSM_CTRL_RTC_ALARM_EN); */

        /* 3. Notify SM system đang suspend (SCMI sys_suspend) */
        /* SM sẽ chuẩn bị:
           - Save power domain states
           - Prepare DDR for retention
        */
        arm_smccc_smc(IMX9_SIP_GPC, IMX9_SIP_GPC_PM_DOMAIN,
                      IMX9_GPC_SUSPEND, 0, 0, 0, 0, 0, &res);

        /* 4. Đưa DDR vào self-refresh (retention) */
        /* QUAN TRỌNG: Sau đây code phải ở SRAM vì DDR sẽ bị gated! */
        imx_dram_enter_retention();
        /*
         * imx_dram_enter_retention():
         *   a. Set DDRC PWRCTL.selfref_sw = 1 (force self-refresh)
         *   b. Đợi DDRC STAT.operating_mode == 3 (self-refresh)
         *   c. Gate DDR PHY DFI clock
         *   d. Tắt DRAM controller clock root via SCMI/CCM
         */

        /* 5. Copy warm boot code vào OCRAM (SRAM) */
        /* Vì DDR đang retention, warm boot handler phải ở SRAM */
        memcpy(OCRAM_BASE, &__sram_text_start,
               (uintptr_t)&__sram_text_end - (uintptr_t)&__sram_text_start);
        flush_dcache_range(OCRAM_BASE,
                           OCRAM_BASE + sram_code_size);
    } else {
        /* CPU-level suspend (không phải system) */
        plat_gic_cpuif_disable();
        plat_gic_pcpu_save(cpu, &imx_gicv3_pcpu_ctx);
    }

    /* 6. Đặt warm resume entry point */
    /* Sau wake up, M33 SM sẽ kick A55, A55 bắt đầu tại địa chỉ này */
    mmio_write_32(SRC_BASE + SRC_GPR0, (uintptr_t)&bl31_warm_entrypoint);
    mmio_write_32(SRC_BASE + SRC_GPR1, 0);

    /* 7. GPC (General Power Controller) power gate A55 */
    /* A55 cluster sẽ bị power gated sau WFI */
    imx_set_sys_lpm(cpu, true);

    /* CPU enters WFI → hardware power gates A55 */
    isb();
    dsb();
    wfi();
    /* ===== SYSTEM IS SLEEPING HERE ===== */
    /* Code tiếp tục sau đây khi resume */
}
```

### 12.3 SM trong khi A55 Sleep

```c
/* File: imx-sm/sm/src/main.c — SM vẫn running trong STR */

/* SM phát hiện A55 đã suspend và bắt đầu quản lý wakeup */

/* Khi RTC alarm fire (ví dụ): */
static void SM_RtcAlarmHandler(void)
{
    SM_PRINTF("SM: RTC alarm! Starting wake-up sequence\r\n");

    /* 1. Khôi phục DDR từ retention */
    SM_DdrExitRetention();
    /*
     * SM_DdrExitRetention():
     *   a. Enable DDR clock root via CCM
     *   b. Enable DDR PHY DFI clock
     *   c. Clear DDRC PWRCTL.selfref_sw
     *   d. Đợi DDRC exit self-refresh
     *   e. Verify DDR accessible
     */

    /* 2. Power on A55 domain */
    PWR_LmPower(LM_AP, SM_POWER_ON);

    /* 3. Kick A55 warm boot via MU */
    /* A55 sẽ bắt đầu tại bl31_warm_entrypoint đã lưu */
    MU_TxTrigger(MU0_BASE);

    SM_PRINTF("SM: A55 resume triggered\r\n");
}
```

### 12.4 Resume Path — Từ ATF → Linux

```c
/* Warm boot resume tại SRAM address bl31_warm_entrypoint */

void bl31_warm_entrypoint(void)
{
    /* Chạy từ SRAM, DDR đã được SM khôi phục */

    /* 1. Restore CPU context (registers) */
    imx_cpu_context_restore();

    /* 2. Re-init GIC distributor (power cycled) */
    plat_gic_redistif_on();

    /* 3. Restore GIC redistributor */
    plat_gic_restore(cpu, &imx_gicv3_ctx);

    /* 4. Clear suspend flags */
    imx_set_sys_wakeup(cpu, false);
    imx_set_sys_lpm(cpu, false);

    /* 5. Return to Linux kernel cpu_suspend() caller */
    bl31_exit_to_normal_world();
    /* Linux tiếp tục từ sau arm_smccc_smc() trong cpu_suspend() */
}

/* Back in Linux kernel (resume path): */
/* kernel/power/suspend.c: */
/* dpm_resume_start() gọi .resume() ngược lại:
   - scmi_clk_resume()     ← RE-negotiate frequencies với SM
   - fsl_sai_resume()      ← Restore SAI registers
   - imx_rpmsg_resume()    ← Notify M7 A55 đã resume
   - xhci_resume()         ← Restore USB controller
*/
```

**STR Dmesg Log đầy đủ:**
```
[  100.000] PM: suspend entry (deep)              ← Echo mem command
[  100.005] Filesystems sync: 0.003 seconds
[  100.010] Freezing user space processes
[  100.012] Freezing user space processes completed (elapsed 0.001s)
[  100.015] OOM killer disabled.
[  100.020] Suspending console(s) (use no_console_suspend to debug)
[  100.030] fsl-sai 42650000.sai: suspend                    ← SAI save
[  100.040] virtio_rpmsg_bus: notifying M7 of suspend        ← RPMsg notify
[  100.100] PM: suspend devices took 0.071 seconds
[  100.101] PM: noirq suspend of devices
[  100.102] Disabling non-boot CPUs ...
[  100.110] CPU1: shutdown                                   ← A55-1 down
[  100.115] CPU2: shutdown                                   ← A55-2 down
[  100.120] CPU3: shutdown
[  100.125] CPU4: shutdown
[  100.130] CPU5: shutdown
[  100.131] PSCI: Entering system sleep                      ← ATF invoked
[  100.132] DDR: entering retention                          ← DDR self-refresh
[  100.133] System entering deep sleep...                    ← WFI, power gate
... [SLEEPING] ...
[COM4 - M33]: SM: RTC alarm! Starting wake-up             ← 30s later
[COM4 - M33]: SM: DDR exit retention OK
[COM4 - M33]: SM: A55 resume triggered
[  130.000] PSCI: Resumed from deep sleep                   ← Back in kernel
[  130.005] CPU1 is up
[  130.010] CPU2 is up
[  130.015] CPU3 is up
[  130.020] CPU4 is up
[  130.025] CPU5 is up
[  130.050] PM: resume devices took 0.025 seconds
[  130.060] PM: resume exit                                  ← Normal operation
[  130.070] fsl-sai 42650000.sai: resumed
[  130.080] audio: ALSA state restored
```

---

## 13. Android Audio Stack — Từ App Xuống SAI Register

### 13.1 AudioTrack → SAI Register Flow

```java
// Android App (Java)
AudioTrack track = new AudioTrack.Builder()
    .setAudioAttributes(new AudioAttributes.Builder()
        .setUsage(AudioAttributes.USAGE_MEDIA)
        .setContentType(AudioAttributes.CONTENT_TYPE_MUSIC)
        .build())
    .setAudioFormat(new AudioFormat.Builder()
        .setSampleRate(48000)
        .setEncoding(AudioFormat.ENCODING_PCM_16BIT)
        .setChannelMask(AudioFormat.CHANNEL_OUT_STEREO)
        .build())
    .setBufferSizeInBytes(AudioTrack.getMinBufferSize(...))
    .build();

track.play();
track.write(pcmData, 0, pcmData.length);  // Write PCM data
```

```cpp
/* frameworks/av/media/libaudioclient/AudioTrack.cpp */
ssize_t AudioTrack::write(const void *buffer, size_t size, bool blocking)
{
    /* Copy data vào shared ring buffer (AudioFlinger sẽ đọc) */
    /* Ring buffer: /dev/ashmem (anonymous shared memory) */
    memcpy(mCblkMemory + mCblk->u.mStreaming.mBuffer,
           buffer + written, toWrite);
    mCblk->u.mStreaming.mBuffer += toWrite;  /* advance write pointer */
    mAudioTrack->step();                      /* notify AudioFlinger */
}
```

```cpp
/* frameworks/av/services/audioflinger/Threads.cpp */
/* MixerThread::threadLoop() — chạy liên tục */
bool AudioFlinger::MixerThread::threadLoop()
{
    while (!exitPending()) {
        /* 1. Mix all active tracks */
        mAudioMixer->process();

        /* 2. Write mixed output tới HAL */
        /* Output buffer = 256 frames = 256 * 4 bytes = 1024 bytes */
        mOutput->stream->write(mOutput->stream,
                               mMixBuffer,
                               mixBufferSize);
        /* ↓ xuống HAL AIDL */
    }
}
```

```cpp
/* hardware/interfaces/audio/ — Audio HAL AIDL */
/* vendor/nxp/imx/audio_hw.cpp (NXP HAL implementation) */
ndk::ScopedAStatus NxpStreamOut::write(
    const std::vector<uint8_t>& buffer,
    int64_t* _aidl_return)
{
    /* Write PCM data via tinyalsa */
    int ret = pcm_write(mPcm,          /* pcm handle cho TAS5828 (card2) */
                        buffer.data(),
                        buffer.size());

    /* pcm_write gọi ALSA ioctl: */
    /* ioctl(pcm_fd, SNDRV_PCM_IOCTL_WRITEI_FRAMES, &xfer); */

    *_aidl_return = (ret == 0) ? buffer.size() : 0;
    return ndk::ScopedAStatus::ok();
}
```

```c
/* sound/core/pcm_lib.c — kernel ALSA */
snd_pcm_sframes_t snd_pcm_lib_write(struct snd_pcm_substream *substream,
                                    const void __user *buf,
                                    snd_pcm_uframes_t frames)
{
    /* Copy từ userspace vào DMA buffer */
    copy_from_user(runtime->dma_area + offset, buf, bytes);

    /* Submit DMA transfer */
    snd_pcm_update_hw_ptr(substream);
}
```

```c
/* sound/soc/fsl/fsl_sai.c — NXP SAI driver */
static int fsl_sai_trigger(struct snd_pcm_substream *substream,
                           int cmd, struct snd_soc_dai *cpu_dai)
{
    struct fsl_sai *sai = snd_soc_dai_get_drvdata(cpu_dai);

    switch (cmd) {
    case SNDRV_PCM_TRIGGER_START:
        /* Enable SAI transmit */
        /* SAI base: 0x42650000 (SAI3 trên i.MX95) */

        /* 1. Enable FIFO DMA request */
        regmap_update_bits(sai->regmap,
                           FSL_SAI_TCSR,    /* 0x42650000 + 0x00 */
                           FSL_SAI_CSR_FRDE,  /* FIFO Request DMA Enable */
                           FSL_SAI_CSR_FRDE);

        /* 2. Enable Transmitter */
        regmap_update_bits(sai->regmap,
                           FSL_SAI_TCSR,
                           FSL_SAI_CSR_TE,    /* Transmitter Enable */
                           FSL_SAI_CSR_TE);
        break;
    }
}

/* SAI Register Map (SAI3 @ 0x42650000): */
/*
Offset   Register   Description
0x00     TCSR       Transmit Control/Status
                    bit 31: TE (Transmitter Enable)
                    bit 5:  FRDE (FIFO Request DMA Enable)
                    bit 4:  FWDE (FIFO Warning DMA Enable)
0x04     TCR1       Transmit Config 1 (FIFO watermark)
0x08     TCR2       Transmit Config 2 (clocking)
                    bit 30: BCS (Bit Clock Swap)
                    bit 25: MSEL (MCLK select)
                    [4:0]:  DIV (Bit clock divider)
0x0C     TCR3       Transmit Config 3 (channel enable)
0x10     TCR4       Transmit Config 4 (frame config)
0x14     TCR5       Transmit Config 5 (word width)
0x20     TDR0       Transmit Data Register (DMA writes here)
0x40     TFR0       Transmit FIFO Register (FIFO level)
0x60     TMR        Transmit Mask Register
*/
```

### 13.2 SAI → I2S → TAS5828 (register dump)

```bash
# Verify SAI hardware state
adb shell devmem 0x42650000 32  # TCSR — should have bit31=1 (TE), bit5=1 (FRDE)
# Expected: 0x80000020

adb shell devmem 0x42650008 32  # TCR2 — clock config
# bit25=1 (MSEL=MCLK), bit[4:0]=0 (DIV=1) for 256fs

adb shell tinymix                # List all mixer controls
# Expected:
# 0: TAS5828 Volume  → verify không bị mute

# Check ALSA PCM state
adb shell cat /proc/asound/card2/pcm0p/sub0/status
# Should show: state: RUNNING

# Verify DMA running
adb shell cat /proc/asound/card2/pcm0p/sub0/hw_params
# Expected:
# access: RW_INTERLEAVED
# format: S16_LE
# subformat: STD
# channels: 2
# rate: 48000 (48000/1)
# period_size: 1024
# buffer_size: 4096

# Check audio_patch (AAOS specific)
adb shell dumpsys media.audio_policy | grep -A 5 "Audio Patch"
```

---

## 14. Debug Cookbook — Lệnh Thực Tế

### 14.1 Boot Sequence Debug

```bash
# ===== SETUP: 4 terminals =====
# COM1 (ttyUSB0): M7 FreeRTOS
# COM3 (ttyUSB2): A55 U-Boot + Kernel
# COM4 (ttyUSB3): M33 System Manager
# ADB: Android

# Terminal 1 — M33 SM Monitor (COM4):
screen /dev/ttyUSB3 115200
# Sau khi boot gõ 'help' để xem commands
> lm list
> clock 24     # SAI3 clock rate

# Terminal 2 — A55 Kernel (COM3):
screen /dev/ttyUSB2 115200
# U-Boot: có thể interrupt với Ctrl+C
# Kernel: dmesg output real-time

# Terminal 3 — ADB:
adb wait-for-device
adb shell dmesg -w  # real-time kernel log

# ===== CHECK BOOT PHASES =====

# Phase 1: OEI ran? (check SM output on COM4)
# Expected: "DDR: Training PASS"

# Phase 2: SPL ran? (check COM3)
# Expected: "U-Boot SPL 2024.04"

# Phase 3: DDR ready?
adb shell cat /proc/meminfo | grep MemTotal
# Expected: MemTotal: ~7800000 kB

# Phase 4: SCMI working?
adb shell cat /sys/kernel/debug/arm-scmi/0/protocols
# Expected: 0x10 0x11 0x13 0x14 0x15 0x19 0x80 0x84

# Phase 5: M7 running?
adb shell cat /sys/class/remoteproc/remoteproc0/state
# Expected: running

# Phase 6: RPMsg working?
adb shell ls /dev/rpmsg*
# Expected: /dev/rpmsg0
```

### 14.2 SCMI Deep Debug

```bash
# Enable SCMI tracing
adb shell
echo 1 > /sys/kernel/debug/tracing/events/arm_scmi/enable
echo 1 > /sys/kernel/debug/tracing/tracing_on

# Trigger một SCMI call (ví dụ: đọc clock rate)
cat /sys/kernel/debug/clk/sai3_root/clk_rate

# Xem trace
cat /sys/kernel/debug/tracing/trace | grep scmi
# Expected output:
# cat-1234 [000] .... scmi_xfer_begin: id=5 proto=0x14 seq=42
# cat-1234 [000] .... scmi_xfer_end:   id=5 proto=0x14 seq=42 ret=0

# Check SCMI error count
cat /sys/kernel/debug/arm-scmi/0/errors
# Expected: 0

# List all SCMI clock domains
cat /sys/kernel/debug/arm-scmi/0/clocks
# Expected: 128 lines, mỗi line là một clock
```

### 14.3 RPMsg Debug

```bash
# Check RPMsg framework
adb shell cat /proc/remoteproc/remoteproc0/state
adb shell cat /proc/remoteproc/remoteproc0/firmware

# Reload M7 firmware
adb shell "echo stop > /sys/class/remoteproc/remoteproc0/state"
adb shell "echo start > /sys/class/remoteproc/remoteproc0/state"
adb shell dmesg | tail -20

# Test RPMsg pingpong
adb shell insmod /vendor/lib/modules/imx_rpmsg_pingpong.ko
adb shell dmesg | grep -E "rpmsg|pingpong"
# Expected:
# imx_rpmsg_pingpong virtio0.rpmsg-openamp-demo-channel.-1.30: new channel: 0x1 -> 0x1e!
# get 1 (src: 0x1e)
# get 2 (src: 0x1e)

# Debug virtio vring
adb shell cat /sys/bus/virtio/devices/virtio0/features
adb shell ls /sys/bus/virtio/devices/virtio0/

# Xem vring statistics (nếu kernel có CONFIG_VIRTIO_DEBUG)
cat /sys/kernel/debug/virtio/virtio0/vqs/0/stats
```

### 14.4 STR Debug

```bash
# === Pre-suspend check ===
adb shell cat /sys/power/state
# Expected: freeze mem (mem = deep = STR)

adb shell cat /sys/power/mem_sleep
# Expected: [s2idle] deep (deep highlighted = default)

# Xem drivers nào có thể wakeup
adb shell cat /sys/kernel/debug/wakeup_sources | head -20

# Test suspend với timeout 30s
adb shell "echo +30 > /sys/class/rtc/rtc0/wakealarm"
adb shell "echo mem > /sys/power/state"  # System sẽ sleep rồi tự wake

# Nếu bị blocked:
adb shell "echo N > /sys/power/pm_async"     # Disable async suspend
adb shell "echo 1 > /sys/power/pm_debug_messages"  # Enable PM debug
adb shell dmesg | grep -E "PM:|suspend|resume|dpm_"

# Tìm driver gây lỗi:
# Error message thường có dạng: "PM: Failed to suspend device X"
adb shell dmesg | grep "PM: Failed"
# Nếu là USB:
adb shell dmesg | grep "xhci\|usb.*suspend"
# Fix: disable auto-suspend cho USB device problematic
echo on > /sys/bus/usb/devices/<device>/power/control

# Check STR statistics
adb shell cat /sys/kernel/debug/suspend_stats
# Expected output:
# success: 5         ← số lần suspend thành công
# fail: 1            ← số lần fail
# failed_freeze: 0
# failed_prepare: 0
# failed_suspend: 1  ← fail ở phase suspend
# last_failed_dev: xhci_hcd.0  ← driver gây lỗi
# last_failed_errno: -110       ← ETIMEDOUT
```

---

## 15. Bảng Tra Cứu: Log → Root Cause

### 15.1 Boot Failures

| Log Message | Ý Nghĩa | Nguyên Nhân | Giải Pháp |
|-------------|---------|-------------|-----------|
| `[COM4 silent]` | Boot ROM không chạy | Boot pins sai / power issue | Kiểm tra SW7, power supply |
| `DDR: Training FAIL` | LPDDR5 PHY training failed | DDR PHY config sai / hardware | Check OEI DDR_CONFIG, verify hardware |
| `DDRMIX is powered OFF` | OEI không chạy | Flash.bin không có OEI | Rebuild flash.bin với OEI |
| `if MU not probed, hang` | SCMI MU init failed | SM chưa ready | SM chưa release MU / SM crash |
| `NOTICE: BL31` sau đó im lặng | ATF crash | ATF không có UART clock | Check SCMI clock init trong ATF |
| `Kernel panic VFS` | rootfs mount fail | Android partition lỗi | Flash lại Android image |

### 15.2 Audio Failures (AOSP specific)

| Log Message | Ý Nghĩa | Root Cause | Fix |
|-------------|---------|-----------|-----|
| `fmqByteCount=0` trong AudioFlinger | Không có data tới HAL | AudioPolicyManager không route | Check `ro.android.car.audio.enableaudiopatch=true` |
| `setPortGain() not called` | Gain = 0 | `enableaudiopatch=false` | Set `enableaudiopatch=true` trong device.mk |
| `SAI: underrun` | FIFO empty | Clock rate mismatch | Verify SCMI clock, SAI TCR2 register |
| `tinyplay: cannot open card2` | PCM device missing | TAS5828 driver fail | Check I2C address, dmesg grep tas5828 |
| `ASRC ratio error` | Sample rate conversion fail | policy_configuration.xml sai | Fix inputSampleRate/outputSampleRate |

### 15.3 RPMsg Failures

| Log Message | Ý Nghĩa | Root Cause | Fix |
|-------------|---------|-----------|-----|
| `imx-rproc: SCMI LMM boot failed` | M7 start fail | SM không start LM2 | Check SM config, verify lmm-id trong DTS |
| `virtio0: timeout waiting for M7` | RPMsg link timeout | M7 firmware không init RPMsg | Check M7 firmware, verify shared mem address |
| `RL_ERR_NO_BUFF` (M7 side) | Buffer exhausted | vring size quá nhỏ | Tăng vdev0buffer size trong DTS |
| `/dev/rpmsg0 not found` | RPMsg device không tạo | M7 không gọi rpmsg_ns_announce | Kiểm tra M7 firmware code |

### 15.4 STR Failures

| Log Message | Ý Nghĩa | Root Cause | Fix |
|-------------|---------|-----------|-----|
| `PM: failed to suspend: -110` | Timeout | Driver không respond | Identify driver từ `last_failed_dev` |
| `xhci: CMD_RUN timeout` | USB controller timeout | USB device attached | Disable USB wakeup hoặc fix DTS power |
| `DDR: exit retention failed` | DDR không recover | DDR PHY state corrupt | Check ATF dram_exit_retention() |
| `SCMI: clock error after resume` | Clock out of sync | SM reset clocks | Re-request clocks trong driver .resume() |
| `[dead after STR]` | System không wake | Wakeup source chưa set | Enable RTC/GPIO wakeup, check GPC config |

---

## Appendix: Quick Reference

### A. UART Connection

```
EVK J31 (USB-C) → PC
  COM1 (ttyUSB0): Cortex-M7 FreeRTOS debug
  COM2 (ttyUSB1): Không dùng thường
  COM3 (ttyUSB2): Cortex-A55 U-Boot + Linux + Android
  COM4 (ttyUSB3): Cortex-M33 System Manager
Baudrate: 115200 8N1
```

### B. Key Source Repos

```
imx-sm:    https://github.com/nxp-imx/imx-sm      (System Manager M33)
imx-oei:   https://github.com/nxp-imx/imx-oei     (DDR init M33)
imx-atf:   https://github.com/nxp-imx/imx-atf     (ATF BL31 A55 EL3)
uboot-imx: https://github.com/nxp-imx/uboot-imx   (U-Boot)
linux-imx: https://github.com/nxp-imx/linux-imx   (Linux kernel)
rpmsg-lite:https://github.com/nxp-mcuxpresso/rpmsg-lite (M7 RPMsg)
```

### C. Key Files

```
Boot:
  arch/arm/mach-imx/imx9/scmi/imximage.cfg         ← Container structure
  board/freescale/imx95_evk/spl.c                  ← SPL init sequence
  plat/imx/imx95/imx95_bl31_setup.c               ← ATF init

SCMI:
  drivers/firmware/arm_scmi/vendors/imx/imx-sm-lmm.c  ← LMM protocol
  drivers/firmware/arm_scmi/clock.c                    ← Clock protocol
  sm/src/main.c                                        ← SM main loop

RPMsg:
  drivers/remoteproc/imx_rproc.c                  ← Remoteproc driver
  middleware/multicore/rpmsg-lite/lib/rpmsg_lite.c ← M7 side RPMsg

STR:
  plat/imx/imx95/imx95_psci.c                     ← PSCI suspend/resume
  kernel/power/suspend.c                           ← Linux PM core
  sound/soc/fsl/fsl_sai.c                         ← SAI suspend/resume

Audio:
  frameworks/av/services/audioflinger/Threads.cpp  ← AudioFlinger
  hardware/interfaces/audio/ (HAL AIDL)            ← Audio HAL
  sound/soc/fsl/fsl_sai.c                         ← SAI register driver
```
