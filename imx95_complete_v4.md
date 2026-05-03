# NXP i.MX95 EVK — Tài Liệu Chuyên Sâu Toàn Diện
## Boot Sequence · Inter-Core Communication · STR · Audio · Porting · Expert Topics

> **Mục tiêu:** Tài liệu duy nhất từ cơ bản đến expert cho Android Automotive Audio PIC làm việc trên i.MX95.  
> **Phiên bản:** 4.0 — Unified Edition  
> **Hardware target:** NXP i.MX95 19x19 EVK (MIMX9596), Android 15 AOSP  

---

## Mục Lục

**Phần I — Kiến Trúc Nền Tảng**
1. [SoC Architecture — Core Topology & Memory Map](#1-soc-architecture)
2. [Power Architecture — GPC, Power Domains, PMIC](#2-power-architecture)
3. [Clock Tree — Từ OSC đến SAI MCLK](#3-clock-tree)
4. [TRDC — Resource Isolation giữa các Core](#4-trdc--resource-isolation)

**Phần II — Boot Sequence Chi Tiết**
5. [Cấu Trúc flash.bin — AHAB Container Format](#5-cấu-trúc-flashbin)
6. [Giai đoạn 0 — Boot ROM (M33)](#6-giai-đoạn-0--boot-rom)
7. [Giai đoạn 1 — OEI: DDR Training (M33)](#7-giai-đoạn-1--oei-ddr-training)
8. [Giai đoạn 2 — System Manager: SM (M33)](#8-giai-đoạn-2--system-manager)
9. [Giai đoạn 3 — U-Boot SPL (A55)](#9-giai-đoạn-3--u-boot-spl)
10. [Giai đoạn 4 — ATF BL31 (A55 EL3)](#10-giai-đoạn-4--atf-bl31)
11. [Giai đoạn 5 — U-Boot Proper (A55)](#11-giai-đoạn-5--u-boot-proper)
12. [Giai đoạn 6 — Linux Kernel Boot (A55)](#12-giai-đoạn-6--linux-kernel-boot)
13. [Giai đoạn 7 — Android Init → App](#13-giai-đoạn-7--android-init--app)

**Phần III — Inter-Core Communication**
14. [MU — Messaging Unit: Hardware Detail](#14-mu--messaging-unit)
15. [SCMI — System Manager ↔ A55: Packet Level](#15-scmi--sm--a55-packet-level)
16. [SCMI LMM Protocol — Source Code Thực Tế](#16-scmi-lmm-protocol)
17. [RPMsg / Virtio — M7 ↔ A55: Deep Dive](#17-rpmsg--virtio--m7--a55)

**Phần IV — Android Audio Stack**
18. [Audio Clock Chain — PLL → CCM → SAI → Codec](#18-audio-clock-chain)
19. [DMA Architecture — eDMA3 Audio Path](#19-dma-architecture)
20. [ASoC DAPM — Widget Graph & Power Sequencing](#20-asoc-dapm)
21. [Android Audio Stack — App đến SAI Register](#21-android-audio-stack)

**Phần V — Suspend to RAM (STR)**
22. [STR Overview — Power States & Modes](#22-str-overview)
23. [STR Suspend Path — Source Code End-to-End](#23-str-suspend-path)
24. [STR Resume Path — Source Code](#24-str-resume-path)
25. [M7 LPA — Low Power Audio khi A55 Sleep](#25-m7-lpa--low-power-audio)

**Phần VI — Expert Topics**
26. [Porting Guide — Custom Board từ EVK95](#26-porting-guide)
27. [Security — AHAB, ELE, AVB, dm-verity](#27-security)
28. [Performance Tuning — Latency, DVFS, CPU Isolation](#28-performance-tuning)
29. [Crash Analysis — Panic, ELE Events, JTAG](#29-crash-analysis)

**Phần VII — Debug Cookbook**
30. [Debug Recipes — Lệnh Thực Tế Theo Từng Layer](#30-debug-recipes)
31. [Log Reference — Annotated Boot Timeline](#31-log-reference)
32. [Bảng Tra Cứu: Lỗi → Root Cause → Fix](#32-bảng-tra-cứu)

---

## PHẦN I — KIẾN TRÚC NỀN TẢNG

## 1. SoC Architecture

### 1.1 Core Topology

```
┌──────────────────────────────────────────────────────────────────────────┐
│                         NXP i.MX95 (MIMX9596)                            │
│                                                                           │
│  ┌───────────── Application Domain (APD) ─────────────────────────────┐ │
│  │  Cortex-A55 × 6  (AArch64)  @1.8 GHz max                          │ │
│  │  GIC-700 base: 0x48000000   L3 cache: 1MB unified                  │ │
│  │  UART1 (A55 debug): 0x44380000  → COM3 trên EVK (ttyUSB2)         │ │
│  └────────────────────────────────────────────────────────────────────┘ │
│                                                                           │
│  ┌───────── Real-Time Domain (RTD) ──────────────────────────────────┐  │
│  │  ┌──────────────────────────┐  ┌──────────────────────────────┐  │  │
│  │  │  Cortex-M33  @ 333 MHz   │  │  Cortex-M7  @ 800 MHz        │  │  │
│  │  │  [Boot Core]             │  │  [Real-time co-processor]    │  │  │
│  │  │  [System Manager]        │  │  ITCM: 0x20480000 (256KB)    │  │  │
│  │  │  ITCM: 0x1FFE0000 (256KB)│  │  DTCM: 0x20500000 (256KB)   │  │  │
│  │  │  DTCM: 0x20000000 (256KB)│  │  UART1 (M7): 0x44380000     │  │  │
│  │  │  UART3 (M33): 0x44570000 │  │  → COM1 trên EVK (ttyUSB0)  │  │  │
│  │  │  → COM4 trên EVK (ttyUSB3)│  └──────────────────────────────┘  │  │
│  │  └──────────────────────────┘                                     │  │
│  └───────────────────────────────────────────────────────────────────┘  │
│                                                                           │
│  ┌─── ELE (EdgeLock Enclave) ──┐  ┌─── Interconnect / NOC ───────────┐ │
│  │  Arm SC300 (secure island)  │  │  MU0 (SM↔A55): 0x44230000       │ │
│  │  Base: 0x47520000 (ELE MU)  │  │  MU1 (SM↔M7):  0x44240000       │ │
│  │  Role: AHAB auth, crypto    │  │  MU2 (M7↔A55 RPMsg): 0x42430000 │ │
│  └─────────────────────────────┘  │  TRDC base: 0x44270000           │ │
│                                    └──────────────────────────────────┘ │
│  ┌─── LPDDR5 (up to 8GB) ──────┐  ┌─── Audio Subsystem ──────────────┐ │
│  │  Controller: DDRC            │  │  SAI1-8 (I2S/TDM)               │ │
│  │  PHY: 0x4E300000             │  │  SAI3: 0x42650000 (TAS5828)     │ │
│  │  DDRC: 0x4E300000            │  │  MICFIL: 0x44520000 (PDM mic)   │ │
│  └──────────────────────────────┘  │  ASRC: 0x42C00000               │ │
│                                    └──────────────────────────────────┘ │
└──────────────────────────────────────────────────────────────────────────┘
```

### 1.2 Core Roles & UART Mapping

| Core | Freq | Vai trò | UART | EVK Port |
|------|------|---------|------|----------|
| **Cortex-M33** | 333 MHz | Boot core → System Manager. Chạy **suốt vòng đời** hệ thống, xử lý SCMI requests | UART3 @ 0x44570000 | COM4 (ttyUSB3) |
| **Cortex-M7** | 800 MHz | Real-time co-processor. FreeRTOS/Zephyr. Được SM khởi động theo yêu cầu | UART1 @ 0x44380000 | COM1 (ttyUSB0) |
| **Cortex-A55 × 6** | 1.8 GHz | Application cores. Linux + Android AOSP | UART1 @ 0x44380000 | COM3 (ttyUSB2) |
| **ELE** | internal | Security enclave. Không lập trình được trực tiếp | N/A | N/A |

### 1.3 Memory Map Quan Trọng

```
Physical Address    Size      Nội dung
──────────────────────────────────────────────────────────────
0x0000_0000         varies    Boot ROM (M33, read-only)
0x1FFE_0000         256 KB    M33 ITCM  ← SM code chạy tại đây
0x2000_0000         256 KB    M33 DTCM
0x2040_0000         varies    M33 OCRAM (SM heap/stack)
0x2048_0000         256 KB    M7 ITCM  (global view)
0x2050_0000         256 KB    M7 DTCM  (global view)
0x4461_1000         128 B     SCMI shared mem (M33/M7 side) ← shmem0
0x4463_1000         128 B     SCMI shared mem (A55 side)    ← shmem1
0x8000_0000         up 8 GB   LPDDR5 (main system RAM)
0x8A20_0000         ~1 MB     ATF BL31 (OCRAM mapped)
0x9020_0000         ~4 MB     U-Boot proper
0x9300_0000         ~4 MB     DTB
0xA800_0000         1 MB      RPMsg shared mem (M7 ↔ A55)
  0xA800_0000       32 KB     vdev0vring0 (A55→M7 TX)
  0xA800_8000       32 KB     vdev0vring1 (M7→A55 TX)
  0xA801_0000       960 KB    Message buffer pool
```

---

## 2. Power Architecture — GPC, Power Domains, PMIC

### 2.1 External Power Rails (PMIC PF9453/PCA9460 trên EVK)

```
PMIC I2C: bus 0, addr 0x32

Power Rail       Voltage   Powers
──────────────────────────────────────────────────────
NVCC_BBSM_1P8   1.8 V     RTC, BBSM GPR  ← MUST be first ON / last OFF
VDD_SOC         0.8–0.9V  Core digital logic, M33, M7, PLLs
VDD_ARM         1.1 V     A55 cores (separate rail → DVFS capable)
VDD_DDR         1.1 V     DDR controller + PHY
VDD_1P8         1.8 V     I/O supplies, audio codec VDD
VDD_3P3         3.3 V     Peripheral I/O

Power-Up Sequence (cold boot):
  1. NVCC_BBSM_1P8 → ON   (must be first)
  2. VDD_SOC       → ON   (0.8V nominal)
  3. VDD_ARM       → ON
  4. VDD_1P8/3P3   → ON
  5. VDD_DDR       → ON
  6. POR_B         → deassert  → M33 Boot ROM starts
```

### 2.2 Internal Power Domains (SM managed)

```
┌────────────────────────────────────────────────────────────────┐
│  AONMIX  (Always-On Mix) — không bao giờ tắt khi powered       │
│    M33, RTC, BBSM, I2C1, BBSM-GPIO, Tamper detection          │
├────────────────────────────────────────────────────────────────┤
│  WAKEUPMIX — tắt trong SUSPEND, bật khi wakeup event          │
│    SAI1-8, LPUART, CAN, SPI, I2C2-8, GPIO, USB PHY            │
├────────────────────────────────────────────────────────────────┤
│  NOCMIX   — Network-on-Chip                                    │
├────────────────────────────────────────────────────────────────┤
│  DDRBLK   — DDR controller + PHY                              │
├────────────────────────────────────────────────────────────────┤
│  DDRMIX   — DDR I/O pads                                      │
├────────────────────────────────────────────────────────────────┤
│  MEDIAMIX — GPU, VPU, ISP, Display                            │
├────────────────────────────────────────────────────────────────┤
│  M7MIX    — Cortex-M7 cluster (OFF khi M7 không chạy)        │
├────────────────────────────────────────────────────────────────┤
│  NPUMIX   — NPU (eIQ Neutron)                                 │
└────────────────────────────────────────────────────────────────┘
```

### 2.3 GPC — General Power Controller

GPC (base: `0x44470000`) điều phối power state transitions. ATF viết GPC registers trong suspend/resume.

```c
/* GPC Key Registers */
#define GPC_BASE    0x44470000
#define GPC_SLPCR   (GPC_BASE + 0x014)  /* System Low Power Control */
/*  GPC_SLPCR bits:
    [2]  VSTBY   — assert PMIC_STBY_REQ khi vào system suspend
    [16] A55_PDN — power down A55 cluster on WFI
    [17] A55_PGC — enable power gate control cho A55
*/

/* Ví dụ: Configure GPC cho system suspend (trong ATF) */
void imx_set_sys_lpm(bool retention) {
    uint32_t val = mmio_read_32(GPC_SLPCR);
    val |= (1 << 16) |  /* A55_PDN: power down A55 cluster */
           (1 << 17) |  /* A55_PGC: enable power gate */
           (1 << 2);    /* VSTBY: tell PMIC to lower voltage */
    mmio_write_32(GPC_SLPCR, val);
}

/* Sau khi A55 thực thi WFI:
   Hardware GPC tự động:
   1. Clock gate A55 cluster
   2. Power gate A55 CPUs (L1/L2 context lost, L3 retained tùy config)
   3. Assert PMIC_STBY_REQ → PMIC hạ VDD_SOC
   4. System enters deepest sleep
*/
```

### 2.4 Power Domain trong DTS

```dts
/* Khai báo power domain dependency */
&sai3 {
    /* SAI3 thuộc WAKEUPMIX → SM tự power on khi SAI3 probe */
    power-domains = <&scmi_power IMX95_PD_AUDIO>;
};

/* Kiểm tra power domain state từ Linux */
/* adb shell cat /sys/kernel/debug/pm_genpd/pm_genpd_summary
   domain              status  slaves
   imx95-pd-audio       on     sai3, sai1
   imx95-pd-m7          off    ← M7 chưa start
   imx95-pd-ddr         on     ddrc
*/
```

---

## 3. Clock Tree — Từ OSC đến SAI MCLK

### 3.1 Clock Hierarchy Tổng Quan

```
24 MHz XTAL Oscillator (OSC24M)
        │
        ├──────────────────────────────────────────────────┐
        │                                                  │
        ▼                                                  ▼
┌───────────────────────┐                    ┌─────────────────────────┐
│  Audio PLL (Frac PLL) │                    │  ARM PLL / SYS PLLs    │
│  AUDIO_PLL1 / PLL2    │                    │  A55 @ up to 1.8 GHz   │
│                       │                    └─────────────────────────┘
│  Formula:             │
│  Fvco = Fref ×        │
│    (MFI + MFN/MFD)    │
│                       │
│  48kHz family:        │
│    MFI=32, MFN=0,MFD=1│
│    Fvco = 786.432 MHz │
│    ÷32  = 24.576 MHz  │
│    ÷64  = 12.288 MHz ←│── SAI MCLK cho 48kHz (256fs)
│                       │
│  44.1kHz family:      │
│    Fvco ≈ 722.534 MHz │
│    ÷64  = 11.290 MHz ←│── SAI MCLK cho 44.1kHz
└──────────┬────────────┘
           │
           ▼
┌──────────────────────────────────────────────────────┐
│  CCM — Clock Control Module (base: 0x44450000)       │
│  123 Clock Roots                                     │
│                                                      │
│  SAI3_CLK_ROOT (index 0x6C):                        │
│    CCM reg addr = 0x44450000 + 0x6C × 0x80          │
│                 = 0x44456800                         │
│    Bits [25:24] MUX    = 01 → AUDIO_PLL1_OUT        │
│    Bits [15:8]  PRE_DIV = n (÷n+1)                  │
│    Bits [5:0]   POST_DIV = m (÷m+1)                 │
│    → SAI3 clock = 786.432 / PRE / POST MHz          │
│                                                      │
│  LPCG (Low Power Clock Gate):                       │
│    SAI3_LPCG = 0x44456000 + offset                  │
│    Bit 0: 1 = enable clock to SAI3 peripheral       │
└──────────┬───────────────────────────────────────────┘
           │ SAI3_CLK (12.288 MHz example)
           ▼
┌──────────────────────────────────────────────────────┐
│  SAI3 Controller (0x42650000)                        │
│                                                      │
│  MCLK  = 12.288 MHz  (input từ CCM)                 │
│  BCLK  = MCLK / DIV  (TCR2[4:0] = DIV)             │
│         Công thức: BCLK = fs × 2 × channels × bits  │
│         Ví dụ stereo 16-bit 48kHz:                  │
│           BCLK = 48000 × 2 × 2 × 16 = 3.072 MHz    │
│           DIV  = 12.288 / 3.072 = 4 → TCR2 = 0x3   │
│  LRCK  = BCLK / (2 × frame_size) = 48000 Hz ✓      │
└──────────┬───────────────────────────────────────────┘
           │ I2S (MCLK + BCLK + LRCK + DATA)
           ▼
    TAS5828 / Codec (I2S Slave)
```

### 3.2 DTS Clock Configuration Đúng Cách

```dts
/* File: arch/arm64/boot/dts/freescale/imx95-19x19-evk.dts */

&sai3 {
    pinctrl-names = "default";
    pinctrl-0 = <&pinctrl_sai3>;

    /* Clock IDs (từ include/dt-bindings/clock/imx95-clock.h) */
    clocks = <&scmi_clk IMX95_CLK_SAI3>,        /* bus/IPG clock */
             <&scmi_clk IMX95_CLK_DUMMY>,        /* mclk0 unused */
             <&scmi_clk IMX95_CLK_SAI3_MCLK1>,  /* mclk1 = MCLK output */
             <&scmi_clk IMX95_CLK_DUMMY>,
             <&scmi_clk IMX95_CLK_DUMMY>;
    clock-names = "bus", "mclk0", "mclk1", "mclk2", "mclk3";

    /* SCMI request: set SAI3 clock root to 12.288 MHz
       Formula: 12288000 = 48000 × 256  (256fs MCLK) */
    assigned-clocks      = <&scmi_clk IMX95_CLK_SAI3>;
    assigned-clock-rates = <12288000>;

    /* SAI3 là master: tạo MCLK, BCLK, LRCK ra ngoài */
    fsl,sai-mclk-direction-output;
    status = "okay";
};

/* IMPORTANT: Để hỗ trợ cả 48kHz VÀ 44.1kHz, cần 2 MCLK sources */
/* 44.1kHz: mclk2 = 11289600 Hz = 44100 × 256 từ AUDIO_PLL2 */
/* SAI driver tự chọn mclk phù hợp trong fsl_sai_set_bclk() */
```

### 3.3 Clock IDs Quan Trọng (imx95-clock.h)

```c
/* include/dt-bindings/clock/imx95-clock.h */
#define IMX95_CLK_SAI1         64
#define IMX95_CLK_SAI2         65
#define IMX95_CLK_SAI3         66   /* ← TAS5828 speaker */
#define IMX95_CLK_SAI4         67
#define IMX95_CLK_SAI5         68
#define IMX95_CLK_SAI3_MCLK1  120  /* ← MCLK output của SAI3 */
#define IMX95_CLK_AUDIO_PLL1  128  /* ← Audio PLL1 VCO */
#define IMX95_CLK_AUDIO_PLL2  129  /* ← Audio PLL2 VCO (44.1kHz) */
#define IMX95_CLK_PDM         142  /* ← MICFIL PDM clock */

/* include/dt-bindings/power/imx95-power.h */
#define IMX95_PD_AUDIO   4   /* Audio power domain */
#define IMX95_PD_DDR     7   /* DDR power domain */
#define IMX95_PD_M7      11  /* M7 power domain */
```

### 3.4 Clock Debug

```bash
# === Kiểm tra Audio PLL qua SM monitor (COM4) ===
> clock 128        # Audio PLL1 VCO
# Expected: 786432000

> clock 66         # SAI3_CLK_ROOT
# Expected: 12288000

# === Kiểm tra từ Linux ===
adb shell cat /sys/kernel/debug/clk/clk_summary | grep -E "sai3|audio_pll"
# Expected:
#  sai3_root    1  1  12288000  0  50000  0
#  sai3_mclk1   1  1  12288000  0  50000  0

# === Decode CCM register trực tiếp ===
adb shell devmem 0x44456800 32
# Ví dụ output: 0x01000001
# bits [25:24] = 01 → AUDIO_PLL1  ✓
# bits [7:0]   = 01 → POST_DIV=1 (÷2) → 786.432/2/2 = ~196MHz?
# Xem RM Chapter 11 để decode đầy đủ

# === Lỗi phổ biến ===
# "failed to get required clock rate 3072000Hz"
# → SAI driver tìm MCLK nhưng không phù hợp
# Fix: assigned-clock-rates phải = fs × mclk-fs
#   48000 × 256 = 12288000 ✓
#   48000 × 512 = 24576000 (nếu mclk-fs=512 trong sound card DTS)

# "SAI: no valid master clock"
# → clock-names thứ tự sai hoặc AUDIO_PLL rate không chính xác
# Fix: Đảm bảo 12288000 hoặc 24576000 là multiple của 786432000
```

---

## 4. TRDC — Resource Isolation giữa các Core

### 4.1 TRDC Overview

TRDC (Trusted Resource Domain Controller, base: `0x44270000`) là hardware IP ngăn các core truy cập tài nguyên không được phép. Vi phạm TRDC gây bus fault / Oops trên A55.

```
Domain IDs (DID):
  DID 0: M33 System Manager  (highest trust, full access)
  DID 1: ELE (EdgeLock Enclave)
  DID 2: M7 core
  DID 3: A55 AP domain (Linux/Android)
  DID 4: Non-secure DMA masters (eDMA, SDMA)
  DID 5: USB, PCIe

TRDC controllers:
  TRDC-A (TRDC1): Memory region access control
  TRDC-C (TRDC3): Peripheral access control
```

### 4.2 Memory Partitioning (SM configures on boot)

```
Memory Region           Address Range         Allowed DIDs
──────────────────────────────────────────────────────────────
M33 ITCM (SM code)      0x1FFE0000-0x201FFFFF DID0 only
M7 ITCM/DTCM            0x20480000-0x2057FFFF DID0, DID2
SCMI shmem (SM side)    0x44611000-0x4461107F DID0 only
SCMI shmem (A55 side)   0x44631000-0x4463107F DID0, DID3
RPMsg shared mem        0xA8000000-0xA8100000 DID2, DID3
DDR (Linux/Android)     0x80000000-0xFFFFFFFF DID3, DID4
ELE secure region       (hardware protected)  DID1 only
```

### 4.3 SM Configuration (mx95evk.cfg excerpt)

```ini
# File: imx-sm/configs/mx95evk.cfg
# Phần TRDC configuration — SM apply khi boot

[TRDC]
# Allow A55 (DID3) full DDR access
MRC = DDR_ALL,       DID3, RWNS

# RPMsg shared memory: M7 và A55 đều truy cập được
MRC = RPMSG_SHMEM,   DID2, RWNS
MRC = RPMSG_SHMEM,   DID3, RWNS

# M7 TCM: chỉ SM và M7
MRC = M7_TCM,        DID0, RWNS
MRC = M7_TCM,        DID2, RWNS

# Peripheral ownership
PRC = SAI3,          DID3, RWNS   # A55 Linux owns SAI3
PRC = MU0,           DID0, RWNS   # SM owns MU0 (SM↔A55)
PRC = MU0,           DID3, RWNS   # A55 can R/W MU0
PRC = MU2,           DID2, RWNS   # M7 owns MU2
PRC = MU2,           DID3, RWNS   # A55 can access MU2 (RPMsg)
```

### 4.4 TRDC Debug

```bash
# Bus fault do TRDC violation:
# [  12.345] Unhandled fault: synchronous external abort
# [  12.345] ESR = 0x96000010
# [  12.345] FAR = 0x1FFE0080   ← đây là M33 TCM → TRDC violation!

# Check từ SM monitor:
> trdc violation     # Đọc TRDC violation capture registers
# Output:
# TRDC_DERR: addr=0x1FFE0080, master=DID3(A55), type=READ
# → A55 đọc M33 TCM trái phép

# Đọc TRDC violation register trực tiếp:
adb shell devmem 0x44270E00 32   # TRDC violation addr
adb shell devmem 0x44270E04 32   # TRDC violation info (DID, type)
```

---

## PHẦN II — BOOT SEQUENCE CHI TIẾT

## 5. Cấu Trúc flash.bin

### 5.1 AHAB Container Format

```
flash.bin layout trên SD card / eMMC:
Offset 0x8000 (32KB):
┌─────────────────────────────────────────────────────────────────┐
│  PRIMARY CONTAINER (Container 1) — Signed bởi NXP + OEM        │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │ AHAB Header: magic=0x87, version=2                      │    │
│  │ Image Array:                                            │    │
│  │  [0] ELE Firmware (mx95b0-ahab-container.img)          │    │
│  │      core=ELE, type=double_auth                        │    │
│  │  [1] OEI-DDR (oei-m33-ddr.bin)                        │    │
│  │      core=M33, type=OEI                                │    │
│  │      load=0x1FFC0000, entry=0x1FFC0001 (thumb)        │    │
│  │  [2] OEI-TCM (oei-m33-tcm.bin)                        │    │
│  │      core=M33, type=OEI                                │    │
│  │  [3] System Manager (m33_image.bin)                    │    │
│  │      core=M33, type=executable                         │    │
│  │      load=0x1FFC0000, entry=0x1FFC0001                 │    │
│  │  [4] DDR PHY firmware (lpddr5_*.bin)                   │    │
│  │      Loaded by OEI-DDR, stored in container            │    │
│  └─────────────────────────────────────────────────────────┘    │
├─────────────────────────────────────────────────────────────────┤
│  SECONDARY CONTAINER (Container 2) — Loaded bởi SPL            │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │  [0] ATF BL31 (bl31.bin)                               │    │
│  │      core=A55, load=0x8A200000 (OCRAM mapped)         │    │
│  │  [1] U-Boot proper (u-boot.bin)                        │    │
│  │      core=A55, load=0x90200000 (DDR)                  │    │
│  │  [2] DTB (imx95-19x19-evk.dtb)                        │    │
│  │      load=0x93000000                                   │    │
│  └─────────────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────────────┘
```

### 5.2 imximage.cfg (Build Config)

```ini
/* arch/arm/mach-imx/imx9/scmi/imximage.cfg */
BOOT_FROM SD
SOC_TYPE  IMX9

APPEND mx95a0-ahab-container.img    /* ELE container (NXP signed) */

CONTAINER
IMAGE OEI  m33-oei-ddrfw.bin  0x1ffc0000  /* DDR training OEI */
HOLD 0x10000                              /* Wait for DDR stable */
IMAGE OEI  oei-m33-tcm.bin   0x1ffc0000  /* TCM init OEI */
IMAGE EXEC m33_image.bin      0x1ffc0000  /* System Manager */

CONTAINER
IMAGE A55  bl31.bin           0x8a200000  /* ATF */
IMAGE A55  u-boot.bin         CONFIG_TEXT_BASE  /* U-Boot: 0x90200000 */
```

### 5.3 Build Commands

```bash
# 1. ELE firmware (binary từ NXP — không tự build được)
wget https://www.nxp.com/lgfiles/NMG/MAD/YOCTO/firmware-ele-imx-2.0.2-89161a8.bin
sh firmware-ele-imx-*.bin --auto-accept
cp firmware-ele-imx-*/mx95b0-ahab-container.img $UBOOT_SRC/

# 2. DDR PHY firmware (binary blob, Synopsys)
wget https://www.nxp.com/lgfiles/NMG/MAD/YOCTO/firmware-imx-8.28-994fa14.bin
sh firmware-imx-*.bin --auto-accept
cp firmware-imx-*/firmware/ddr/synopsys/lpddr5*v202409.bin $UBOOT_SRC/

# 3. OEI (DDR init, chạy trên M33)
git clone https://github.com/nxp-imx/imx-oei.git
cd imx-oei
make board=mx95lp5 oei=ddr DEBUG=1 r=B0 \
     DDR_CONFIG=XIMX95LPD5EVK19_6400mbps_train_timing_a1 all
cp build/mx95lp5/ddr/oei-m33-ddr.bin  $UBOOT_SRC/
make board=mx95lp5 oei=tcm DEBUG=1 r=B0 all
cp build/mx95lp5/tcm/oei-m33-tcm.bin  $UBOOT_SRC/

# 4. System Manager
git clone https://github.com/nxp-imx/imx-sm.git
cd imx-sm
make config=mx95evk all
cp build/mx95evk/m33_image.bin $UBOOT_SRC/

# 5. ATF BL31
git clone -b lf_v2.12 https://github.com/nxp-imx/imx-atf.git
cd imx-atf
make PLAT=imx95 bl31
cp build/imx95/release/bl31.bin $UBOOT_SRC/

# 6. U-Boot
cd $UBOOT_SRC
make imx95_19x19_evk_defconfig
make -j$(nproc)

# 7. Package flash.bin
cd imx-mkimage/iMX95/
# Copy tất cả binaries vào đây, rồi:
make SOC=iMX95 dtbs=imx95-19x19-evk.dtb flash_evk
# Output: flash.bin

# 8. Flash lên SD card
sudo dd if=flash.bin of=/dev/sdX bs=1k seek=32 conv=fsync
```

---

## 6. Giai Đoạn 0 — Boot ROM (M33)

### 6.1 Flow

```
Power On → M33 Boot ROM (ROM, không có source)
    │
    ├── Đọc boot mode pins: SW7[1:4] trên EVK
    │     0b1011 → SD Card (USDHC2)
    │     0b1010 → eMMC   (USDHC1)
    │     0b1001 → USB Serial Download (SDP)
    │     0b0000 → JTAG
    │
    ├── Init USDHC (SD/eMMC controller)
    │     Enable clock via CCM
    │     Read SD card at offset 0x8000 (32KB)
    │
    ├── Parse AHAB container header
    │     Verify magic = 0x87 ✓
    │
    ├── Request ELE authenticate container
    │     ELE MU: 0x47520000
    │     Command: ELE_AUTH_IMAGE_REQ
    │     ELE verifies RSA/ECDSA signature
    │
    ├── Load ELE Firmware → ELE core (internal)
    ├── Load OEI-DDR  → M33 TCM 0x1FFC0000
    ├── Execute OEI-DDR (branch to 0x1FFC0001)
    │     [OEI runs, trains DDR, returns]
    ├── Load OEI-TCM  → M33 TCM, execute, return
    ├── Load SM binary → M33 TCM 0x1FFC0000
    └── Branch to SM entry (0x1FFC0001)
        [Boot ROM never returns from here]
```

**Quan trọng:** Boot ROM **hoàn toàn im lặng** — không có output. Dấu hiệu Boot ROM chạy thành công là khi SM bắt đầu in log trên COM4.

### 6.2 Boot Stuck tại đây — Symptoms

```
Triệu chứng: Không có gì trên COM4 sau khi cắm nguồn
Nguyên nhân có thể:
  1. Boot mode pins sai (SW7 config) → đọc sai device
  2. flash.bin không đúng offset → AHAB parse fail
  3. flash.bin ELE container lỗi → authentication fail
  4. PMIC không cấp đủ điện → Boot ROM không chạy được
  5. JTAG mode → cần JTAG debugger

Debug:
  adb shell "nxpuuu" hoặc UUU tool để flash lại
  Kiểm tra SW7 theo sơ đồ EVK User Guide
  Dùng JTAG attach tới M33 core để check state
```

---

## 7. Giai Đoạn 1 — OEI: DDR Training (M33)

### 7.1 OEI Entry Point

```c
/* imx-oei/devices/MIMX9596/startup/startup.c */
void Reset_Handler(void) {
    SystemInitHook();   /* Setup vector table, stack */
    main();             /* OEI main logic */
    /* Return to Boot ROM caller */
}

/* imx-oei/boards/mx95lp5/ddr/ddr_init.c */
int main(void) {
    debug_init();  /* Enable UART output trên COM4 */
    debug_printf("\r\n[OEI] DDR: LPDDR5\r\n");

    /* Step 1: Configure DDR Controller registers */
    ddrc_config(&ddrc_cfg);
    debug_printf("[OEI] DDR: DDRC configured\r\n");

    /* Step 2: Load DDR PHY firmware vào PHY IMEM/DMEM */
    ddr_phy_init(DDR_PHY_BASE, &phy_fw, &phy_cfg);
    debug_printf("[OEI] DDR: PHY firmware loaded\r\n");

    /* Step 3: Run PHY training */
    int ret = ddrc_train(&train_cfg);
    if (ret) {
        debug_printf("[OEI] DDR: Training FAILED! ret=%d\r\n", ret);
        return ret;  /* Boot ROM sẽ nhận lỗi và halt */
    }
    debug_printf("[OEI] DDR: Training PASS\r\n");
    debug_printf("[OEI] DDR: Size=%dMB @ %dMT/s\r\n",
                 get_ddr_size_mb(), 6400);
    return 0;
}
```

### 7.2 DDR PHY Training Steps

```c
/* imx-oei/components/ddr/phy_train.c */
int ddr_phy_init(uintptr_t phy_base, const struct phy_fw *fw,
                 const struct phy_cfg *cfg)
{
    /* 1. Assert PHY reset */
    mmio_write_32(phy_base + DDRPHY_RESET, 0x1);

    /* 2. Load IMEM (microcode) */
    for (int i = 0; i < fw->imem_size / 4; i++)
        mmio_write_32(phy_base + DDRPHY_IMEM_BASE + i*4, fw->imem[i]);

    /* 3. Load DMEM (message block + data) */
    for (int i = 0; i < fw->dmem_size / 4; i++)
        mmio_write_32(phy_base + DDRPHY_DMEM_BASE + i*4, fw->dmem[i]);

    /* 4. Configure training parameters via Message Block */
    struct msg_block *mb = (void*)(phy_base + DDRPHY_DMEM_BASE);
    mb->SequenceCtrl = 0x031F; /* All training steps */
    mb->DramFreq     = 3200;   /* 3200 MHz (6400 MT/s LPDDR5) */
    mb->CsPresent    = 0xF;    /* 4 CS (ranks) */
    mb->HdtCtrl      = 0x5;    /* Debug verbosity */

    /* 5. Release PHY μC, trigger training */
    mmio_write_32(phy_base + DDRPHY_RESET, 0x0);
    mmio_write_32(phy_base + DDRPHY_MICRO_CONT_MUX_SEL, 0x0);

    /* 6. Poll for completion */
    return wait_training_done(phy_base);
}

static int wait_training_done(uintptr_t phy_base) {
    for (int i = 0; i < 500000; i++) {
        uint32_t msg = mmio_read_32(phy_base + DDRPHY_UCT_SHADOW_REGS);
        if (msg == 0x07) return 0;  /* Training complete */
        if (msg == 0xFF) return -1; /* Training failed */
        /* Ack để PHY tiếp tục gửi messages */
        mmio_write_32(phy_base + DDRPHY_UCT_WRITE_PROT, 0x0);
        udelay(1);
    }
    return -ETIMEDOUT;
}
/* Training steps (SequenceCtrl bits):
   bit 0: DevInit        — DDR device initialization
   bit 1: WrLvl          — Write Leveling
   bit 2: RxEnCal        — Receive Enable Calibration
   bit 3: RdDqsCal       — Read DQS Calibration
   bit 4: RdDeskew       — Read Per-bit Deskew
   bit 5: MxRdLat        — Max Read Latency
   bit 8: WrDeskew       — Write Per-bit Deskew
   bit 9: WrEye          — Write Eye Training
*/
```

**OEI Log (COM4):**
```
[OEI] DDR: LPDDR5
[OEI] DDR: DDRC configured
[OEI] DDR: PHY firmware loaded (imem=49152B, dmem=24576B)
[OEI] DDR: Running DevInit...   OK
[OEI] DDR: Running WrLvl...     OK
[OEI] DDR: Running RxEnCal...   OK
[OEI] DDR: Running RdDqsCal...  OK
[OEI] DDR: Running WrEye...     OK
[OEI] DDR: Training PASS
[OEI] DDR: Size=8192MB @ 6400 MT/s
[OEI] OEI complete
```

---

## 8. Giai Đoạn 2 — System Manager (M33)

### 8.1 SM Main Loop

```c
/* imx-sm/sm/src/main.c */
int main(void) {
    /* 1. Platform hardware init */
    BRD_SM_Init();

    /* 2. TRDC — configure resource isolation */
    CONFIG_Load(g_trdc_config, g_trdc_config_len);
    SM_PRINTF("SM: TRDC configured\r\n");

    /* 3. Power domain init */
    PWR_Init();
    SM_PRINTF("SM: Power init complete\r\n");

    /* 4. Clock init (PLLs, clock roots) */
    CLK_Init();
    SM_PRINTF("SM: Clock init complete\r\n");

    /* 5. Create Logical Machines */
    LM_Init();

    /* 6. Boot LM0 (SM itself) */
    LM_Boot(0U);

    /* 7. Boot LM1 (A55 Application Processor) */
    LM_Boot(1U);
    /* → SM sets A55 reset vector = SPL address = 0x20480000 */
    /* → SM deasserts A55 cluster reset                       */
    /* → A55 core 0 starts executing at 0x20480000            */
    SM_PRINTF("SM: LM1 (AP) started, A55 running\r\n");

    /* 8. LM2 (M7) — only boot when remoteproc requests */

    /* 9. SCMI service loop — runs forever */
    SM_PRINTF("SM: Entering SCMI service mode\r\n");
    SM_PRINTF(">\r\n");  /* SM monitor prompt */

    while (true) {
        RPC_SCMI_Process(0U);   /* Handle A55 SCMI requests via MU0 */
        RPC_SCMI_Process(1U);   /* Handle M7 SCMI requests via MU1 */
        SM_EventProcess();      /* Thermal, power events */
        __WFI();                /* Wait for interrupt (power saving) */
    }
}
```

### 8.2 LM_Boot() — Cách SM Release A55

```c
/* imx-sm/sm/src/lmm/sm_lmm.c */
int32_t LM_Boot(uint32_t lmId) {
    lm_t *lm = &g_lm[lmId];

    /* Power on domain */
    PWR_LmPower(lmId, SM_POWER_ON);

    /* Set CPU reset vector */
    /* A55 (LM1): bootAddr = 0x20480000 (SPL in OCRAM) */
    CPU_ResetVectorSet(lm->cpuId, lm->bootAddr);

    /* Deassert reset — CPU starts executing */
    CPU_Reset(lm->cpuId, false);

    lm->state = LM_STATE_ON;
    SM_PRINTF("SM: LM%u '%s' booted @ 0x%08X\r\n",
              lmId, lm->name, lm->bootAddr);
    return SM_ERR_SUCCESS;
}
```

### 8.3 SCMI Server — Clock Request Handler

```c
/* imx-sm/sm/rpc/scmi/rpc_scmi_clock.c */
int32_t RPC_SCMI_ClockRateSet(uint32_t lmId, uint32_t channel,
                               const uint32_t *payload) {
    uint32_t clockId = le32toh(payload[1]);
    uint64_t rate    = ((uint64_t)le32toh(payload[3]) << 32)
                      | le32toh(payload[2]);

    SM_PRINTF("SM: SCMI CLK_RATE_SET: clk=%u rate=%llu\r\n",
              clockId, rate);

    /* Check permission: LM1 (A55) có được set clock này không? */
    if (!LM_ClockIsAccessible(lmId, clockId))
        return SM_ERR_NOT_FOUND;

    /* Thực sự set clock trong CCM */
    int32_t status = CLK_RateSet(clockId, rate);
    /* CLK_RateSet viết CCM_CLOCKROOTn_CONTROL register */

    /* Ghi response vào shared memory */
    uint32_t response = (status == SM_ERR_SUCCESS) ? 0 : SM_ERR_DENIED;
    RPC_SCMI_WriteResponse(channel, &response, sizeof(response));

    /* Notify A55 via MU doorbell */
    MU_TxTrigger(MU0_BASE);
    return SM_ERR_SUCCESS;
}
```

**SM Log (COM4) — Full Annotated:**
```
[OEI] ...training complete...           ← OEI output
SM: v2.4.0 (NXP System Manager i.MX95) ← SM starts
SM: Board: mx95evk
SM: Config: 3 LMs, SCMI v3.2
SM: TRDC: 8 domains configured
SM: Power: 12 domains
SM: Clock: 128 clocks
SM: LM0 (M33): ready
SM: LM1 (AP): booting...
SM: A55 reset vector = 0x20480000
SM: LM1 (AP): started                  ← A55 core 0 running
SM: SCMI service ready on MU0/MU1
>                                       ← SM monitor prompt
```

### 8.4 SM Monitor Commands (COM4)

```bash
# Kết nối: screen /dev/ttyUSB3 115200
> help             # Xem tất cả commands
> lm list          # List Logical Machines
# LM0: M33  [ON]
# LM1: AP   [ON]  boot=0x20480000
# LM2: M7   [OFF]

> clock 66         # Xem SAI3 clock rate
# Clock 66 (SAI3_CLK_ROOT): 12288000 Hz

> sensor 0         # Đọc CPU temperature
# Sensor 0 (A55_TEMP): 42°C

> scmi stat        # SCMI transaction statistics
# CH0 (A55): TX=1247 RX=1247 ERR=0
# CH1 (M7):  TX=0    RX=0    ERR=0

> lm boot 2        # Start M7 LM (nếu binary đã set)
```

---

## 9. Giai Đoạn 3 — U-Boot SPL (A55)

### 9.1 SPL spl.c — Source Code Thực Tế

```c
/* board/freescale/imx95_evk/spl.c
   Source: NXP upstream patch (Ye Li, Alice Guo — NXP 2025) */

void spl_board_init(void) {
    int ret;

    /* CRITICAL: Probe MU TRƯỚC KHI có bất kỳ console output.
     * UART clock phụ thuộc SCMI, SCMI cần MU.
     * Nếu MU fail → hang() (im lặng, không có output)         */
    ret = imx9_probe_mu();
    if (ret)
        hang();

    arch_cpu_init();
    board_early_init_f();

    /* Bây giờ mới có console (SCMI đã sẵn sàng cấp clock UART) */
    preloader_console_init();

    debug("SOC: 0x%x\n", gd->arch.soc_rev);   /* 0x95960010 */
    debug("LC: 0x%x\n", gd->arch.lifecycle);   /* 0x0010 = OEM_OPEN */

    /* Set A55 to max frequency via SCMI */
    clock_init_late();

    /* Verify DDR đã được OEI power up */
    struct udevice *dev;
    u32 state = 0;
    ret = uclass_get_device_by_name(UCLASS_CLK, "protocol@14", &dev);
    ret = scmi_pwd_state_get(dev, IMX95_PD_DDR, &state);
    if (state == BIT(30)) {
        panic("DDRMIX is powered OFF — OEI did not run properly!\n");
    } else {
        printf("DDRMIX: powered UP, DDR ready\n");
        dram_init();
    }
}

void board_init_r(gd_t *dummy1, ulong dummy2) {
    /* Load secondary container (ATF + U-Boot) từ eMMC/SD */
    spl_load_image(spl_boot_device());
    /* jump_to_image_no_args() → bl31_entrypoint */
}
```

**SPL Log (COM3):**
```
U-Boot SPL 2024.04-lf_v2024.04 (May 2025)
DDRMIX: powered UP, DDR ready
SOC: 0x95960010
LC: 0x0010
NOTICE: BL31: v2.12.0(release):lf-6.12.20
NOTICE: BL31: Built : May 2025
```

---

## 10. Giai Đoạn 4 — ATF BL31 (A55 EL3)

### 10.1 BL31 Setup

```c
/* plat/imx/imx95/imx95_bl31_setup.c */

void bl31_early_platform_setup2(u_register_t arg0, u_register_t arg1,
                                 u_register_t arg2, u_register_t arg3) {
    /* Setup UART console */
    console_imx_uart_register(IMX_BOOT_UART_BASE, /* 0x44380000 */
                              IMX_BOOT_UART_CLK_IN_HZ,
                              115200, &imx95_console);

    /* BL33 = U-Boot proper */
    bl33_image_ep_info.pc   = 0x90200000;  /* U-Boot text base */
    bl33_image_ep_info.spsr = get_el2_daif_spsr();
    SET_SECURITY_STATE(bl33_image_ep_info.h.attr, NON_SECURE);
}

void bl31_platform_setup(void) {
    plat_imx_gic_driver_init();   /* Init GIC-700 */
    plat_gic_init();
    imx_setup_power_domains();    /* Register PSCI handlers */
    imx9_scmi_setup_resources();  /* Establish SCMI link with SM */
}

void bl31_main(void) {
    NOTICE("BL31: v%s\n", version_string);
    NOTICE("BL31: Built : %s, %s\n", build_date, build_time);

    /* Register runtime services:
       - PSCI (cpu_on, cpu_off, system_suspend)
       - SCMI SMC proxy (A55 → ATF → MU → SM)
       - TSPD (OP-TEE dispatcher, nếu build với OP-TEE) */
    runtime_svc_init();

    /* Jump to U-Boot */
    bl31_run_next_image(&bl33_image_ep_info);
}
```

### 10.2 PSCI CPU_ON — Bring Up A55 Secondary Cores

```c
/* plat/imx/imx95/imx95_psci.c */

/* Gọi khi Linux muốn bật A55 core N (N = 1..5) */
int imx_pwr_domain_on(u_register_t mpidr) {
    unsigned int cpu = plat_core_pos_by_mpidr(mpidr); /* 1..5 */

    /* Store entry point cho secondary core */
    imx_mailbox[cpu] = (uintptr_t)secondary_entry;

    /* Yêu cầu SM power on + release reset cho core N */
    imx9_scmi_cpu_start(cpu);
    /* SM: CLK_RootEnable(A55_CORE_N_CLK)
       SM: CPU_Reset(cpu, false)  */

    return PSCI_E_SUCCESS;
}
```

**ATF Log (COM3 — xuất hiện trước U-Boot):**
```
NOTICE: BL31: v2.12.0(release):lf-6.12.20-2.0.0
NOTICE: BL31: Built : 10:30:00, May 2025
```

---

## 11. Giai Đoạn 5 — U-Boot Proper (A55)

### 11.1 Init Sequence

```c
/* U-Boot init_sequence_f[] → init_sequence_r[] */

board_init_f():
  ├── setup_mon_len()
  ├── fdtdec_setup()         /* Parse DTB */
  ├── serial_init()          /* UART console via SCMI clock */
  ├── display_options()      /* Print banner */
  └── dram_init()            /* Report DRAM size to gd */
  └── relocate_code()        /* Copy U-Boot → final DDR address */

board_init_r():
  ├── initr_dm()             /* Driver Model init */
  ├── initr_mmc()            /* MMC/SD driver */
  ├── initr_scmi()           /* SCMI channel verify */
  ├── run_main_loop()
  │     └── bootcmd: "run distro_bootcmd"
  │         └── extlinux.conf hoặc Android boot.img
  └── do_bootm_linux()
        ├── fixup_fdt()       /* Patch DTB: RAM map, bootargs */
        └── kernel_entry(0, ~0, fdt_addr)
```

**U-Boot Log (COM3):**
```
U-Boot 2024.04-lf_v2024.04 (May 2025)

CPU:   NXP i.MX95 Rev2.0 A55 at 1800 MHz
DRAM:  7.8 GiB
Core:  305 devices, 26 uclasses, devicetree: separate
MMC:   FSL_SDHC: 0, FSL_SDHC: 1
Loading Environment from MMC... OK
In:    serial@44380000
Out:   serial@44380000
Model: NXP i.MX95 19x19 EVK
Net:   eth0: ethernet@4cc00000
Booting device 0:1 in 2...
```

---

## 12. Giai Đoạn 6 — Linux Kernel Boot (A55)

### 12.1 Entry đến start_kernel()

```c
/* arch/arm64/kernel/head.S: _head → primary_entry() */
primary_entry():
    preserve_boot_args()      /* Save x0=FDT addr */
    el2_setup()               /* Configure EL2 hypervisor regs */
    __create_page_tables()
    __primary_switch()
        → __primary_switched()
            → start_kernel()  /* C code starts here */

/* init/main.c */
start_kernel():
    setup_arch()              /* unflatten_device_tree(), paging_init() */
    mm_init()
    sched_init()
    time_init()               /* ARM arch timer */
    console_init()
    rest_init()
        → kernel_thread(kernel_init)  /* PID 1 */
        → cpu_startup_entry()         /* idle loop */
```

### 12.2 Driver Init (do_one_initcall)

```c
/* Theo thứ tự priority: early → core → postcore → arch → subsys → fs → device */

/* Audio subsystem init: */
[   0.45] imx-clk-scmi: 128 clock domains from SCMI
[   0.52] fsl-sai 42650000.sai: SCMI clock SAI3=12288000Hz
[   0.52] fsl-sai 42650000.sai: probe OK → registered DAI 'sai3'
[   0.53] imx-pcm-dma: registered
[   0.55] tas5828 3-004c: TAS5828 found, configured amplifier
[   0.56] imx95-audio-card: probed → /dev/snd/pcmC2D0p created

/* RPMsg / remoteproc init: */
[   0.60] imx-rproc: probed, M7 firmware: imx/m7-rpmsg.elf
[   0.61] virtio_rpmsg_bus: RPMsg bus registered
```

### 12.3 SMP — Bring Up A55 Cores 1-5

```c
/* arch/arm64/kernel/smp.c */
smp_init()
    → __cpu_up(cpu=1..5)
        → psci_cpu_on(cpu, entry=secondary_startup)
            → SMC PSCI_CPU_ON → ATF → SM SCMI → CPU_Reset(n, false)

/* Log: */
[   0.35] smp: Bringing up secondary CPUs ...
[   0.36] CPU1: Booted secondary processor 0x0000000001
[   0.37] CPU2: Booted secondary processor 0x0000000002
...
[   0.42] smp: Brought up 1 node, 6 CPUs
```

---

## 13. Giai Đoạn 7 — Android Init → App

### 13.1 Android Init Chain

```
/init (First Stage Init)
    ├── Mount /proc, /sys, /dev
    ├── Load kernel modules (insmod *.ko)
    └── exec /system/bin/init (Second Stage)

/system/bin/init (Second Stage)
    ├── Parse /init.rc, /vendor/etc/init/*.rc
    ├── Trigger: early-init → init → late-init
    ├── Start: servicemanager → hwservicemanager → vndservicemanager
    ├── Start: audioserver (AudioFlinger + AudioPolicyService)
    ├── Start: car_service (CarAudioService for AAOS)
    └── Start: zygote → app_process → ZygoteInit.main()
```

### 13.2 CarAudioService — AAOS Specific

```java
// packages/services/Car/service/src/com/android/car/audio/CarAudioService.java
public class CarAudioService extends ICarAudio.Stub {
    // QUAN TRỌNG: enableaudiopatch phải = true
    // ro.android.car.audio.enableaudiopatch=true
    // Nếu false → setPortGain() không được gọi → volume không hoạt động

    private void setupAudioPolicy() {
        if (mCarAudioConfigurationPath != null) {
            mCarAudioZones = loadCarAudioConfiguration();
        }
        // Setup audio zones, routing, focus
    }

    // Gọi AudioPolicyManager.setPortGain() để điều chỉnh volume
    public void setVolumeGroupVolume(int zoneId, int groupId, int index) {
        AudioPolicyManager.setPortGain(portId, gainConfig);
        // → xuống HAL setAudioPortConfig()
        // → tinymix set ALSA control
        // → SAI/TAS5828 volume register
    }
}
```

### 13.3 Annotated Boot Timeline (All UARTs)

```
T+0ms    [COM4] Boot ROM starts (silent)
T+10ms   [COM4] [OEI] DDR: LPDDR5
T+50ms   [COM4] [OEI] DDR: Training PASS
T+55ms   [COM4] SM: v2.4.0 ... TRDC configured
T+56ms   [COM4] SM: LM1 (AP) started, A55 running
T+57ms   [COM3] U-Boot SPL 2024.04
T+58ms   [COM3] NOTICE: BL31: v2.12.0
T+200ms  [COM3] U-Boot 2024.04 ... CPU: i.MX95 1.8GHz
T+3000ms [COM3] Starting kernel...
T+3100ms [COM3] Booting Linux on CPU 0x00
T+3200ms [COM3] Machine model: NXP i.MX95 19x19 EVK
T+3500ms [COM3] SCMI: 128 clock domains
T+4000ms [COM3] fsl-sai 42650000.sai: probe OK
T+8000ms [COM3] init: first stage started!
T+12000ms[COM3] servicemanager: starting
T+15000ms[COM3] audioserver: starting AudioFlinger
T+20000ms[COM3] CarAudioService: init done
T+25000ms[COM3] Zygote: started
T+35000ms[COM3] SystemServer: boot completed
         [ADB]  adb devices → shows device
```


---

## PHẦN III — INTER-CORE COMMUNICATION

## 14. MU — Messaging Unit: Hardware Detail

### 14.1 MU Register Map

```c
/* MU0 (SM↔A55): base 0x44230000 */
/* MU1 (SM↔M7):  base 0x44240000 */
/* MU2 (M7↔A55 RPMsg): base 0x42430000 */

#define MU_VER   0x000  /* Version: 0x02000003 */
#define MU_CR    0x008  /* Control Register */
#define MU_SR    0x00C  /* Status Register
                           bit 0: TX FIFO not full
                           bit 4: RX FIFO not empty (triggers interrupt) */
#define MU_GIER  0x110  /* General Interrupt Enable Register */
#define MU_GCR   0x114  /* General Control Register (DOORBELL)
                           Write bit N → triggers GIP N interrupt on other side */
#define MU_GSR   0x118  /* General Status Register */
#define MU_TCR   0x120  /* TX Control: enable TX interrupt */
#define MU_TSR   0x124  /* TX Status: which TX reg is empty */
#define MU_RCR   0x128  /* RX Control: enable RX interrupt */
#define MU_RSR   0x12C  /* RX Status: which RX reg has data */
#define MU_TR0   0x200  /* TX Register 0 (32-bit) */
#define MU_TR1   0x204  /* TX Register 1 */
#define MU_TR2   0x208  /* TX Register 2 */
#define MU_TR3   0x20C  /* TX Register 3 */
#define MU_RR0   0x280  /* RX Register 0 */
#define MU_RR1   0x284  /* RX Register 1 */
#define MU_RR2   0x288  /* RX Register 2 */
#define MU_RR3   0x28C  /* RX Register 3 */
```

### 14.2 DTS Binding

```dts
/* arch/arm64/boot/dts/freescale/imx95.dtsi */
mu0: mailbox@44230000 {
    compatible = "fsl,imx95-mu";
    reg = <0 0x44230000 0 0x10000>;
    interrupts = <GIC_SPI 234 IRQ_TYPE_LEVEL_HIGH>;
    clocks = <&scmi_clk IMX95_CLK_MU1A>;
    #mbox-cells = <2>;  /* [channel, type] */
};

mu2: mailbox@42430000 {
    compatible = "fsl,imx95-mu";
    reg = <0 0x42430000 0 0x10000>;
    interrupts = <GIC_SPI 241 IRQ_TYPE_LEVEL_HIGH>;
    #mbox-cells = <2>;
};

/* SCMI sử dụng MU0 */
scmi: firmware {
    compatible = "arm,scmi-smc";
    arm,smc-id = <0xc2000002>;
    shmem = <&scmi_shmem_a>;   /* 0x44631000 */
    mboxes = <&mu0 0 2>;       /* MU0, channel 0, type TX */
    mbox-names = "tx";
};

/* RPMsg M7↔A55 sử dụng MU2 */
remoteproc: cm7 {
    compatible = "fsl,imx95-cm7";
    mboxes = <&mu2 0 0>, <&mu2 0 1>;
    mbox-names = "tx", "rx";
};
```

---

## 15. SCMI — SM ↔ A55: Packet Level

### 15.1 SCMI Message Format trong Shared Memory

```c
/* Shared memory layout (ARM SCMI spec DEN0056) */
struct scmi_shared_mem {
    uint32_t reserved;        /* +0x00: reserved */
    uint32_t channel_status;  /* +0x04: bit0=BUSY, bit1=ERROR */
    uint32_t reserved1[2];    /* +0x08-0x0F */
    uint32_t flags;           /* +0x10 */
    uint32_t length;          /* +0x14: tổng bytes từ header+payload */
    uint32_t msg_header;      /* +0x18:
                                  bits [3:0]   = token (sequence ID)
                                  bits [9:8]   = msg_type
                                                 0=Command, 2=Delayed_resp, 3=Notification
                                  bits [15:8]  = message_id
                                  bits [27:16] = protocol_id */
    uint8_t  msg_payload[0];  /* +0x1C: payload bắt đầu */
};

/* Ví dụ SCMI_CLOCK_RATE_SET (set SAI3 clock = 12.288 MHz): */
/* Ghi tại 0x44631000 (A55-side shmem): */
channel_status = 0x00000001; /* Set BUSY */
length         = 0x00000014; /* 20 bytes: 4B header + 16B payload */
msg_header     = 0x00141402; /* proto=0x14(CLOCK), msg=0x05(RATE_SET), token=2 */
/* payload[0..3]  */ = 0x00000042; /* flags: 0=sync */
/* payload[4..7]  */ = 0x00000042; /* clock_id = 0x42 = 66 (SAI3) */
/* payload[8..11] */ = 0x00BB8000; /* rate_low  = 12288000 & 0xFFFFFFFF */
/* payload[12..15]*/ = 0x00000000; /* rate_high = 12288000 >> 32 = 0 */

/* Sau đó: SMC call để notify ATF → ATF → MU0 doorbell → SM */
/* SM reads shmem, sets SAI3 clock, writes response: */
channel_status = 0x00000000; /* Clear BUSY */
length         = 0x00000008; /* 8 bytes: header + 4B response */
/* payload[0..3] */ = 0x00000000; /* status = SUCCESS */
/* SM triggers MU0 interrupt → A55 kernel reads response */
```

### 15.2 Full Round-Trip Trace

```
Linux (A55 EL1): drivers/clk/imx/clk-imx95.c
    imx95_scmi_clk_prepare()
    → ph->clk_ops->enable(ph, clock_id=66)

    drivers/firmware/arm_scmi/clock.c: scmi_clock_enable()
    → Build SCMI message in shmem @ 0x44631000
    → Set channel_status = BUSY
    → SMC: hvc #0 (SMC-ID 0xC2000002, x0=shmem_pa)

[A55 EL3 — ATF BL31]
    plat/imx/imx95/imx95_scmi.c: imx_scmi_smc_handler()
    → Đọc shmem_pa từ x0
    → Write to MU0_GCR bit[0] = 1 (doorbell to M33)

[MU0 hardware]
    → Triggers interrupt on M33

[M33 — System Manager]
    rpc_scmi_clock.c: RPC_SCMI_ClockConfigSet()
    → Parse header: proto=0x14, msg=0x07 (CLOCK_CONFIG_SET)
    → Check LM1 permission: OK
    → CCM write: CCM_CLOCKROOTn @ 0x44456800
    → Write response into shmem: status=SUCCESS
    → Trigger MU0 TX register → A55 interrupt

[A55 — Linux kernel interrupt handler]
    drivers/firmware/arm_scmi/smc.c: scmi_smc_rx_callback()
    → Read response from shmem
    → Complete pending SCMI transaction
    → imx95_scmi_clk_prepare() returns 0 (OK)

Latency: ~50–200 µs một round-trip
```

---

## 16. SCMI LMM Protocol — Source Code Thực Tế

Source code thực tế từ NXP patch (Peng Fan, Oct 2025, upstream U-Boot):

```c
/* drivers/firmware/scmi/vendors/imx/imx-sm-lmm.c */

/* LMM Protocol ID = 0x80 (vendor extension) */
enum scmi_imx_lmm_protocol_cmd {
    SCMI_IMX_LMM_ATTRIBUTES       = 0x3,
    SCMI_IMX_LMM_BOOT             = 0x4,  /* Boot một LM */
    SCMI_IMX_LMM_RESET            = 0x5,
    SCMI_IMX_LMM_SHUTDOWN         = 0x6,  /* Shutdown một LM */
    SCMI_IMX_LMM_WAKE             = 0x7,
    SCMI_IMX_LMM_SUSPEND          = 0x8,
    SCMI_IMX_LMM_NOTIFY           = 0x9,
    SCMI_IMX_LMM_RESET_REASON     = 0xA,
    SCMI_IMX_LMM_POWER_ON         = 0xB,
    SCMI_IMX_LMM_RESET_VECTOR_SET = 0xC,  /* Set entry point */
};

/* Set reset vector của một CPU trong LM (trước khi boot M7) */
int scmi_imx_lmm_reset_vector_set(struct udevice *dev, u32 lmid,
                                   u32 cpuid, u32 flags, u64 vector) {
    struct scmi_imx_lmm_reset_vector_set_in in = {
        .lmid            = lmid,
        .cpuid           = cpuid,
        .flags           = flags,
        .resetvectorlow  = vector & 0xFFFFFFFF,
        .resetvectorhigh = vector >> 32,
    };
    s32 status;
    struct scmi_msg msg = {
        .protocol_id = 0x80,                        /* IMX_LMM */
        .message_id  = SCMI_IMX_LMM_RESET_VECTOR_SET,
        .in_msg      = (u8 *)&in,
        .in_msg_sz   = sizeof(in),
        .out_msg     = (u8 *)&status,
        .out_msg_sz  = sizeof(status),
    };
    int ret = devm_scmi_process_msg(dev, &msg);
    if (ret) return ret;
    return scmi_to_linux_errno(le32_to_cpu(status));
}

/* Boot một LM (start M7) */
int scmi_imx_lmm_power_boot(struct udevice *dev, u32 lmid, bool boot) {
    s32 status;
    struct scmi_msg msg = {
        .protocol_id = 0x80,
        .message_id  = boot ? SCMI_IMX_LMM_BOOT : SCMI_IMX_LMM_POWER_ON,
        .in_msg      = (u8 *)&lmid,
        .in_msg_sz   = sizeof(lmid),
        .out_msg     = (u8 *)&status,
        .out_msg_sz  = sizeof(status),
    };
    int ret = devm_scmi_process_msg(dev, &msg);
    if (ret) return ret;
    return scmi_to_linux_errno(le32_to_cpu(status));
}

/* Shutdown LM */
int scmi_imx_lmm_shutdown(struct udevice *dev, u32 lmid, bool graceful) {
    struct { u32 lmid; u32 flags; } in = {
        .lmid  = lmid,
        .flags = graceful ? BIT(0) : 0,
    };
    s32 status;
    struct scmi_msg msg = {
        .protocol_id = 0x80,
        .message_id  = SCMI_IMX_LMM_SHUTDOWN,
        .in_msg = (u8*)&in, .in_msg_sz = sizeof(in),
        .out_msg = (u8*)&status, .out_msg_sz = sizeof(status),
    };
    int ret = devm_scmi_process_msg(dev, &msg);
    if (ret) return ret;
    return scmi_to_linux_errno(le32_to_cpu(status));
}
```

### 16.2 remoteproc Driver — Dùng LMM để Start M7

```c
/* drivers/remoteproc/imx_rproc.c */
static int imx95_rproc_start(struct rproc *rproc) {
    struct imx_rproc *priv = rproc->priv;

    /* 1. Set M7 entry point qua SCMI LMM */
    int ret = scmi_imx_lmm_reset_vector_set(
        priv->scmi_dev,
        priv->lmm_id,    /* DTS: fsl,lmm-id = <2> */
        priv->cpu_id,    /* DTS: fsl,cpu-id = <3> */
        0,
        rproc->bootaddr  /* M7 firmware entry từ ELF */
    );
    if (ret) return ret;

    /* 2. Boot M7 LM */
    return scmi_imx_lmm_power_boot(priv->scmi_dev, priv->lmm_id, true);
}

/* DTS cho M7 remoteproc */
/*
cm7: remoteproc {
    compatible = "fsl,imx95-cm7";
    fsl,lmm-id = <2>;          ← LM2 trong SM config
    fsl,cpu-id = <3>;          ← CPU ID 3 = M7
    mboxes = <&mu2 0 0>, <&mu2 0 1>;
    mbox-names = "tx", "rx";
    memory-region = <&vdev0vring0>, <&vdev0vring1>,
                    <&vdev0buffer>, <&m7_reserved>;
    fsl,auto-boot;
    fsl,auto-boot-firmware = "imx/m7-rpmsg.elf";
};
*/
```

---

## 17. RPMsg / Virtio — M7 ↔ A55: Deep Dive

### 17.1 Shared Memory Layout

```
0xA8000000 ── vdev0vring0 (A55 TX → M7 RX) ──────────────
│  struct vring:
│    desc[256]:   vring_desc { addr(8B), len(4B), flags(2B), next(2B) }
│                 = 256 × 16 = 4096 bytes
│    avail:       vring_avail { flags(2B), idx(2B), ring[256](2B×256) }
│                 = 516 bytes  → padded to 4096 bytes
│    used:        vring_used { flags(2B), idx(2B), ring[256](8B×256) }
│                 = 2052 bytes → padded to page boundary
0xA8008000 ── vdev0vring1 (M7 TX → A55 RX) ──────────────
│  (same structure)
0xA8010000 ── Message Buffers (pool) ────────────────────
│  Buffer 0:  512 bytes (16B RPMsg header + 496B payload)
│  Buffer 1:  512 bytes
│  ...
│  Buffer N:  512 bytes
│  (RL_BUFFER_COUNT = 256 buffers per direction)
0xA8100000 ────────────────────────────────────────────────
```

### 17.2 RPMsg Protocol — 10-Step Send Flow (A55 → M7)

```c
/* Linux rpmsg_send() internals */

/* Step 1: A55 gets free descriptor from free list */
int desc_idx = virtqueue_get_buf_index(vq);

/* Step 2: Write RPMsg header + data vào buffer */
struct rpmsg_hdr {
    uint32_t src;   /* EPT source address */
    uint32_t dst;   /* EPT destination address = 30 */
    uint32_t reserved;
    uint16_t len;   /* payload length */
    uint16_t flags;
} *hdr = (void*)(buffer_pool_base + desc_idx * 512);
hdr->src = LOCAL_EPT_ADDR;
hdr->dst = 30;
hdr->len = data_len;
memcpy(hdr + 1, data, data_len);

/* Step 3: Fill descriptor */
desc[desc_idx].addr  = buffer_pa;    /* Physical address */
desc[desc_idx].len   = sizeof(*hdr) + data_len;
desc[desc_idx].flags = 0;

/* Step 4: Add to avail ring */
avail->ring[avail->idx % 256] = desc_idx;

/* Step 5: Memory barrier + increment avail idx */
smp_wmb();
avail->idx++;

/* Step 6: Kick M7 via MU2 doorbell */
mbox_send_message(mbox_chan, NULL);
/* → writes MU2_GCR |= BIT(0) */

/* Step 7: MU2 hardware triggers interrupt on M7 */

/* Step 8: M7 interrupt handler */
/* M7 reads avail ring: avail->ring[last_seen] = desc_idx */
/* M7 reads data from buffer[desc_idx] */
/* M7 processes message */

/* Step 9: M7 returns buffer */
/* M7 writes to used ring: used->ring[used->idx].id = desc_idx */
/* M7 increments used->idx */
/* M7 kicks A55 via MU2: MU2_TR0 = desc_idx */

/* Step 10: A55 MU2 interrupt → virtqueue_interrupt() */
/* A55 reads used ring, recycles buffer back to free list */
```

### 17.3 M7 FreeRTOS RPMsg-Lite Init

```c
/* M7 side — FreeRTOS application */
#include "rpmsg_lite.h"
#include "rpmsg_queue.h"
#include "rpmsg_ns.h"

rpmsg_lite_instance_t my_rpmsg_ctxt;
rpmsg_queue_t         my_queue_ctxt;
rpmsg_lite_ept_static_context_t my_ept_ctxt;

void rpmsg_task(void *param) {
    /* Step 1: Init RPMsg-Lite as remote */
    struct rpmsg_lite_instance *rpmsg =
        rpmsg_lite_remote_init(
            (void *)0xA8000000UL,   /* Shared mem base */
            RPMSG_LITE_LINK_ID,
            RL_NO_FLAGS,
            &my_rpmsg_ctxt
        );

    /* Step 2: Đợi A55 master set VIRTIO_CONFIG_S_DRIVER_OK */
    rpmsg_lite_wait_for_link_up(rpmsg, RL_BLOCK);
    /* (polls: vdev_status & VIRTIO_CONFIG_S_DRIVER_OK) */

    /* Step 3: Create receive queue */
    rpmsg_queue_handle_t queue =
        rpmsg_queue_create(rpmsg, &my_queue_ctxt);

    /* Step 4: Create endpoint at address 30 */
    struct rpmsg_lite_endpoint *ept =
        rpmsg_lite_create_ept(rpmsg, 30,
                              rpmsg_queue_rx_cb,
                              queue, &my_ept_ctxt);

    /* Step 5: Announce service to A55 */
    rpmsg_ns_announce(rpmsg, ept,
                      "rpmsg-openamp-demo-channel",
                      RL_NS_CREATE);
    /* → A55 Linux creates /dev/rpmsg0 */

    /* Step 6: Message loop */
    char buf[512];
    uint32_t len, src;
    while (1) {
        rpmsg_queue_recv(rpmsg, queue, &src,
                         buf, sizeof(buf), &len, RL_BLOCK);
        /* Echo back */
        rpmsg_lite_send(rpmsg, ept, src, buf, len, RL_BLOCK);
    }
}
```

**RPMsg Log (dmesg khi M7 start):**
```
[  10.125] imx-rproc: loading firmware imx/m7-rpmsg.elf
[  10.456] imx-rproc: SCMI LMM reset_vector_set: lmid=2 cpu=3 addr=0x20480000
[  10.458] imx-rproc: SCMI LMM boot: lmid=2  OK
[  10.500] virtio_rpmsg_bus virtio0: rpmsg host is online
[  10.600] virtio_rpmsg_bus: creating channel rpmsg-openamp-demo-channel addr=0x1e
[  10.601] rpmsg_char: new /dev/rpmsg0 (src=0x1 dst=0x1e)
[COM1-M7]: RPMsg remote init OK
[COM1-M7]: Link up, EPT 30 ready
```


---

## PHẦN IV — ANDROID AUDIO STACK

## 18. Audio Clock Chain — PLL → CCM → SAI → Codec

(Xem mục 3 — Clock Tree cho đầy đủ PLL config và debug)

### 18.1 fsl_sai_set_bclk() — Chọn MCLK Phù Hợp

```c
/* sound/soc/fsl/fsl_sai.c */
/* Hàm này được gọi khi open audio stream để set BCLK */
static int fsl_sai_set_bclk(struct snd_soc_dai *dai,
                              bool tx, u32 freq) {
    struct fsl_sai *sai = snd_soc_dai_get_drvdata(dai);
    u32 id, best_div = 0, best_rate = 0;
    int best_id = -1;

    /* freq = BCLK cần = sample_rate × 2 × channels × bit_depth
     * Ví dụ: 48000 × 2 × 2 × 32 = 6.144 MHz
     *        48000 × 2 × 2 × 16 = 3.072 MHz            */

    /* Duyệt qua các MCLK sources (mclk0, mclk1, mclk2, mclk3) */
    for (id = 0; id < FSL_SAI_MCLK_MAX; id++) {
        u32 mclk_rate = clk_get_rate(sai->mclk_clk[id]);
        if (mclk_rate == 0) continue;

        /* Tìm divisor phù hợp: BCLK = MCLK / (2 × div) */
        u32 div = mclk_rate / freq / 2;
        u32 actual = mclk_rate / 2 / div;

        if (actual == freq && div > 0) {
            best_id   = id;
            best_div  = div;
            best_rate = mclk_rate;
            break;
        }
    }

    if (best_id < 0) {
        dev_err(&dai->dev,
                "failed to get required clock rate %uHz\n", freq);
        return -EINVAL;
        /* → Lỗi này xuất hiện khi MCLK không phù hợp
           Fix: assigned-clock-rates trong DTS = sample_rate × mclk_fs */
    }

    /* Set TCR2: MSEL (MCLK source select) + DIV (BCLK divider) */
    regmap_update_bits(sai->regmap, FSL_SAI_TCR2,
                       FSL_SAI_CR2_MSEL_MASK | FSL_SAI_CR2_DIV_MASK,
                       FSL_SAI_CR2_MSEL(best_id) |
                       FSL_SAI_CR2_DIV(best_div - 1));
    return 0;
}
```

---

## 19. DMA Architecture — eDMA3 Audio Path

### 19.1 eDMA3 Audio Setup

```c
/* sound/soc/fsl/fsl_sai.c + sound/core/pcm_dmaengine.c */
static int fsl_sai_pcm_hw_params(struct snd_soc_component *component,
                                  struct snd_pcm_substream *substream,
                                  struct snd_pcm_hw_params *params) {
    struct fsl_sai *sai = snd_soc_dai_get_drvdata(dai);

    /* eDMA3 slave config cho SAI3 TX */
    struct dma_slave_config dma_cfg = {
        .direction      = DMA_MEM_TO_DEV,
        .dst_addr       = sai->res->start + FSL_SAI_TDR0, /* 0x42650220 */
        .dst_addr_width = DMA_SLAVE_BUSWIDTH_4_BYTES,
        .dst_maxburst   = 8,   /* FIFO watermark: 8 words = 32 bytes */
    };
    dmaengine_slave_config(sai->dma_chan, &dma_cfg);
}

/* DTS DMA binding */
/*
&sai3 {
    dmas = <&edma3 0 IMX95_EDMA_SAI3_TX>,   ← eDMA3, SAI3 TX request
           <&edma3 0 IMX95_EDMA_SAI3_RX>;
    dma-names = "tx", "rx";
};
*/
```

### 19.2 DMA Data Flow

```
AudioFlinger write PCM data
    │  (userspace → kernel shmem via mmap)
    ▼
DMA buffer in DDR (period_size × period_count)
    │  eDMA3 transfers automatically when SAI FIFO < watermark
    ▼
SAI3 TDR0 (0x42650220) — Transmit Data Register
    │  SAI serializes: 32-bit word → I2S bit stream
    ▼
I2S pins: BCLK (3.072 MHz) + LRCK (48 kHz) + DATA
    │
    ▼
TAS5828 I2S input → Class D amplifier → Speaker

Period size: 1024 frames = 1024 × 2ch × 2B = 4096 bytes
Buffer size: 4 periods  = 16384 bytes
DMA cycles through 4 periods liên tục → seamless audio
```

---

## 20. ASoC DAPM — Widget Graph & Power Sequencing

### 20.1 DAPM Widget Graph (TAS5828)

```
[Platform DAI — fsl_sai]        [Codec — TAS5828]
┌──────────────────────┐        ┌────────────────────────────────┐
│                      │        │                                │
│ CPU DAI TX  ─────────┼──link──▶ AIF IN (I2S digital input)    │
│                      │        │    │                          │
│ (powered when stream │        │    ▼                          │
│  is active)          │        │ Volume Control (ALSA mixer)   │
│                      │        │    │                          │
└──────────────────────┘        │    ▼                          │
                                │ DSP (EQ + DRC)                │
                                │    │                          │
                                │    ▼                          │
                                │ Class D Amplifier             │
                                │    │                          │
                                │    ▼                          │
                                │ Speaker Output  ──── Speaker  │
                                └────────────────────────────────┘
```

### 20.2 DAPM Power Startup Sequence

```c
/* sound/soc/codecs/tas5828.c */
static const struct snd_soc_dapm_widget tas5828_dapm_widgets[] = {
    SND_SOC_DAPM_AIF_IN("AIF IN", "Playback", 0, SND_SOC_NOPM, 0, 0),

    /* DSP: controlled by TAS5828 power register (I2C write) */
    SND_SOC_DAPM_PGA("DSP", TAS5828_REG_CTRL, 0, 0, NULL, 0),

    /* Output driver: separate I2C register */
    SND_SOC_DAPM_OUT_DRV("Class D", TAS5828_REG_CTRL, 1, 0, NULL, 0),

    SND_SOC_DAPM_OUTPUT("Speaker"),
};

/* Khi mở AudioTrack → DAPM traversal: */
/* T+0ms:   power on AIF IN (platform side enable SAI3) */
/* T+1ms:   power on DSP (I2C write to TAS5828 @ 0x4C) */
/* T+5ms:   power on Class D Amp */
/* T+10ms:  TAS5828 starts DC offset calibration internally */
/* T+110ms: Audio output stable → no pop noise */

/* QUAN TRỌNG: 110ms startup delay là bình thường cho TAS5828.
   Nếu muốn giảm: implement volume ramp (mute → play → unmute)
   trong HAL hoặc codec driver */
```

### 20.3 DAPM Debug

```bash
# Xem DAPM widget state
adb shell cat /sys/kernel/debug/asoc/imx95-19x19-evk/dapm
# Output: mỗi dòng = 1 widget với [on]/[off]

# Kiểm tra audio path có connect không
adb shell cat /sys/kernel/debug/asoc/imx95-19x19-evk/dapm_widget
# Expected khi playing:
#  "Speaker" power: on
#  "Class D" power: on
#  "DSP" power: on
#  "AIF IN" power: on

# Force enable một widget (debug không cần audio)
echo 1 > "/sys/kernel/debug/asoc/.../Class D Amp"

# Enable DAPM trace
echo 1 > /sys/kernel/debug/tracing/events/asoc/snd_soc_dapm_done/enable
echo 1 > /sys/kernel/debug/tracing/tracing_on
# Mở audio stream → xem trace:
cat /sys/kernel/debug/tracing/trace | grep dapm
```

---

## 21. Android Audio Stack — App đến SAI Register

### 21.1 Full Stack (từ app đến hardware)

```java
// 1. Android App (Java)
AudioTrack track = new AudioTrack.Builder()
    .setAudioAttributes(new AudioAttributes.Builder()
        .setUsage(AudioAttributes.USAGE_MEDIA).build())
    .setAudioFormat(new AudioFormat.Builder()
        .setSampleRate(48000).setEncoding(PCM_16BIT)
        .setChannelMask(STEREO).build())
    .build();
track.play();
track.write(pcmData, 0, pcmData.length);
```

```cpp
// 2. AudioFlinger MixerThread (frameworks/av/services/audioflinger/Threads.cpp)
bool AudioFlinger::MixerThread::threadLoop() {
    while (!exitPending()) {
        mAudioMixer->process();          // Mix all active tracks
        mOutput->stream->write(          // Write to HAL
            mOutput->stream,
            mMixBuffer, mixBufferSize);
    }
}
```

```cpp
// 3. Audio HAL AIDL (vendor/nxp/imx/audio_hw/)
ndk::ScopedAStatus NxpStreamOut::write(
    const vector<uint8_t>& buffer, int64_t* ret) {
    // tinyalsa write → ALSA PCM device
    pcm_write(mPcm,             // pcm handle for TAS5828 (card2)
              buffer.data(),
              buffer.size());
    // pcm_write → ioctl(fd, SNDRV_PCM_IOCTL_WRITEI_FRAMES, &xfer)
    *ret = buffer.size();
    return ndk::ScopedAStatus::ok();
}
```

```c
// 4. Kernel ALSA (sound/core/pcm_lib.c)
snd_pcm_sframes_t snd_pcm_lib_write(substream, buf, frames) {
    copy_from_user(runtime->dma_area + offset, buf, bytes);
    snd_pcm_update_hw_ptr(substream);
    // eDMA3 auto-transfers from dma_area to SAI TDR0
}
```

```c
// 5. fsl_sai_trigger() — enable SAI hardware
// sound/soc/fsl/fsl_sai.c
case SNDRV_PCM_TRIGGER_START:
    /* Enable FIFO DMA request (SAI3 TCSR, 0x42650000) */
    regmap_update_bits(sai->regmap, FSL_SAI_TCSR,
                       FSL_SAI_CSR_FRDE,   /* FIFO Request DMA Enable */
                       FSL_SAI_CSR_FRDE);

    /* Enable Transmitter */
    regmap_update_bits(sai->regmap, FSL_SAI_TCSR,
                       FSL_SAI_CSR_TE,     /* Transmitter Enable */
                       FSL_SAI_CSR_TE);
    // SAI3 TCSR @ 0x42650000 = 0x80000020
    // bit31 = TE=1, bit5 = FRDE=1
```

### 21.2 SAI3 Register Map Reference

```
SAI3 base: 0x42650000

Offset  Register  Description
0x00    TCSR      Transmit Control/Status
                  bit31: TE (Transmitter Enable) ← set khi play start
                  bit 5: FRDE (FIFO Request DMA Enable)
                  bit 4: FWDE (FIFO Warning DMA Enable)
                  bit 3: FEF (FIFO Error Flag — underrun!)
                  bit 2: SEF (Sync Error Flag)

0x04    TCR1      FIFO Watermark = TFW[4:0]
                  Set to 8: DMA triggers when FIFO ≤ 8 words

0x08    TCR2      Bit Clock config
                  bits[27:26] BCP=0 (clock polarity)
                  bits[25:24] MSEL=01 → MCLK1 source
                  bits[5:0]   DIV → BCLK = MCLK / (2×(DIV+1))
                  Ví dụ: MCLK=12.288MHz, DIV=1 → BCLK=3.072MHz

0x0C    TCR3      Channel Enable
                  TCE[3:0]: enable transmit channel N

0x10    TCR4      Frame config
                  FRSZ[4:0] = 1 → 2 words per frame (stereo)
                  SYWD[4:0] = 31 → 32-bit slot width
                  MF=1 → MSB first
                  FSE=1 → frame sync one bit before data

0x14    TCR5      Word N length
                  W0W[4:0] = 31 → word 0 = 32 bits
                  WNW[4:0] = 31 → other words = 32 bits
                  FBT[4:0] = 31 → first bit shifted = bit 31

0x20    TDR0      Transmit Data Register
                  DMA writes 32-bit samples here
                  Auto-serialized to I2S pins

0x40    TFR0      Transmit FIFO Read pointer
0x44    TFR1      Transmit FIFO Write pointer
                  FIFO depth = 32 words (128 bytes)

0x60    TMR       Transmit Mask Register
                  Bit N = 1 → mask (silence) word N in frame
```

### 21.3 Kiểm Tra SAI State

```bash
# Kiểm tra SAI3 đang active
adb shell devmem 0x42650000 32   # TCSR
# Khi play: 0x80000020 (TE=1, FRDE=1)
# Khi idle: 0x00000000 (tắt hoàn toàn)

adb shell devmem 0x42650008 32   # TCR2
# Decode MSEL bits[25:24]:
# 00 = Bus clock, 01 = MCLK1, 10 = MCLK2, 11 = MCLK3

# Kiểm tra underrun (FIFO error)
adb shell devmem 0x42650000 32   # Check bit3 (FEF)
# Nếu FEF=1: SAI FIFO underrun → audio glitch
# Nguyên nhân: DMA too slow, CPU overload, period size too small

# Xem PCM state
adb shell cat /proc/asound/card2/pcm0p/sub0/status
# RUNNING = đang phát, PREPARED = ready, SUSPENDED = STR
```


---

## PHẦN V — SUSPEND TO RAM (STR)

## 22. STR Overview — Power States & Modes

### 22.1 Power Modes trên i.MX95

```
RUN Mode:     A55 active, M33 active, M7 optional
              DDR: normal
              Power: full

IDLE Mode:    A55 WFI (kernel idle), M33 active
              A55 L1/L2 data retained, L3 retained
              DDR: auto clock gating
              Power: reduced ~30%

SUSPEND Mode: A55 power gated, M33 active (in AONMIX)
(= STR)       M7 optional (if LPA enabled)
              DDR: self-refresh / retention
              VDD_SOC: lowered via PMIC_STBY_REQ
              Power: reduced ~85-95%
              Wake latency: ~100-200ms

BBSM Mode:    = RTC mode
              Only NVCC_BBSM_1P8 domain alive
              M33 also off
              DDR: off (context lost)
              Power: ~µW
              Wake: cold boot required
```

### 22.2 STR Trigger

```bash
# Từ Android shell:
adb shell "echo mem > /sys/power/state"

# Verify loại sleep:
adb shell cat /sys/power/mem_sleep
# Expected: s2idle [deep]  ← "deep" = system STR (platform suspend)

# Set wakeup source trước (VD: RTC sau 30s):
adb shell "echo +30 > /sys/class/rtc/rtc0/wakealarm"
adb shell "echo mem > /sys/power/state"
```

---

## 23. STR Suspend Path — Source Code End-to-End

### 23.1 Linux PM Core

```c
/* kernel/power/suspend.c */
int pm_suspend(suspend_state_t state) { /* state = PM_SUSPEND_MEM */
    error = suspend_freeze_processes();
    /* LOG: "Freezing user space processes completed" */

    error = dpm_suspend_start(PMSG_SUSPEND);
    /* Calls .suspend() cho mỗi driver (bottom-up):
       fsl_sai_suspend():
         - Disable SAI TX/RX
         - Save SAI register set
         - Release DMA channel

       imx_rpmsg_suspend():
         - Send SUSPEND notification to M7 via RPMsg
         - M7 nhận, saves its state, acks

       scmi_clk_suspend():
         - Nothing: SM maintains clock state

       xhci_hcd suspend():
         - CRITICAL: nếu USB device attached → có thể timeout!
         - LOG: "xhci: CMD_RUN timeout" → suspend FAIL
    */

    error = dpm_suspend_noirq(PMSG_SUSPEND);
    /* GIC được disable trước bước này */

    /* Enter arch-specific suspend */
    error = suspend_ops->enter(PM_SUSPEND_MEM);
    /* → cpu_suspend() */
}
```

### 23.2 ATF BL31 — System Suspend

```c
/* plat/imx/imx95/imx95_psci.c */
void imx_domain_suspend(const psci_power_state_t *target_state) {
    unsigned int cpu = plat_my_core_pos(); /* = 0, last core */

    if (is_local_state_retn(SYSTEM_PWR_STATE(target_state))) {
        /* === SYSTEM-LEVEL SUSPEND (last CPU going down) === */

        /* 1. Save GIC redistributor context */
        plat_gic_save(cpu, &imx_gicv3_ctx);

        /* 2. Configure wakeup sources (GPC/BBSM) */
        imx_set_sys_wakeup(cpu, true);
        /* Enable RTC alarm, GPIO, BBSM wakeup in GPC */

        /* 3. Notify SM via SCMI: system suspending */
        imx9_scmi_sys_suspend();
        /* SM: saves power domain states,
                prepares DDR for retention */

        /* 4. Enter DDR self-refresh (RETENTION)
           CRITICAL: After this point, DDR is not accessible!
           All code from here must run from SRAM (OCRAM) */

        /* 4a. Copy warm boot handler to SRAM */
        imx_copy_bl31_to_sram();

        /* 4b. Execute DDR retention from SRAM */
        /* (Code pointer is switched to SRAM copy) */
        imx_dram_enter_retention();
        /*
         * imx_dram_enter_retention():
         *   a. DDRC: set PWRCTL.selfref_sw=1 (force self-refresh)
         *   b. Poll DDRC STAT.operating_mode == 3 (self-refresh OK)
         *   c. Gate DDR PHY DFI clock via CCM
         *   d. Gate DDR controller clock root
         */

        /* 5. Configure GPC for power gating */
        imx_set_sys_lpm(cpu, true);
        /* GPC_SLPCR: A55_PDN=1, VSTBY=1 */
        /* VSTBY=1 → GPC asserts PMIC_STBY_REQ after WFI */

        /* 6. Store warm resume entry in SRC GPR */
        mmio_write_32(SRC_BASE + SRC_GPR0,
                      (uintptr_t)&bl31_warm_entrypoint);
        mmio_write_32(SRC_BASE + SRC_GPR1, 0);
    }

    /* Enable GIC CPU interface */
    plat_gic_cpuif_enable();

    /* Execute WFI — CPU power-gated by GPC hardware */
    isb();
    dsb();
    wfi();
    /* ═══════════ SYSTEM IS SLEEPING HERE ═══════════ */
    /* Code sau đây chạy khi resume */
}
```

### 23.3 M33 SM trong Khi A55 Sleep

```c
/* imx-sm/sm/src/main.c — SM vẫn chạy liên tục */
void SM_WakeupHandler(void) {
    /* Được gọi khi wakeup event xảy ra (RTC, GPIO, etc.) */
    SM_PRINTF("SM: Wakeup event detected!\r\n");

    /* 1. DDR exit retention */
    SM_DdrExitRetention();
    /*
     * a. Enable DDR controller clock root via CCM
     * b. Enable DDR PHY DFI clock
     * c. DDRC: clear PWRCTL.selfref_sw (exit self-refresh)
     * d. Poll DDRC STAT: wait for exit self-refresh
     * e. Run DDR PHY re-training if needed
     */
    SM_PRINTF("SM: DDR exit retention OK\r\n");

    /* 2. Power on A55 domain */
    PWR_LmPower(LM_AP, SM_POWER_ON);

    /* 3. Kick A55 warm boot */
    /* A55 will start at bl31_warm_entrypoint (stored in SRC GPR0) */
    MU_TxTrigger(MU0_BASE);
    SM_PRINTF("SM: A55 resume triggered\r\n");
}
```

---

## 24. STR Resume Path — Source Code

### 24.1 ATF Warm Resume

```c
/* Runs from SRAM at bl31_warm_entrypoint */
void bl31_warm_entrypoint(void) {
    /* DDR đã được SM khôi phục trước khi A55 được kick */

    /* 1. Restore CPU architectural registers */
    imx_cpu_context_restore();

    /* 2. Re-enable GIC redistributor */
    plat_gic_redistif_on();

    /* 3. Restore GIC context */
    plat_gic_restore(cpu, &imx_gicv3_ctx);

    /* 4. Clear suspend configuration */
    imx_set_sys_wakeup(cpu, false);
    imx_set_sys_lpm(cpu, false);
    /* GPC_SLPCR: A55_PDN=0, VSTBY=0 */

    /* 5. Return to Linux kernel */
    /* A55 continues from after cpu_suspend() SMC call */
    bl31_exit_to_normal_world();
}
```

### 24.2 Linux Resume Path

```c
/* kernel/power/suspend.c — resume side */
/* dpm_resume_start() calls .resume() in reverse order: */

fsl_sai_resume():
    /* Restore SAI registers */
    regmap_write(sai->regmap, FSL_SAI_TCR2, saved_tcr2);
    /* Re-request SCMI clock */
    scmi_clk_set_rate(SAI3_CLK, 12288000);

imx_rpmsg_resume():
    /* Notify M7: A55 is back */
    rpmsg_send_notification(RESUME_NOTIFY);
    /* M7 resumes its SAI operation */
```

**STR Full Log:**
```
[100.000] PM: suspend entry (deep)
[100.005] Filesystems sync: 0.003 seconds
[100.010] Freezing user space processes completed
[100.030] fsl-sai 42650000.sai: suspend
[100.040] imx-rproc: notifying M7 of suspend
[100.100] PM: suspend devices took 0.070 seconds
[100.101] PM: noirq suspend of devices
[100.110] Disabling non-boot CPUs ...
[100.130] CPU5: shutdown
[100.131] PSCI: Entering system sleep
[100.132] ATF: DDR entering retention
[100.133] GPC: asserting PMIC_STBY_REQ
[100.133] System entering deep sleep (WFI)
           .... SYSTEM SLEEPING ....
[COM4]   SM: RTC alarm! Wakeup event detected
[COM4]   SM: DDR exit retention OK
[COM4]   SM: PMIC voltage restored
[COM4]   SM: A55 resume triggered
[130.000] PSCI: Resumed from deep sleep
[130.005] CPU1 is up ... CPU5 is up
[130.050] PM: resume devices took 0.025 seconds
[130.060] fsl-sai 42650000.sai: resumed
[130.070] PM: resume exit
```

---

## 25. M7 LPA — Low Power Audio khi A55 Sleep

### 25.1 SRTM Pattern

```
Kịch bản: A55 suspend nhưng M7 vẫn phát BT audio

A55 Android          M7 FreeRTOS
    │                     │
    │── SRTM_AUDIO_START ─▶│ M7 opens SAI1, starts DMA
    │── PCM data ──────────▶│ M7 buffers into ring buffer
    │── PCM data ──────────▶│
    │ [A55 entering suspend]│
    │── SRTM_SUSPEND ───────▶│ M7 acks: "I'll keep playing"
    │ [A55 off, DDR retention]│
    │                     │ M7 plays from TCM buffer
    │                     │ M7 needs more data: send WAKE via MU
    │ [A55 wakes briefly]  │
    │◀─ SRTM_DATA_REQ ───── │
    │── PCM data ──────────▶│
    │ [A55 suspends again]  │ M7 continues
    │ [Final wakeup]        │
    │── SRTM_AUDIO_STOP ───▶│ M7 stops SAI1
```

### 25.2 M7 SRTM Audio Service

```c
/* middleware/multicore/rpmsg-lite/middleware/srtm/ */
static srtm_status_t srtm_audio_rx_callback(
    srtm_service_t service, srtm_request_t request) {

    uint32_t cmd = SRTM_CommMessage_GetCommand(request);
    switch (cmd) {
    case SRTM_AUDIO_CMD_START:
        /* Init SAI1 (BT SCO SAI) */
        SAI_TxInit(SAI1, &sai_config);
        /* Start eDMA */
        SAI_TransferTxCreateHandleEDMA(SAI1, &handle, cb, NULL);
        SAI_TransferTxSetConfigEDMA(SAI1, &handle, &edma_cfg);
        SRTM_Message_SetStatus(request, SRTM_Status_Success);
        break;

    case SRTM_AUDIO_CMD_DATA:
        /* Buffer PCM data vào M7 ring buffer (TCM-based) */
        ring_buffer_write(SRTM_Message_GetPayload(request),
                         SRTM_Message_GetPayloadLen(request));
        break;

    case SRTM_AUDIO_CMD_SUSPEND:
        /* A55 going to sleep. Flag that we should request wake
           when ring buffer < threshold */
        lpa_mode = true;
        break;

    case SRTM_AUDIO_CMD_STOP:
        SAI_TxEnable(SAI1, false);
        lpa_mode = false;
        break;
    }
    return SRTM_Status_Success;
}

/* When ring buffer runs low in LPA mode: */
void lpa_request_wake(void) {
    /* Kick A55 via MU2 to wake up and send more data */
    MU_TxTrigger(MU2_BASE);
    /* A55 ISR: schedule wake-up, send SRTM_AUDIO_CMD_DATA */
}
```

---

## PHẦN VI — EXPERT TOPICS

## 26. Porting Guide — Custom Board từ EVK95

### 26.1 Components Thường Thay Đổi

```
EVK95 Component         Custom Board          Thay đổi cần làm
──────────────────────────────────────────────────────────────
LPDDR5 8GB (Micron)  → 4GB (Samsung)       OEI DDR config mới
TAS5828 (I2C 0x4C)   → MAX98357 (no I2C)   DTS: thay codec node
PCM1808 (I2C mic)    → INMP441 (PDM)       DTS: SAI→MICFIL
TEF6657A (FM tuner)  → không có            Xóa DTS node
PMIC PF9453          → khác layout         PMIC I2C addr/config
SAI3 pins            → routed khác         Pinctrl trong DTS
```

### 26.2 Step-by-Step Porting Workflow

```bash
# ═══════════ BƯỚC 1: OEI DDR — Nếu thay DDR chip ═══════════

# Lấy JEDEC timing specs từ DDR vendor
# NXP cung cấp DDR Stress Test Tool để verify timing
# Sửa DDR config:
cd imx-oei/boards/my_board/
cp -r mx95lp5 my_board
# Edit ddr_timing.h: tRCD, tRP, tRC, tRAS, ... từ JEDEC spec

make board=my_board oei=ddr DEBUG=1 r=B0 \
     DDR_CONFIG=MY_CONFIG all
# Test: nếu OEI log "Training FAIL" → adjust timing params

# ═══════════ BƯỚC 2: SM Config — Resources ═══════════

cp imx-sm/configs/mx95evk.cfg imx-sm/configs/my_board.cfg
# Thêm/bớt resources, sửa TRDC rules
# VD: thêm SAI6 nếu dùng thêm audio output:
# [RESOURCE]
# name = SAI6
# lm = AP
make config=my_board all

# ═══════════ BƯỚC 3: DTS — Board Description ═══════════

cp arch/arm64/boot/dts/freescale/imx95-19x19-evk.dts \
   arch/arm64/boot/dts/freescale/imx95-my-board.dts

# 3a. Thay codec (TAS5828 → MAX98357 — không có I2C):
# Xóa i2c3 tas5828 node
# Thêm simple-audio-card với max98357:
# max98357: max98357 {
#     compatible = "maxim,max98357a";
#     #sound-dai-cells = <0>;
#     sdmode-gpios = <&gpio3 10 GPIO_ACTIVE_HIGH>;
# };

# 3b. Thay mic (PCM1808 I2S → INMP441 PDM):
# Xóa SAI1-based mic node
# Thêm MICFIL (PDM filter) node:
# &micfil {
#     pinctrl-names = "default";
#     pinctrl-0 = <&pinctrl_pdm>;
#     assigned-clocks = <&scmi_clk IMX95_CLK_PDM>;
#     assigned-clock-rates = <49152000>;
#     status = "okay";
# };

# 3c. Update pinctrl nếu route khác:
# pinctrl_sai3: sai3grp {
#     fsl,pins = <
#         MX95_PAD_SAI3_TXFS__SAI3_TX_SYNC  0x31e
#         MX95_PAD_SAI3_TXC__SAI3_TX_BCLK   0x31e
#         MX95_PAD_SAI3_MCLK__SAI3_MCLK     0x31e
#         MX95_PAD_SAI3_TXD0__SAI3_TX_DATA00 0x31e
#     >;
# };

# ═══════════ BƯỚC 4: audio_policy_configuration.xml ═══════════

# Nếu audio card index thay đổi, sửa card number trong HAL
# Nếu bỏ FM tuner, xóa capture profile tương ứng

# ═══════════ BƯỚC 5: Verify ═══════════

# Boot và kiểm tra:
adb shell cat /proc/asound/cards           # card index đúng chưa?
adb shell i2cdetect -y -r 3               # codec trên I2C chưa?
adb shell dmesg | grep -E "sai|tas|max|micfil"  # driver probe OK?
adb shell tinyplay test.wav -D 2 -d 0    # test tinyplay trực tiếp
```

### 26.3 Common Custom Board Issues

| Vấn đề | Symptom | Root Cause | Fix |
|--------|---------|-----------|-----|
| Audio card index sai | HAL open /dev/pcmC2D0 fail | Codec bỏ, card re-numbered | Sửa card index trong HAL, thêm `audio_policy_configuration.xml` |
| I2C codec không phản hồi | `i2cdetect` không thấy addr | Power rail off, hoặc pin sai | Check DTS `power-domains`, check pinmux |
| DDR không ổn định | Kernel crash ngẫu nhiên | Timing params sai | Chạy `memtester 200M 5`, adjust OEI timing |
| Clock sai sau porting | SAI set_bclk fail | assigned-clock-rates không match | Đảm bảo rate = fs × mclk_fs |
| Boot không lên | OEI training fail | DDR config sai chip | Chạy DDR Stress Test Tool, rebuild OEI |

---

## 27. Security — AHAB, ELE, AVB, dm-verity

### 27.1 Chain of Trust

```
NXP Private Key (NXP HSM — không ai truy cập được)
    │ signs
    ▼
NXP Container (firmware-ele-imx-*.bin) → ELE Firmware
    │ AHAB verified by Boot ROM
    ▼
OEM Container (flash.bin) → SM + SPL + ATF + U-Boot
    │ AHAB verified by ELE (nếu OEM_CLOSED lifecycle)
    ▼
U-Boot → AVB (Android Verified Boot)
    │ verifies boot.img using vbmeta
    ▼
Linux Kernel (verified)
    │ mounts system/vendor với
    ▼
dm-verity → /system, /vendor, /product (read-only, hash-verified)
    │
    ▼
Android Keystore → OP-TEE (if available) / hardware-backed keys
```

### 27.2 AHAB Debug

```bash
# === U-Boot console ===
=> ahab_status
Lifecycle: 0x0010, OEM_OPEN   ← development phase
# Lifecycle values:
# 0x0001 = NXP provisioned
# 0x0010 = OEM_OPEN (development, AHAB warnings allowed)
# 0x0020 = OEM_CLOSED (production, strict authentication)
# 0x0080 = FIELD_RETURN

# AHAB events (authentication failures):
=> ahab_status
ELE Event[0] = 0x0087EE00
  CMD = AHAB_AUTH_CONTAINER_REQ (0x87)
  IND = AHAB_NO_AUTHENTICATION_IND (0xEE)
  → OEM_OPEN: boot continues despite unsigned image
  → OEM_CLOSED: boot FAILS

# Check ELE firmware version:
nxpele -f mimx9596 -d uboot_serial get-info
# Output: ELE firmware: 2.0.2, UUID: f42b...

# === Linux ===
adb shell dmesg | grep -i "ele\|ahab\|lifecycle"
# Expected:
# "ELE: firmware version 2.0.2"
# "ELE: lifecycle OEM_OPEN"
```

### 27.3 dm-verity

```bash
# Kiểm tra dm-verity active
adb shell cat /proc/mounts | grep dm-
# /dev/block/dm-0 /system ext4 ro,...
# /dev/block/dm-1 /vendor ext4 ro,...

# dm-verity violation log:
adb shell dmesg | grep verity
# "device-mapper: verity: 253:0: data block 12345 is corrupted"
# → System partition bị modify → device enters Orange/Red state

# Disable cho development (userdebug build only):
adb disable-verity    # cần userdebug build + root
adb reboot

# Re-enable:
adb enable-verity
adb reboot
```

---

## 28. Performance Tuning — Latency, DVFS, CPU Isolation

### 28.1 Audio Latency Breakdown

```
Component              Typical Value   Giảm bằng cách
────────────────────────────────────────────────────────────────
AudioFlinger period    5ms (240 frames) Giảm period_size trong HAL
HAL buffer             10ms (2 periods) Giảm PLAYBACK_PERIOD_COUNT
SAI FIFO depth         32 words/0.67ms  Giảm FIFO watermark (TCR1)
DMA transfer           < 0.1ms          Tăng eDMA priority
Codec startup          110ms (TAS5828)  Implement mute ramp
──────────────────────────────────────────────────────────────────
Total (cold start)     ~125ms
Total (steady state)   ~15ms (= 2 periods + FIFO)
```

### 28.2 CPU Governor & DVFS

```bash
# Set performance governor (disable frequency scaling)
for cpu in /sys/devices/system/cpu/cpu*/cpufreq; do
    echo performance > $cpu/scaling_governor
done

# Kiểm tra current frequency
cat /sys/devices/system/cpu/cpu0/cpufreq/scaling_cur_freq
# Expected: 1800000 (1.8 GHz = max)

# SCMI performance domain (DVFS via SM)
cat /sys/kernel/debug/arm-scmi/0/perf_domains
# Available performance levels: 0(low) 1(med) 2(high) 3(max)

# Lock A55 to max via SCMI:
echo 3 > /sys/kernel/debug/arm-scmi/0/perf_domain/0/level
```

### 28.3 CPU Isolation cho Audio Real-Time

```bash
# Trong kernel bootargs (via U-Boot):
# androidboot.boot_devices=... isolcpus=5 nohz_full=5 rcu_nocbs=5

# Sau khi boot, pin audioserver thread vào CPU5:
taskset -p -c 5 $(pgrep -f audioserver)

# Tăng priority (SCHED_FIFO):
chrt -r 80 -p $(pgrep -f audioserver)

# Kiểm tra:
cat /proc/$(pgrep -f audioserver)/status | grep Cpu
# Expected: VmRSS... Cpu_allowed: 20  ← bit5 = CPU5
```

---

## 29. Crash Analysis — Panic, ELE Events, JTAG

### 29.1 Kernel Panic Analysis

```bash
# Panic log mẫu:
# [123.456] Unable to handle kernel NULL pointer dereference at 0x0
# [123.457] ESR = 0x0000000096000044
# [123.458] EC = 0x25: DABT (current EL), IL = 32 bits
# [123.459] pc : fsl_sai_trigger+0x48/0x1c0 [snd_soc_fsl_sai]
# [123.460] lr : snd_soc_dai_trigger+0x58/0x90

# Decode ESR:
# ESR[31:26] = EC = 0x25 → Data Abort from current EL
# ESR[24:16] = SAS → access size
# ESR[9:6]   = DFSC → fault type

# Decode pc với addr2line:
arm-linux-gnueabihf-addr2line -e vmlinux \
    $(grep "fsl_sai_trigger" /proc/kallsyms | awk '{print $1}')
# Hoặc full stack decode:
python3 scripts/decode_stacktrace.py vmlinux < panic_log.txt

# Common audio crash causes:
# 1. NULL sai->regmap  → SAI driver unbind while DMA running
# 2. NULL rpmsg ept    → M7 crashed, endpoint gone
# 3. DMA use-after-free → period_elapsed callback after close
```

### 29.2 JTAG Debug Setup

```bash
# Hardware: J-Link Pro hoặc i.MX J-Link kết nối vào J30 (JTAG header)
# 4 serial ports qua J31 (USB-C):
#   ttyUSB0 = M7
#   ttyUSB1 = unused
#   ttyUSB2 = A55
#   ttyUSB3 = M33 SM

# JLink commands để debug M33 SM:
JLinkExe
connect
Device MIMX9596_M33
Interface SWD
Speed 4000
connect
halt
regs               # Xem registers
mem 0x1FFE0000 200 # Dump M33 TCM
set PC 0x1FFE0000  # Reset PC (careful!)

# JLink commands để debug A55 core 0:
Device MIMX9596_A55_0
connect
halt
info regs
# (A55 cores must be enabled by bootloader first)
```

---

## PHẦN VII — DEBUG COOKBOOK

## 30. Debug Recipes — Lệnh Thực Tế Theo Từng Layer

### 30.1 Boot Layer

```bash
# Setup UART terminals:
screen /dev/ttyUSB3 115200  # COM4 = M33 SM
screen /dev/ttyUSB2 115200  # COM3 = A55 kernel
screen /dev/ttyUSB0 115200  # COM1 = M7

# Flash flash.bin:
sudo dd if=flash.bin of=/dev/sdX bs=1k seek=32 conv=fsync
# hoặc qua UUU (USB):
uuu -b emmc flash.bin

# Interrupt U-Boot:
# Nhấn phím bất kỳ trong 2s sau khi thấy U-Boot banner
# Tại prompt:
=> printenv bootcmd     # xem boot command
=> setenv bootdelay -1  # disable autoboot (permanent debug)
=> saveenv
=> boot                 # manual boot

# Check ELE/AHAB tại U-Boot:
=> ahab_status
=> ele_mfg_tool         # nếu có manufacturing mode
```

### 30.2 SCMI Layer

```bash
# Enable SCMI kernel tracing
echo 'arm_scmi:scmi_xfer_begin arm_scmi:scmi_xfer_end' \
    > /sys/kernel/tracing/set_event
echo 1 > /sys/kernel/tracing/tracing_on
# ... trigger SCMI action (e.g., cat clock rate) ...
cat /sys/kernel/tracing/trace | grep scmi
# scmi_xfer_begin: id=5 proto=0x14 seq=42
# scmi_xfer_end:   id=5 proto=0x14 seq=42 ret=0
# Latency: thời gian giữa begin và end ≈ 50-200 µs

# Check SCMI protocols:
cat /sys/kernel/debug/arm-scmi/0/protocols
# 0x10 0x11 0x13 0x14 0x15 0x19 0x80 0x84

# Check SCMI errors:
cat /sys/kernel/debug/arm-scmi/0/errors

# Check all clocks:
cat /sys/kernel/debug/clk/clk_summary | head -50
cat /sys/kernel/debug/clk/clk_summary | grep sai3

# SM monitor:
# (COM4): > clock 66    → SAI3 rate
# (COM4): > scmi stat   → transaction count
```

### 30.3 RPMsg / M7 Layer

```bash
# Check M7 state:
cat /sys/class/remoteproc/remoteproc0/state  # running/offline

# Start/stop M7:
echo stop  > /sys/class/remoteproc/remoteproc0/state
echo "imx/m7-firmware.elf" > /sys/class/remoteproc/remoteproc0/firmware
echo start > /sys/class/remoteproc/remoteproc0/state

# Kiểm tra RPMsg channels:
ls /dev/rpmsg*
cat /sys/bus/virtio/devices/virtio0/features  # virtio features

# Test pingpong:
insmod /vendor/lib/modules/imx_rpmsg_pingpong.ko
dmesg | grep -E "rpmsg|pingpong"
# Expected: "get 1 (src: 0x1e)"

# Debug vring (nếu kernel có VIRTIO_DEBUG):
cat /sys/kernel/debug/virtio/virtio0/vqs/0/stats

# Check MU2 registers (M7↔A55):
devmem 0x42430000 32  # MU2_VER
devmem 0x4243012C 32  # MU2_RSR (RX status)
```

### 30.4 Audio Layer

```bash
# === ALSA level ===
cat /proc/asound/cards
# 0: ... imx-audio-hdmi
# 2: ... tas5828-audio  ← TAS5828 (speaker)

# Kiểm tra mixer controls:
tinymix -D 2              # card 2 (TAS5828)
tinymix -D 2 "Volume" 80  # Set volume 80%

# Test tinyplay/tinycap:
tinyplay test48k_s16le_stereo.wav -D 2 -d 0 -p 1024 -n 4
tinycap capture.pcm -D 1 -d 0 -c 2 -r 48000 -b 16

# SAI registers:
devmem 0x42650000 32  # TCSR: TE bit, FEF (underrun)
devmem 0x42650008 32  # TCR2: MSEL, DIV

# PCM state:
cat /proc/asound/card2/pcm0p/sub0/status
cat /proc/asound/card2/pcm0p/sub0/hw_params

# DAPM state:
cat /sys/kernel/debug/asoc/imx95-19x19-evk/dapm

# === Android level ===
dumpsys media.audio_flinger | grep -E "output|latency|underrun"
dumpsys media.audio_policy | grep -E "Audio Patch|setGain|port"

# Audio focus:
dumpsys audio | grep -A 5 "Audio Focus"

# Kiểm tra property quan trọng:
getprop ro.android.car.audio.enableaudiopatch
# Expected: true (nếu false → setPortGain không được gọi)
```

### 30.5 STR Layer

```bash
# Pre-suspend check:
cat /sys/power/state         # Available states
cat /sys/power/mem_sleep     # [deep] = system suspend
cat /sys/kernel/debug/wakeup_sources | head  # Active wakeup sources

# Enable PM debug:
echo N > /sys/power/pm_async
echo 1 > /sys/power/pm_debug_messages
dmesg -w &

# Trigger STR với RTC wakeup:
echo +30 > /sys/class/rtc/rtc0/wakealarm
echo mem > /sys/power/state

# Nếu suspend bị blocked:
dmesg | grep "PM:"
# Look for: "PM: Failed to suspend device <name>: error -110"
# Fix: identify driver and fix its .suspend() callback

# STR statistics:
cat /sys/kernel/debug/suspend_stats
# success, fail, last_failed_dev, last_failed_errno

# Test DDR integrity qua STR:
md5sum /data/test.bin      # trước suspend
# ... suspend + resume ...
md5sum /data/test.bin      # sau resume → phải khớp
```

---

## 31. Log Reference — Annotated Boot Timeline

```
══════════════════════════════════════════════════════════════════
COM4 (M33 System Manager)            Timestamp (từ POR)
══════════════════════════════════════════════════════════════════
[OEI] DDR: LPDDR5                    T+10ms
[OEI] DDR: DDRC configured           T+15ms
[OEI] DDR: PHY firmware loaded       T+20ms
[OEI] DDR: Running WrLvl...          T+25ms
[OEI] DDR: Training PASS             T+48ms
[OEI] DDR: Size=8192MB               T+49ms
SM: v2.4.0 NXP System Manager        T+51ms  ← SM starts
SM: TRDC configured                  T+52ms
SM: Power init complete              T+53ms
SM: Clock: 128 clocks                T+54ms
SM: LM1 (AP) started @ 0x20480000   T+55ms  ← A55 RELEASED
>                                    T+56ms  ← SM prompt

══════════════════════════════════════════════════════════════════
COM3 (A55 U-Boot → Kernel)
══════════════════════════════════════════════════════════════════
U-Boot SPL 2024.04                   T+56ms  ← SPL starts
DDRMIX: powered UP                   T+57ms
NOTICE: BL31: v2.12.0               T+58ms  ← ATF runs
U-Boot 2024.04  CPU: i.MX95 1.8GHz  T+200ms ← U-Boot proper
DRAM:  7.8 GiB                       T+210ms
MMC:   FSL_SDHC: 0, 1               T+220ms
Booting device 0:1 in 2...          T+250ms
Starting kernel ...                  T+3000ms
Booting Linux on CPU 0x00           T+3010ms
Machine model: NXP i.MX95 19x19 EVK T+3020ms
GIC: Using split EOI/Deactivate     T+3100ms
SCMI: 128 clock domains             T+3500ms
imx-rproc: probed                   T+3800ms
fsl-sai 42650000.sai: probe OK      T+4000ms
smp: Brought up 1 node, 6 CPUs      T+4200ms
init: first stage started!          T+8000ms  ← Android init
servicemanager: starting             T+12000ms
audioserver: AudioFlinger started   T+15000ms
CarAudioService: init done          T+20000ms
Zygote: started                     T+25000ms
SystemServer: boot completed        T+35000ms

══════════════════════════════════════════════════════════════════
COM1 (M7 FreeRTOS) — sau khi remoteproc start
══════════════════════════════════════════════════════════════════
[M7] FreeRTOS v10.5.1               T+10500ms ← remoteproc start
[M7] RPMsg-Lite remote init         T+10510ms
[M7] Waiting for link up...         T+10515ms
[M7] Link UP! EPT 30 ready          T+10600ms ← /dev/rpmsg0 created
```

---

## 32. Bảng Tra Cứu: Lỗi → Root Cause → Fix

### 32.1 Boot Failures

| Log / Symptom | Root Cause | Fix |
|---------------|-----------|-----|
| COM4 hoàn toàn im lặng sau power on | Boot mode pins sai / PMIC không đủ điện / flash.bin không đúng offset | Check SW7[1:4], measure VDD_SOC, re-flash |
| `[OEI] DDR: Training FAIL ret=-1` | DDR PHY config sai chip vendor/size/timing | Rebuild OEI với DDR_CONFIG đúng |
| `[OEI] DDR: Training TIMEOUT` | Clock đến DDR PHY bị missing | Check OEI DDR_CONFIG, verify CCM clock |
| `DDRMIX is powered OFF — OEI did not run` | OEI không có trong flash.bin | Rebuild flash.bin với OEI included |
| `if MU not probed, hang` (SPL không boot) | SM chưa init MU / SM crash | Check SM log COM4, SM may have crashed |
| `NOTICE: BL31` rồi im lặng | ATF crash (UART clock missing) | Check ATF console config, SCMI clock init |
| U-Boot: `## Error: DTB not found` | DTB load address sai | Check container.cfg load addresses |
| Kernel `Unable to handle kernel NULL` at `fsl_sai_trigger` | regmap NULL (SAI driver bug) | Check SAI driver init order, power domain |
| Android boot loop tại init | SELinux policy reject | Add `androidboot.selinux=permissive` tạm thời |

### 32.2 Audio Failures

| Log / Symptom | Root Cause | Fix |
|---------------|-----------|-----|
| `fmqByteCount=0` trong AudioFlinger logs | AudioPolicyManager không route tới HAL | Check `ro.android.car.audio.enableaudiopatch=true` |
| `setPortGain() not called` | `enableaudiopatch=false` | Set `true` trong device.mk |
| `SAI: failed to get required clock rate 3072000Hz` | MCLK không phù hợp với sample rate | `assigned-clock-rates = <12288000>` (= 48000 × 256) |
| `SAI: no valid master clock` | `clock-names` thứ tự sai trong DTS | Check thứ tự clock-names = "bus mclk0 mclk1..." |
| SAI TCSR bit3 (FEF) set = underrun | DMA quá chậm hoặc period size quá nhỏ | Tăng period_size, giảm CPU load, check eDMA priority |
| TAS5828 không respond | I2C address sai / VDD_CODEC off | `i2cdetect -y 3`, check power-domain DTS |
| `tas5828_config.json not found` | Wrong path trong vendor | Đặt file đúng: `/vendor/etc/tas5828_config.json` |
| Tiếng click/pop khi mở audio | DAPM power sequence quá nhanh | Implement mute ramp trong codec driver |
| Volume không thay đổi | `enableaudiopatch=false` hoặc ALSA control name sai | Check mixer control name với `tinymix -D 2` |

### 32.3 RPMsg / M7 Failures

| Log / Symptom | Root Cause | Fix |
|---------------|-----------|-----|
| `SCMI LMM boot failed` | SM không có LM2 configured | Check SM mx95evk.cfg có M7 LM, check fsl,lmm-id |
| `virtio0: timeout waiting for M7 link` | M7 firmware không gọi `rpmsg_lite_remote_init()` | Check M7 firmware source, verify shared mem address |
| `/dev/rpmsg0 not found` | M7 không gọi `rpmsg_ns_announce()` | Verify M7 firmware, check EPT address |
| `RL_ERR_NO_BUFF` (M7 side) | Vring buffer pool exhausted | Tăng vdev0buffer size trong DTS (0x100000 → 0x200000) |
| M7 crash (COM1 stops output) | M7 firmware bug, stack overflow | Check M7 stack size, verify M7 TCM layout |
| RPMsg data corrupt | Cache coherency issue | Ensure noncacheable memory region cho RPMsg shmem |

### 32.4 STR Failures

| Log / Symptom | Root Cause | Fix |
|---------------|-----------|-----|
| `PM: failed to suspend device xhci_hcd: -110` | USB xHCI suspend timeout | Set `echo on > /sys/bus/usb/devices/.../power/control` hoặc fix DTS |
| `PM: failed to suspend device fsl-sai: -EBUSY` | SAI stream không close trước suspend | Implement proper .suspend() trong SAI driver |
| System không wake sau STR | Wakeup source không được configure | Enable RTC/GPIO wakeup: `echo +30 > /sys/class/rtc/rtc0/wakealarm` |
| DDR data corrupt sau resume | DDR exit retention fail / timing issue | Check ATF `dram_exit_retention()`, verify DDR init after resume |
| `SCMI clock error after resume` | SM reset clock state | Re-request clocks trong driver `.resume()` callback |
| System resets ngay sau enter suspend | PMIC sequence issue | Check PMIC_STBY_REQ handling, VDD_SOC timing |

### 32.5 TRDC / Security

| Log / Symptom | Root Cause | Fix |
|---------------|-----------|-----|
| `Unhandled fault: synchronous external abort` ở M33 TCM address | A55 truy cập M33 TCM trái phép | Bug trong driver; chỉ SM mới được access M33 TCM |
| `ELE Event: AHAB_NO_AUTHENTICATION_IND` | Image không được sign | OK với OEM_OPEN lifecycle; sẽ fail ở OEM_CLOSED |
| `dm-verity: data block X is corrupted` | /system hoặc /vendor partition bị modify | Re-flash partition; hoặc `adb disable-verity` với userdebug |

---

## Appendix A — File & Repo Reference

```
Repository                  URL                                   Mục đích
────────────────────────────────────────────────────────────────────────────
imx-sm         github.com/nxp-imx/imx-sm            System Manager (M33)
imx-oei        github.com/nxp-imx/imx-oei           DDR init OEI (M33)
imx-atf        github.com/nxp-imx/imx-atf           ATF BL31 (A55 EL3)
uboot-imx      github.com/nxp-imx/uboot-imx         U-Boot
linux-imx      github.com/nxp-imx/linux-imx         Linux kernel
rpmsg-lite     github.com/nxp-mcuxpresso/rpmsg-lite  RPMsg-Lite (M7 FreeRTOS)
```

## Appendix B — Key Source Files

```
Boot:
  arch/arm/mach-imx/imx9/scmi/imximage.cfg      Container structure
  board/freescale/imx95_evk/spl.c               SPL init
  plat/imx/imx95/imx95_bl31_setup.c            ATF platform init
  plat/imx/imx95/imx95_psci.c                  PSCI (CPU on/off/suspend)
  imx-sm/sm/src/main.c                          SM main loop
  imx-sm/sm/src/lmm/sm_lmm.c                   LM_Boot()

SCMI:
  drivers/firmware/arm_scmi/vendors/imx/imx-sm-lmm.c  LMM protocol
  drivers/firmware/arm_scmi/clock.c                    Clock protocol
  drivers/firmware/arm_scmi/smc.c                      SMC transport
  drivers/clk/imx/clk-imx95.c                          Linux clock driver

RPMsg / Remoteproc:
  drivers/remoteproc/imx_rproc.c               Linux remoteproc
  middleware/multicore/rpmsg-lite/rpmsg_lite.c  M7 RPMsg-Lite

Audio:
  sound/soc/fsl/fsl_sai.c                      SAI driver
  sound/soc/codecs/tas5828.c                   TAS5828 codec
  frameworks/av/services/audioflinger/Threads.cpp  AudioFlinger
  packages/services/Car/service/src/.../CarAudioService.java  AAOS

STR:
  kernel/power/suspend.c                       Linux PM core
  plat/imx/imx95/imx95_psci.c                 ATF suspend/resume
```

## Appendix C — UART Quick Reference

```
Kết nối J31 (USB-C trên EVK) → 4 COM ports
  ttyUSB0 (COM1): Cortex-M7 FreeRTOS debug
  ttyUSB1 (COM2): Không dùng thường
  ttyUSB2 (COM3): Cortex-A55 U-Boot + Linux + Android
  ttyUSB3 (COM4): Cortex-M33 System Manager

Baudrate: 115200 8N1 tất cả ports
Linux: screen /dev/ttyUSB3 115200
Windows: TeraTerm / PuTTY

Quan trọng: Luôn mở COM4 trước khi power on để không miss SM log
```

## Appendix D — Expert Reading List

```
NXP Documents:
  IMX95RM        — i.MX95 Reference Manual (must-read chapters: 4,9,10,11,12,18,19,33)
  IMX95IEC       — i.MX95 Data Sheet (power modes, supply voltages)
  UG10163        — i.MX Linux User's Guide
  AN14748        — How to Run App on M7 Core of i.MX95
  AN14641        — Fast and Secure Boot (Falcon Mode)
  AN14533        — Anti-rollback Protection
  AN13400        — M Core Low Power Audio Playback

ARM Specifications:
  DEN0022        — PSCI (Power State Coordination Interface)
  DEN0056        — SCMI (System Control and Management Interface)
  ARM DDI 0595   — Cortex-A55 TRM
  ARM IHI 0048   — AXI Protocol Specification

Android/AOSP:
  Android Automotive Audio HAL specification
  AAOS Car Audio Focus documentation
  Audio Policy Configuration reference

Upstream Kernel Patches (read to understand NXP intent):
  lore.kernel.org — search "imx95" from peng.fan@nxp.com (2024-2025)
  Particularly: LMM/CPU protocol patches, SCMI BBM/MISC patches
```

