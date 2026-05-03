# Multi-Core Boot & IPC — Tài liệu chuyên sâu
## Dành cho: Embedded Android Automotive / NXP i.MX95 / Renesas R-Car

---

## Mục lục

**Phần nền tảng — đọc theo thứ tự này:**
0. [Chương 0 — Boot Timeline Đầy Đủ: Power ON → Android](#chương-0--boot-timeline-đầy-đủ-power-on--android-running)
0B. [Chương 0B — MU Hardware: Cơ chế vật lý giao tiếp core](#chương-0b--mu-messaging-unit-hardware--cơ-chế-vật-lý)
0C. [Chương 0C — Virtio Layer: Cầu nối Remoteproc ↔ RPMsg](#chương-0c--virtio-layer-cầu-nối-giữa-remoteproc-và-rpmsg)

**Phần cốt lõi:**
1. [Hướng 1 — SMP Boot Flow (ARM64 Kernel)](#hướng-1--smp-boot-flow-arm64-kernel)
2. [Hướng 2 — ATF/TF-A & PSCI](#hướng-2--atftf-a--psci)
3. [Hướng 3 — AMP với Remoteproc + RPMsg](#hướng-3--amp-với-remoteproc--rpmsg)
4. [Kiến trúc tổng quan i.MX95](#kiến-trúc-tổng-quan-imx95)
5. [Practical Lab](#practical-lab)

**Phần nâng cao:**
6. [Chương 4 — SCMI](#chương-4--scmi-system-control--management-interface)
7. [Chương 5 — Cache Coherency & Shared Memory](#chương-5--cache-coherency--shared-memory-giữa-các-core)
8. [Chương 6 — ELE (EdgeLock Enclave)](#chương-6--ele-edgelock-enclave--security-core)
9. [Chương 7 — Power Management & Power Domains](#chương-7--power-management--power-domains)
10. [Chương 8 — Inter-Core Debugging Chuyên sâu](#chương-8--inter-core-debugging-chuyên-sâu)
11. [Chương 9 — Cortex-M33 System Manager](#chương-9--cortex-m33-system-manager--core-thứ-3-của-evk95)
12. [Chương 10 — CPU Frequency Scaling DVFS](#chương-10--cpu-frequency-scaling-dvfs--thực-tế-với-android)
13. [Chương 11 — Inter-Core Synchronization Patterns](#chương-11--inter-core-synchronization-patterns)
14. [Chương 12 — Automotive Safety Core Coordination](#chương-12--automotive-specific-safety-core-coordination)
15. [Sơ đồ tổng kết đầy đủ](#tổng-kết--sơ-đồ-đầy-đủ-imx95-multi-core)

---

## Kiến trúc tổng quan i.MX95

```
┌─────────────────────────────────────────────────────────────────┐
│                        NXP i.MX95 SoC                           │
│                                                                 │
│  ┌──────────────────────────┐   ┌──────────────────────────┐   │
│  │   Cortex-A55 Cluster     │   │     Cortex-M7            │   │
│  │   (4 cores, SMP)         │   │     (Real-time)          │   │
│  │                          │   │                          │   │
│  │  Core0  Core1  Core2  Core3  │   │  FreeRTOS / bare-metal   │   │
│  │   ↑                     │   │                          │   │
│  │  Boot CPU                │   │  - CAN handling          │   │
│  │                          │   │  - Audio DSP offload     │   │
│  │  Android 15 (Linux)      │   │  - SOME/IP low-latency   │   │
│  └──────────┬───────────────┘   └──────────┬───────────────┘   │
│             │                              │                    │
│             │     ┌────────────────┐       │                    │
│             └────►│  MU (Messaging │◄──────┘                    │
│                   │   Unit) HW     │                            │
│                   └────────────────┘                            │
│                                                                 │
│  ┌──────────────────────────┐   ┌──────────────────────────┐   │
│  │    Cortex-M33            │   │    ELE (EdgeLock)         │   │
│  │    (Safety/Security)     │   │    Security Subsystem     │   │
│  └──────────────────────────┘   └──────────────────────────┘   │
│                                                                 │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │                 Shared OCRAM / DDR                        │  │
│  │         (vring buffers, resource table, IPC data)         │  │
│  └──────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────┘
```

---

## Hướng 1 — SMP Boot Flow (ARM64 Kernel)

### 1.1 Tổng quan

SMP (Symmetric Multi-Processing): Tất cả core Cortex-A55 cùng kiến trúc, chạy cùng Linux kernel image.

**Flow tổng quát:**
```
Power ON → BootROM → SPL → ATF → U-Boot → Linux (Core0 only)
                                              │
                                      smp_init() gọi cpu_up(1..3)
                                              │
                                    PSCI CPU_ON → ATF wake Core1..3
                                              │
                                    secondary_start_kernel()
                                              │
                                    Core1..3 join scheduler → SMP ready
```

### 1.2 Source Code — Kernel SMP Init

**File:** `arch/arm64/kernel/smp.c`

```c
/*
 * smp_init() — được gọi từ kernel_init() sau khi Core0 setup xong
 * File: init/main.c
 */
void __init smp_init(void)
{
    unsigned int cpu;

    /* Bring up các secondary CPUs */
    for_each_present_cpu(cpu) {
        if (num_online_cpus() >= setup_max_cpus)
            break;
        if (!cpu_online(cpu))
            cpu_up(cpu, CPUHP_ONLINE);  /* kick từng core */
    }
}
```

```c
/*
 * cpu_up() cuối cùng gọi __cpu_up() → boot_secondary()
 * File: arch/arm64/kernel/smp.c
 */
int __cpu_up(unsigned int cpu, struct task_struct *idle)
{
    int ret;

    /*
     * Set secondary_entry là điểm nhảy của Core mới
     * Core mới sẽ bắt đầu execute từ secondary_holding_pen
     */
    secondary_data.task = idle;
    secondary_data.stack = task_stack_page(idle) + THREAD_SIZE;

    /* Gọi platform-specific boot, thường là PSCI */
    ret = boot_secondary(cpu, idle);
    if (ret) {
        pr_err("CPU%u: failed to boot: %d\n", cpu, ret);
        return ret;
    }

    /* Chờ Core mới online */
    wait_for_completion_timeout(&cpu_running,
                                msecs_to_jiffies(5000));
    return 0;
}
```

```c
/*
 * secondary_start_kernel() — Core1..3 nhảy vào đây
 * File: arch/arm64/kernel/smp.c
 */
asmlinkage notrace void secondary_start_kernel(void)
{
    struct mm_struct *mm = &init_mm;
    unsigned int cpu = smp_processor_id();

    /* Setup MMU, cache cho core này */
    cpuinfo_store_cpu();
    store_cpu_topology(cpu);

    /* Notify Core0 rằng mình đã ready */
    complete(&cpu_running);

    /* Core này join vào scheduler */
    cpu_startup_entry(CPUHP_AP_ONLINE_IDLE);
    /* Không bao giờ return — core này chạy scheduler loop */
}
```

### 1.3 Device Tree — CPU enable-method

```dts
/* arch/arm64/boot/dts/freescale/imx95.dtsi */
cpus {
    #address-cells = <1>;
    #size-cells = <0>;

    cpu-map {
        cluster0 {
            core0 { cpu = <&cpu0>; };
            core1 { cpu = <&cpu1>; };
            core2 { cpu = <&cpu2>; };
            core3 { cpu = <&cpu3>; };
        };
    };

    cpu0: cpu@0 {
        compatible = "arm,cortex-a55";
        device_type = "cpu";
        reg = <0x0>;
        enable-method = "psci";          /* ← dùng PSCI để wake */
        next-level-cache = <&l2_cache>;
        cpu-idle-states = <&CPU_SLEEP>;
    };

    cpu1: cpu@100 {
        compatible = "arm,cortex-a55";
        device_type = "cpu";
        reg = <0x100>;
        enable-method = "psci";
    };

    cpu2: cpu@200 {
        compatible = "arm,cortex-a55";
        device_type = "cpu";
        reg = <0x200>;
        enable-method = "psci";
    };

    cpu3: cpu@300 {
        compatible = "arm,cortex-a55";
        device_type = "cpu";
        reg = <0x300>;
        enable-method = "psci";
    };
};
```

### 1.4 Spin-table method (U-Boot không có PSCI)

Một số board cũ dùng spin-table thay PSCI:

```dts
cpu1: cpu@1 {
    enable-method = "spin-table";
    cpu-release-addr = <0x0 0xfff8>;  /* địa chỉ Core0 ghi entry point */
};
```

```c
/* Core1..3 spin trong vòng này ở EL2/EL1 */
secondary_holding_pen:
    ldr  x0, =secondary_holding_pen_release  /* đọc địa chỉ */
    cbz  x0, secondary_holding_pen           /* nếu = 0, tiếp tục spin */
    br   x0                                  /* nhảy tới entry point */
```

```c
/* Core0 ghi địa chỉ để release Core1 */
void write_pen_release(u64 val)
{
    void *start = (void *)&secondary_holding_pen_release;
    *((u64 *)start) = val;
    /* flush cache để Core1 thấy giá trị mới */
    __flush_dcache_area(start, sizeof(u64));
    sev();  /* Send Event — đánh thức Core1 khỏi WFE */
}
```

### 1.5 Debug SMP boot

```bash
# Xem số core online
adb shell cat /sys/devices/system/cpu/online
# Output: 0-3  (4 cores)

# Hotplug — tắt Core3
adb shell echo 0 > /sys/devices/system/cpu/cpu3/online

# Kernel log SMP boot
adb shell dmesg | grep -i "cpu\|smp\|psci"
# [    0.000000] SMP: Allowing 4 CPUs up to 4
# [    0.428315] CPU1: Booted secondary processor 0x0000000100
# [    0.432190] CPU2: Booted secondary processor 0x0000000200
# [    0.435862] CPU3: Booted secondary processor 0x0000000300
```

---

## Hướng 2 — ATF/TF-A & PSCI

### 2.1 ATF là gì?

**ATF = ARM Trusted Firmware** (hay TF-A = Trusted Firmware-A)

ATF chạy ở **EL3** (Exception Level 3) — mức đặc quyền cao nhất của ARM, không có gì có thể override. Đây là "security monitor" của toàn SoC.

```
Exception Level Stack (ARM64):
┌─────────────────────────────────────────────┐
│  EL3 — ATF/TF-A (Secure Monitor)            │  ← highest privilege
│  - PSCI implementation                      │
│  - Secure boot chain                        │
│  - SMC (Secure Monitor Call) handler        │
├─────────────────────────────────────────────┤
│  EL2 — Hypervisor (U-Boot, KVM)             │
├─────────────────────────────────────────────┤
│  EL1 — OS Kernel (Linux)                    │  ← kernel chạy đây
├─────────────────────────────────────────────┤
│  EL0 — User Applications                    │  ← app Android chạy đây
└─────────────────────────────────────────────┘
```

### 2.2 Boot Flow với ATF

```
BootROM (EL3)
    │
    ▼
SPL (EL3 → chuyển sang EL2 khi load U-Boot)
    │
    ▼
BL2 — ATF early boot (EL3)
    │ setup secure memory, authenticate BL3x
    ▼
BL31 — ATF Runtime (EL3, luôn present)
    │ đây là PSCI handler
    ▼
BL32 — TEE/OP-TEE (EL1-Secure, optional)
    │
    ▼
BL33 — U-Boot (EL2)
    │
    ▼
Linux Kernel (EL1)
```

### 2.3 PSCI — Power State Coordination Interface

PSCI là chuẩn ARM (SMCCC — SMC Calling Convention) để kernel/U-Boot giao tiếp với ATF để quản lý power.

**PSCI Function IDs (ARM spec):**

```c
/* File: include/uapi/linux/psci.h */
#define PSCI_0_2_FN_BASE            0x84000000
#define PSCI_0_2_FN(n)              (PSCI_0_2_FN_BASE + (n))
#define PSCI_0_2_FN64_BASE          0xC4000000
#define PSCI_0_2_FN64(n)            (PSCI_0_2_FN64_BASE + (n))

#define PSCI_0_2_FN_PSCI_VERSION    PSCI_0_2_FN(0)
#define PSCI_0_2_FN_CPU_SUSPEND     PSCI_0_2_FN(1)
#define PSCI_0_2_FN_CPU_OFF         PSCI_0_2_FN(2)
#define PSCI_0_2_FN64_CPU_ON        PSCI_0_2_FN64(3)  /* ← quan trọng nhất */
#define PSCI_0_2_FN_AFFINITY_INFO   PSCI_0_2_FN(4)
#define PSCI_0_2_FN_SYSTEM_OFF      PSCI_0_2_FN(8)
#define PSCI_0_2_FN_SYSTEM_RESET    PSCI_0_2_FN(9)
```

**Kernel gọi PSCI CPU_ON:**

```c
/* File: drivers/firmware/psci/psci.c */
static int psci_cpu_on(unsigned long cpuid, unsigned long entry_point)
{
    int err;
    u32 fn = psci_function_id[PSCI_FN_CPU_ON];

    /*
     * Gọi SMC instruction để trap vào EL3 (ATF)
     * fn = PSCI_0_2_FN64_CPU_ON
     * cpuid = MPIDR của core cần boot (vd: 0x100 cho Core1)
     * entry_point = địa chỉ secondary_startup
     */
    err = invoke_psci_fn(fn, cpuid, entry_point, 0);
    return psci_to_linux_errno(err);
}

/* invoke_psci_fn cuối cùng thực thi: */
static noinline int __invoke_psci_fn_smc(u64 function_id,
                                          u64 arg0, u64 arg1, u64 arg2)
{
    struct arm_smccc_res res;
    arm_smccc_smc(function_id, arg0, arg1, arg2,
                  0, 0, 0, 0, &res);  /* SMC instruction */
    return res.a0;
}
```

### 2.4 ATF PSCI Implementation (i.MX95)

```c
/*
 * File: plat/imx/imx95/imx95_psci.c (trong TF-A repo)
 * ATF handler khi nhận SMC CPU_ON
 */
int plat_setup_psci_ops(uintptr_t sec_entrypoint,
                         const plat_psci_ops_t **psci_ops)
{
    /* Lưu entry point của secondary core */
    imx_mailbox_init(sec_entrypoint);
    *psci_ops = &imx_plat_psci_ops;
    return 0;
}

static const plat_psci_ops_t imx_plat_psci_ops = {
    .cpu_standby          = imx_cpu_standby,
    .pwr_domain_on        = imx_pwr_domain_on,      /* ← CPU_ON handler */
    .pwr_domain_off       = imx_pwr_domain_off,
    .pwr_domain_suspend   = imx_pwr_domain_suspend,
    .pwr_domain_on_finish = imx_pwr_domain_on_finish,
    .system_off           = imx_system_off,
    .system_reset         = imx_system_reset,
};

int imx_pwr_domain_on(u_register_t mpidr)
{
    unsigned int core_id = MPIDR_AFFLVL0_VAL(mpidr);

    /* Release core khỏi reset */
    imx_set_cpu_pwr_on(core_id);

    return PSCI_E_SUCCESS;
}
```

### 2.5 Debug ATF

```bash
# Xem PSCI version được kernel detect
adb shell cat /sys/bus/platform/devices/psci/psci_version
# Output: 1.1

# Kernel log PSCI
adb shell dmesg | grep -i psci
# [    0.000000] PSCI v1.1 detected in firmware.
# [    0.000000] PSCI: Using standard PSCI v0.2 function IDs
# [    0.000000] PSCI: Trusted OS migration not required

# SMC log (nếu ATF build với DEBUG=1)
# Xem qua UART0 — ATF in ra serial trước kernel boot
```

---

## Hướng 3 — AMP với Remoteproc + RPMsg

> **Đây là hướng quan trọng nhất với Android Automotive**

### 3.1 Kiến trúc tổng thể

```
┌──────────────────────────────────────────────────────────────────┐
│                    Linux (Cortex-A55)                            │
│                                                                  │
│  Userspace:                                                      │
│  ┌─────────────────┐  ┌─────────────────┐  ┌────────────────┐  │
│  │  Android App    │  │  HAL Service    │  │  rpmsg-char    │  │
│  │  (Java/Kotlin)  │  │  (C++ AIDL)    │  │  /dev/rpmsg0   │  │
│  └────────┬────────┘  └────────┬────────┘  └───────┬────────┘  │
│           │ Binder              │ Binder             │ read/write│
│  ─────────┼─────────────────────┼────────────────────┼───────── │
│  Kernel:  │                     │                    │          │
│  ┌────────▼──────────────────────▼────────────────────▼───────┐ │
│  │              RPMsg Framework (kernel)                       │ │
│  │  rpmsg_char  rpmsg_tty  rpmsg_audio  ...                   │ │
│  └────────────────────────┬────────────────────────────────────┘ │
│                           │                                      │
│  ┌────────────────────────▼────────────────────────────────────┐ │
│  │              Remoteproc Framework                           │ │
│  │  - Load firmware (.elf) lên M7                              │ │
│  │  - Manage lifecycle (start/stop/crash)                      │ │
│  │  - Setup shared memory (vring)                              │ │
│  └────────────────────────┬────────────────────────────────────┘ │
│                           │ MU (Messaging Unit) driver           │
└───────────────────────────┼──────────────────────────────────────┘
                            │ Hardware MU interrupt
┌───────────────────────────┼──────────────────────────────────────┐
│                    FreeRTOS (Cortex-M7)                          │
│                           │                                      │
│  ┌────────────────────────▼────────────────────────────────────┐ │
│  │              RPMsg-Lite (MCU side)                          │ │
│  │  rpmsg_lite_create_ept()                                    │ │
│  │  rpmsg_lite_send() / rpmsg_lite_receive()                   │ │
│  └────────────────────────┬────────────────────────────────────┘ │
│                           │                                      │
│  ┌────────────────────────▼────────────────────────────────────┐ │
│  │              Application Tasks (FreeRTOS)                   │ │
│  │  - CAN Task: nhận CAN frame, gửi qua RPMsg                  │ │
│  │  - Audio Task: low-latency audio processing                 │  │
│  │  - SOME/IP Task: xử lý SOME/IP messages                     │ │
│  └─────────────────────────────────────────────────────────────┘ │
└──────────────────────────────────────────────────────────────────┘
```

### 3.2 Shared Memory Layout (Vring)

```
DDR Shared Memory Region (ví dụ: 0xB8000000 - 0xBC000000 = 64MB)
┌─────────────────────────────────────────────────────────────┐
│  Resource Table (M7 firmware chứa bảng này)                 │
│  - Mô tả vring addresses, sizes                             │
│  - Offset: 0x0                                              │
├─────────────────────────────────────────────────────────────┤
│  Vring 0 — TX (A55 → M7)                                    │
│  ┌──────────────────────────────────────────────────────┐   │
│  │  Vring Descriptor Ring (256 entries)                  │   │
│  │  ┌──────┬──────┬──────┬──────┐                        │   │
│  │  │ buf@ │ len  │flags │ next │ × 256                  │   │
│  │  └──────┴──────┴──────┴──────┘                        │   │
│  ├──────────────────────────────────────────────────────┤   │
│  │  Available Ring (A55 writes)                          │   │
│  ├──────────────────────────────────────────────────────┤   │
│  │  Used Ring (M7 writes)                                │   │
│  └──────────────────────────────────────────────────────┘   │
├─────────────────────────────────────────────────────────────┤
│  Vring 1 — RX (M7 → A55)                                    │
│  (cấu trúc tương tự Vring 0)                                │
├─────────────────────────────────────────────────────────────┤
│  Buffer Pool (TX + RX buffers)                              │
│  512 buffers × 512 bytes = 256KB                            │
└─────────────────────────────────────────────────────────────┘
```

### 3.3 Device Tree — Remoteproc & MU

```dts
/* arch/arm64/boot/dts/freescale/imx95-evk.dts */

/* Messaging Unit — Hardware IPC trigger */
mu1: mailbox@44220000 {
    compatible = "fsl,imx95-mu";
    reg = <0x0 0x44220000 0x0 0x10000>;
    interrupts = <GIC_SPI 234 IRQ_TYPE_LEVEL_HIGH>;
    clocks = <&scmi_clk IMX95_CLK_MU1_A>;
    #mbox-cells = <2>;
};

/* Remoteproc node cho Cortex-M7 */
imx95_cm7: cm7@0 {
    compatible = "fsl,imx95-cm7";
    clocks = <&scmi_clk IMX95_CLK_M7>;

    /* Dùng MU để gửi/nhận kick (notify vring) */
    mboxes = <&mu1 0 0>,    /* TX channel */
             <&mu1 1 0>;    /* RX channel */
    mbox-names = "tx", "rx";

    /* Memory regions */
    memory-region = <&vdev0vring0>,   /* Vring TX */
                    <&vdev0vring1>,   /* Vring RX */
                    <&vdev0buffer>,   /* Shared buffers */
                    <&rsc_table>;     /* Resource table */

    /* Firmware path (trong /vendor/firmware/) */
    firmware-name = "imx95_m7.elf";
};

/* Reserved memory cho M7 firmware + shared mem */
reserved-memory {
    #address-cells = <2>;
    #size-cells = <2>;
    ranges;

    /* M7 ITCM/DTCM hoặc OCRAM */
    m7_reserved: m7@80000000 {
        reg = <0 0x80000000 0 0x1000000>;  /* 16MB */
        no-map;
    };

    /* Vring TX (A55 → M7) */
    vdev0vring0: vring0@b8000000 {
        reg = <0 0xb8000000 0 0x8000>;     /* 32KB */
        no-map;
    };

    /* Vring RX (M7 → A55) */
    vdev0vring1: vring1@b8008000 {
        reg = <0 0xb8008000 0 0x8000>;     /* 32KB */
        no-map;
    };

    /* Shared buffers */
    vdev0buffer: buffer@b8400000 {
        compatible = "shared-dma-pool";
        reg = <0 0xb8400000 0 0x100000>;   /* 1MB */
        no-map;
    };

    /* Resource table (M7 firmware ghi vào đây) */
    rsc_table: rsc_table@b8000000 {
        reg = <0 0xb8000000 0 0x1000>;
        no-map;
    };
};
```

### 3.4 Linux Kernel — Remoteproc Driver

```c
/*
 * File: drivers/remoteproc/imx_rproc.c (simplified)
 * Quản lý lifecycle của M7 firmware từ Linux
 */

struct imx_rproc {
    struct rproc        *rproc;
    struct mbox_chan    *tx_ch;     /* MU TX channel */
    struct mbox_chan    *rx_ch;     /* MU RX channel */
    void __iomem        *rsc_table; /* resource table mapped addr */
};

/* Platform driver ops */
static const struct rproc_ops imx_rproc_ops = {
    .prepare        = imx_rproc_prepare,
    .start          = imx_rproc_start,       /* boot M7 */
    .stop           = imx_rproc_stop,        /* shutdown M7 */
    .kick           = imx_rproc_kick,        /* notify M7 có data */
    .load           = imx_rproc_elf_load,    /* load .elf */
    .get_boot_addr  = imx_rproc_get_boot_addr,
};

/* Boot M7 firmware */
static int imx_rproc_start(struct rproc *rproc)
{
    struct imx_rproc *priv = rproc->priv;

    /* Release M7 khỏi reset */
    regmap_update_bits(priv->regmap, SRC_M7_RCR,
                       M7_CORE_RESET, 0);  /* deassert reset */

    dev_info(&rproc->dev, "M7 core started\n");
    return 0;
}

/* Kick M7 khi có data trong vring */
static void imx_rproc_kick(struct rproc *rproc, int vqid)
{
    struct imx_rproc *priv = rproc->priv;
    u32 val = vqid;

    /* Gửi interrupt tới M7 qua MU */
    mbox_send_message(priv->tx_ch, &val);
}

/* MU RX interrupt handler — M7 kick A55 */
static void imx_rproc_rx_callback(struct mbox_client *cl, void *data)
{
    struct imx_rproc *priv = container_of(cl, struct imx_rproc, rx_cl);
    u32 vqid = *(u32 *)data;

    /* Notify remoteproc framework có data từ M7 */
    rproc_vq_interrupt(priv->rproc, vqid);
}
```

### 3.5 Linux Kernel — RPMsg Driver

```c
/*
 * File: drivers/rpmsg/virtio_rpmsg_bus.c (simplified)
 * RPMsg bus layer — abstract hóa vring thành message API
 */

/*
 * Gửi message tới M7
 * src_ept: endpoint trên A55 (channel source)
 * dst: endpoint trên M7 (channel destination)
 */
int rpmsg_send(struct rpmsg_endpoint *ept, void *data, int len)
{
    struct virtproc_info *vrp = ept->rpdev->vrp;
    struct scatterlist sg;
    void *msg_buf;
    int err;

    /* Lấy buffer từ pool */
    msg_buf = rpmsg_get_tx_buf(vrp);
    if (!msg_buf)
        return -ENOMEM;

    /* Copy data vào buffer */
    memcpy(msg_buf, data, len);

    /* Thêm vào vring descriptor ring */
    sg_init_one(&sg, msg_buf, len);
    err = virtqueue_add_outbuf(vrp->svq, &sg, 1, msg_buf, GFP_KERNEL);

    /* Kick M7 qua MU interrupt */
    virtqueue_kick(vrp->svq);

    return err;
}

/*
 * Đăng ký callback nhận message từ M7
 */
struct rpmsg_endpoint *rpmsg_create_ept(struct rpmsg_device *rpdev,
                                         rpmsg_rx_cb_t cb,
                                         void *priv,
                                         struct rpmsg_channel_info chinfo)
{
    struct virtproc_info *vrp = rpdev->vrp;
    struct rpmsg_endpoint *ept;

    ept = kzalloc(sizeof(*ept), GFP_KERNEL);
    ept->rpdev  = rpdev;
    ept->cb     = cb;      /* callback khi có message đến */
    ept->priv   = priv;
    ept->addr   = chinfo.src;

    /* Register endpoint, keyed by address */
    idr_alloc(&vrp->endpoints, ept, ept->addr, ept->addr + 1, GFP_KERNEL);

    return ept;
}
```

### 3.6 RPMsg Character Device — Userspace Interface

```c
/*
 * File: drivers/rpmsg/rpmsg_char.c
 * Tạo /dev/rpmsg0, /dev/rpmsg1... để userspace dùng
 */

static ssize_t rpmsg_dev_write(struct file *filp,
                                const char __user *buf,
                                size_t len, loff_t *f_pos)
{
    struct rpmsg_char_dev *rcdev = filp->private_data;
    char kbuf[512];

    if (copy_from_user(kbuf, buf, len))
        return -EFAULT;

    /* Gửi qua RPMsg → vring → M7 */
    return rpmsg_send(rcdev->ept, kbuf, len);
}

static ssize_t rpmsg_dev_read(struct file *filp,
                               char __user *buf,
                               size_t len, loff_t *f_pos)
{
    struct rpmsg_char_dev *rcdev = filp->private_data;
    struct sk_buff *skb;

    /* Block cho đến khi có message từ M7 */
    wait_event_interruptible(rcdev->readq,
                             !skb_queue_empty(&rcdev->queue));

    skb = skb_dequeue(&rcdev->queue);
    if (copy_to_user(buf, skb->data, skb->len))
        return -EFAULT;

    return skb->len;
}
```

### 3.7 M7 Firmware — FreeRTOS + RPMsg-Lite

```c
/*
 * File: m7_firmware/main.c (FreeRTOS trên Cortex-M7)
 * Sử dụng RPMsg-Lite — lightweight version cho MCU
 */

#include "rpmsg_lite.h"
#include "rpmsg_queue.h"
#include "rpmsg_ns.h"

/* Shared memory address (phải match với Linux DTS) */
#define RPMSG_LITE_SHMEM_BASE   0xB8000000
#define RPMSG_LITE_LINK_ID      RL_PLATFORM_IMX95_M7_USER_LINK_ID

struct rpmsg_lite_instance *rpmsg_inst;
struct rpmsg_lite_endpoint *my_ept;
rpmsg_queue_handle          my_queue;

/* Task: xử lý message từ A55 */
void rpmsg_task(void *param)
{
    uint8_t recv_buf[512];
    uint32_t recv_len;
    uint32_t src_addr;
    int32_t  result;

    /* Khởi tạo RPMsg-Lite trên M7 side */
    rpmsg_inst = rpmsg_lite_remote_init(
        (void *)RPMSG_LITE_SHMEM_BASE,
        RPMSG_LITE_LINK_ID,
        RL_NO_FLAGS
    );

    /* Chờ A55 (master) sẵn sàng */
    while (!rpmsg_lite_is_link_up(rpmsg_inst))
        vTaskDelay(pdMS_TO_TICKS(10));

    /* Tạo queue và endpoint */
    my_queue = rpmsg_queue_create(rpmsg_inst);
    my_ept   = rpmsg_lite_create_ept(
        rpmsg_inst,
        LOCAL_EPT_ADDR,      /* địa chỉ endpoint M7, e.g. 30 */
        rpmsg_queue_rx_cb,   /* callback vào queue */
        my_queue
    );

    /* Announce endpoint tới A55 (name service) */
    rpmsg_ns_announce(rpmsg_inst, my_ept,
                      "rpmsg-audio-ch",    /* channel name */
                      RL_NS_CREATE);

    /* Main loop — xử lý message */
    while (1) {
        /* Block nhận message từ A55 */
        result = rpmsg_queue_recv(
            rpmsg_inst, my_queue,
            &src_addr,
            (char *)recv_buf, sizeof(recv_buf),
            &recv_len,
            RL_BLOCK
        );

        if (result == RL_SUCCESS) {
            /* Xử lý command từ A55 */
            process_command(recv_buf, recv_len);

            /* Gửi response về A55 */
            rpmsg_lite_send(
                rpmsg_inst, my_ept,
                src_addr,              /* gửi về source của A55 */
                (char *)response_buf,
                response_len,
                RL_BLOCK
            );
        }
    }
}
```

### 3.8 Resource Table — "Hợp đồng" giữa M7 và Linux

Resource Table là cấu trúc dữ liệu trong M7 firmware ELF, Linux đọc để biết cần setup vring ở đâu.

```c
/*
 * File: m7_firmware/resource_table.c
 * PHẢI có section ".resource_table" trong linker script
 */

#include "rsc_table.h"

/* Số lượng entries trong resource table */
#define NO_RESOURCE_ENTRIES     2

struct remote_resource_table {
    struct resource_table   base;
    uint32_t                offset[NO_RESOURCE_ENTRIES];

    /* Entry 0: VirtIO device (vring pairs) */
    struct fw_rsc_vdev      vdev;
    struct fw_rsc_vdev_vring vring[2];    /* TX + RX */

    /* Entry 1: Carveout memory */
    struct fw_rsc_carveout  carveout;
} __attribute__((packed));

/* Đặt vào section .resource_table — Linux ELF parser tìm ở đây */
__attribute__((section(".resource_table")))
const struct remote_resource_table resources = {
    .base = {
        .ver        = 1,
        .num        = NO_RESOURCE_ENTRIES,
        .reserved   = {0, 0},
    },
    .offset = {
        offsetof(struct remote_resource_table, vdev),
        offsetof(struct remote_resource_table, carveout),
    },

    /* VirtIO RPMsg device */
    .vdev = {
        .type       = RSC_VDEV,
        .id         = VIRTIO_ID_RPMSG,    /* 7 */
        .notifyid   = 0,
        .dfeatures  = 1,                  /* VIRTIO_RPMSG_F_NS */
        .config_len = 0,
        .num_of_vrings = 2,
    },

    /* Vring 0: A55 → M7 */
    .vring[0] = {
        .da     = 0xB8000000,             /* phải match DTS */
        .align  = 0x10,
        .num    = 256,                    /* số descriptor */
        .notifyid = 0,
    },

    /* Vring 1: M7 → A55 */
    .vring[1] = {
        .da     = 0xB8008000,             /* phải match DTS */
        .align  = 0x10,
        .num    = 256,
        .notifyid = 1,
    },
};
```

### 3.9 Android Userspace — HAL tương tác với M7

```cpp
/*
 * File: hardware/imx/rpmsg_hal/RpmsgHal.cpp
 * AIDL HAL Service để Android app giao tiếp với M7 qua /dev/rpmsg0
 */

#include <fcntl.h>
#include <unistd.h>

class RpmsgHal : public BnRpmsgHal {
public:
    int mFd = -1;

    /* Mở rpmsg channel khi HAL init */
    bool openChannel(const std::string& channelName) {
        /* Tìm /dev/rpmsg* theo tên channel */
        /* Kernel tạo /dev/rpmsg0 khi M7 announce endpoint */
        mFd = open("/dev/rpmsg0", O_RDWR);
        if (mFd < 0) {
            ALOGE("Failed to open rpmsg channel: %s", strerror(errno));
            return false;
        }
        return true;
    }

    /* Gửi command xuống M7 */
    ScopedAStatus sendCommand(const std::vector<uint8_t>& cmd,
                               std::vector<uint8_t>* response) override {
        /* Write → rpmsg_char → vring → M7 */
        ssize_t written = write(mFd, cmd.data(), cmd.size());
        if (written < 0) {
            return ScopedAStatus::fromExceptionCode(EX_ILLEGAL_STATE);
        }

        /* Read response từ M7 */
        uint8_t resp_buf[512];
        ssize_t recv_len = read(mFd, resp_buf, sizeof(resp_buf));
        if (recv_len > 0) {
            response->assign(resp_buf, resp_buf + recv_len);
        }

        return ScopedAStatus::ok();
    }
};
```

### 3.10 Practical — Load và control M7 firmware từ Android

```bash
# === Userspace control M7 firmware via sysfs ===

# Xem trạng thái M7
adb shell cat /sys/class/remoteproc/remoteproc0/state
# Output: offline

# Load firmware (file phải có ở /vendor/firmware/)
adb shell echo "imx95_m7.elf" > /sys/class/remoteproc/remoteproc0/firmware
adb shell echo "start"        > /sys/class/remoteproc/remoteproc0/state

# Kiểm tra đã boot thành công
adb shell cat /sys/class/remoteproc/remoteproc0/state
# Output: running

# Kiểm tra rpmsg channels được tạo
adb shell ls /dev/rpmsg*
# Output: /dev/rpmsg0  /dev/rpmsg1

# Test gửi/nhận thủ công
adb shell "echo 'HELLO_M7' > /dev/rpmsg0"
adb shell "cat /dev/rpmsg0"  # đọc response từ M7

# Xem log remoteproc
adb shell dmesg | grep -i "remoteproc\|rpmsg\|virtio"

# Stop M7
adb shell echo "stop" > /sys/class/remoteproc/remoteproc0/state
```

### 3.11 Debug RPMsg — Quan trọng

```bash
# === Debug vring stats ===
adb shell cat /sys/kernel/debug/remoteproc/remoteproc0/vring0  # TX stats
adb shell cat /sys/kernel/debug/remoteproc/remoteproc0/vring1  # RX stats

# === Trace RPMsg messages (kernel ftrace) ===
adb shell "echo 1 > /sys/kernel/debug/tracing/events/rpmsg/enable"
adb shell "cat /sys/kernel/debug/tracing/trace_pipe"
# Output sẽ show từng message gửi/nhận

# === Check MU interrupt ===
adb shell cat /proc/interrupts | grep mu
# Output: 234:  0  0  0  0  GIC  234  fsl,mu1

# === Kernel log khi M7 crash (watchdog) ===
adb shell dmesg | grep -i "rproc\|crash\|coredump"
# [  123.456789] remoteproc0: crash detected!
# [  123.456800] remoteproc0: collecting coredump
# Coredump sẽ ở /sys/class/remoteproc/remoteproc0/coredump

# === Memory mapping check ===
adb shell cat /proc/iomem | grep -i "b8"
# b8000000-b8007fff : vdev0vring0
# b8008000-b800ffff : vdev0vring1
# b8400000-b84fffff : vdev0buffer
```

### 3.12 Use Case Audio — M7 làm Audio DSP offload

```
Scenario: Low-latency audio processing trên M7, Android chỉ nhận PCM final

Linux/Android:
  AudioRecord → AudioFlinger → HAL
      └── write PCM raw qua /dev/rpmsg_audio → M7

M7 (FreeRTOS):
  Nhận PCM raw
  → Apply EQ / ANC (Active Noise Cancellation)
  → Apply Volume Ramp
  → Gửi processed PCM về A55 qua RPMsg
  → hoặc output trực tiếp ra SAI/I2S (hardware route)
```

```c
/* M7 Audio processing task */
void audio_rpmsg_task(void *param)
{
    int16_t  raw_pcm[FRAME_SIZE];     /* nhận từ A55 */
    int16_t  proc_pcm[FRAME_SIZE];    /* sau xử lý */
    uint32_t len, src;

    while (1) {
        /* Nhận PCM frame từ A55 (Linux AudioFlinger) */
        rpmsg_queue_recv(rpmsg_inst, audio_queue,
                         &src, (char *)raw_pcm,
                         sizeof(raw_pcm), &len, RL_BLOCK);

        /* Xử lý DSP — chạy trên M7 không ảnh hưởng A55 load */
        apply_eq(raw_pcm, proc_pcm, FRAME_SIZE);
        apply_anc(proc_pcm, FRAME_SIZE);
        apply_volume_ramp(proc_pcm, FRAME_SIZE, current_volume);

        /* Option 1: Gửi processed PCM về A55 */
        rpmsg_lite_send(rpmsg_inst, audio_ept,
                        src, (char *)proc_pcm,
                        sizeof(proc_pcm), RL_BLOCK);

        /* Option 2: Output trực tiếp qua SAI (DMA hardware) */
        SAI_TransferSendEDMA(AUDIO_SAI, &sai_handle,
                             proc_pcm, FRAME_SIZE);
    }
}
```

---

## Practical Lab

### Lab 1 — Verify Remoteproc setup

```bash
# Kiểm tra device tree đã có remoteproc chưa
adb shell ls /sys/class/remoteproc/
# Nếu trống → chưa enable CONFIG_REMOTEPROC trong kernel config

# Kiểm tra kernel config
adb shell zcat /proc/config.gz | grep -i remoteproc
# CONFIG_REMOTEPROC=y
# CONFIG_IMX_REMOTEPROC=y
# CONFIG_RPMSG_CHAR=y
# CONFIG_RPMSG_VIRTIO=y
```

### Lab 2 — Build M7 firmware đơn giản (echo server)

```c
/* Firmware tối giản: nhận message, echo lại kèm "M7_REPLY:" */
void echo_task(void *param)
{
    char buf[256];
    uint32_t len, src;

    rpmsg_inst = rpmsg_lite_remote_init(...);
    while (!rpmsg_lite_is_link_up(rpmsg_inst));

    my_queue = rpmsg_queue_create(rpmsg_inst);
    my_ept   = rpmsg_lite_create_ept(rpmsg_inst, 30,
                                      rpmsg_queue_rx_cb, my_queue);
    rpmsg_ns_announce(rpmsg_inst, my_ept, "rpmsg-echo", RL_NS_CREATE);

    while (1) {
        rpmsg_queue_recv(rpmsg_inst, my_queue,
                         &src, buf, sizeof(buf), &len, RL_BLOCK);

        /* Prepend "M7_REPLY:" */
        char reply[256];
        snprintf(reply, sizeof(reply), "M7_REPLY: %.*s", len, buf);
        rpmsg_lite_send(rpmsg_inst, my_ept, src,
                        reply, strlen(reply), RL_BLOCK);
    }
}
```

```bash
# Test echo server
adb shell echo "start" > /sys/class/remoteproc/remoteproc0/state
adb shell "echo 'PING' > /dev/rpmsg0 && cat /dev/rpmsg0"
# Expected: M7_REPLY: PING
```

### Lab 3 — CAN Bridge qua RPMsg (Real automotive use case)

```
CAN Bus (vật lý)
    │
M7 (CAN controller driver, SocketCAN-like)
    │ RPMsg
A55 (Android)
    │ /dev/rpmsg_can → SocketCAN vcan interface
    │
Android App (SOME/IP, UDS Diagnostic...)
```

---

---

## Chương 0 — Boot Timeline Đầy Đủ: Power ON → Android Running

> Đây là "bản đồ" tổng thể. Đọc chương này trước để có định hướng, các chương sau đào sâu từng giai đoạn.

### 0.1 Timeline hoàn chỉnh trên EVK95

```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
 t=0ms   POWER ON / RESET
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
         Core đầu tiên thức dậy: Cortex-M33 (System Manager)
         Lý do: BootROM i.MX95 boot M33 TRƯỚC A55

 [M33]   BootROM on-chip (read-only, NXP code)
         → Scan boot source: eMMC/SD/QSPI/USB (via BOOT_MODE pins)
         → Load + verify System Manager firmware vào OCRAM
         → Jump to SM entry point

 [M33]   System Manager firmware starts (EL1-S, secure world)
         → Khởi tạo ELE (EdgeLock): secure key storage, TRNG
         → Khởi tạo PLLs cơ bản (system PLL, DRAM PLL)
         → Khởi tạo DDR controller + DDRPHY training
              (DDR training mất 50-200ms tùy LPDDR5/DDR5 config)
         → Khởi tạo power domains cơ bản
         → Start SCMI server — sẵn sàng nhận request từ A55
         → Release A55 Core0 khỏi reset

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
 t≈200ms  A55 Core0 thức dậy (M33 vẫn chạy song song)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
 [A55]   BootROM A55 on-chip (EL3)
         → Đọc IVT (Image Vector Table) từ eMMC boot partition
            offset 0x400 (SD) hoặc 0x8400 (eMMC boot partition)
         → IVT chứa: địa chỉ SPL, DCD, Boot Data
         → Load SPL vào OCRAM (DDR chưa accessible từ A55 BootROM)
         → Verify SPL signature nếu AHAB/HAB enabled (secure boot)
         → Jump to SPL

 [A55/EL3]  SPL chạy (trong OCRAM ~256KB)
         → Init UART → console output: "U-Boot SPL 2024.04..."
         → Request clocks cho A55 qua SCMI → M33 xử lý
         → Load ATF BL2, BL31, BL32(OP-TEE opt), BL33(U-Boot) vào DDR
         → Verify chain of trust
         → Jump to ATF BL2

 [A55/EL3]  ATF BL2
         → Setup TZC-400 (TrustZone Controller): secure/non-secure DDR regions
         → Authenticate các BL images
         → Transfer to BL31

 [A55/EL3]  ATF BL31 — Runtime Firmware (LUÔN RESIDENT trong RAM)
         → Init PSCI: đăng ký cpu_on/off/suspend handlers
         → Init SMC dispatcher
         → Setup entry point cho EL2 (U-Boot)
         → Jump to BL32 nếu có OP-TEE, sau đó BL33

 [A55/EL2]  U-Boot chạy
         → "U-Boot 2024.04 (imx95-evk)"
         → Init MMC/eMMC driver
         → Load environment variables
         → Chờ bootdelay (2-3s) — nhấn bất kỳ phím để interrupt
         → Chạy bootcmd:
              load mmc 0:1 ${loadaddr} Image         ← kernel
              load mmc 0:1 ${fdt_addr} imx95-evk.dtb ← DTB
              booti ${loadaddr} - ${fdt_addr}         ← boot!
         → FDT fixup: patch DTB (memory size, serial#, bootargs)
         → Setup ARM64 boot protocol registers:
              x0 = physical address của DTB
              x1 = x2 = x3 = 0 (reserved)
         → Switch sang EL1, MMU off, cache off
         → Jump to kernel entry point

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
 t≈5s    Linux Kernel boot (A55 Core0 only, EL1)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
 [Core0] start_kernel() → setup_arch()
         → Parse DTB: memory map, cpu topology, peripheral nodes
         → Setup MMU + page tables (enable virtual memory)
         → Init interrupt controller GIC-600
         → Init clock framework → gửi SCMI clock requests tới M33
         → Init DMA, IOMMU (SMMU)

 [Core0] smp_init() — đánh thức Core1..3
         → cpu_up(1) → PSCI CPU_ON SMC → ATF EL3 → release Core1
         → Core1 chạy secondary_start_kernel() → join scheduler
         → Tương tự Core2, Core3
         ⚠️  M33 vẫn chạy liên tục, xử lý SCMI requests

 [All cores]  Driver probe cascade
         → MU driver probe: kết nối hardware channels
         → SCMI driver probe (MU2): bắt tay với M33 SCMI server
         → Clock/power domains setup qua SCMI
         → Remoteproc driver probe: sẵn sàng load M7
         → SAI, ASRC, TAS5828, PCM1808 drivers probe
         → Android init process start

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
 t≈25s   Android framework up, M7 firmware loaded
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
 [Android]  init.rc → start vendor services
         → vendor.remoteproc service:
              echo "imx95_m7.elf" > /sys/class/remoteproc/.../firmware
              echo "start"        > /sys/class/remoteproc/.../state

 [Linux]  Remoteproc load M7:
         → Đọc .elf từ /vendor/firmware/
         → Parse resource table → setup vring shared memory
         → Copy firmware vào M7 ITCM/DTCM
         → SCMI → M33 → deassert M7 reset

 [M7]    FreeRTOS starts
         → rpmsg_lite_remote_init() với shared memory address
         → Announce endpoints qua Name Service
         → /dev/rpmsg0 xuất hiện trên Linux side

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
 STEADY STATE: 3 processors chạy song song
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
 A55 (4 cores):   Android 15, EL0/EL1, scheduler
 M33 (1 core):    SCMI server, safety watchdog, clock/power gating
 M7  (1 core):    FreeRTOS, RPMsg endpoints, CAN/audio RT tasks
 ELE (dedicated): Crypto, TRNG, secure lifecycle — always on
 ATF (BL31):      Resident EL3 firmware, handles SMC calls
```

### 0.2 IVT — BootROM tìm gì trong eMMC?

```c
/*
 * Image Vector Table — NXP i.MX95 Reference Manual
 * Tại: eMMC offset 0x400 (SD) hoặc 0x8400 (eMMC boot partition)
 */
struct ivt {
    uint32_t header;        /* 0xD1002041 — magic + version */
    uint32_t entry;         /* Jump address sau khi load (SPL entry) */
    uint32_t reserved1;
    uint32_t dcd;           /* Device Config Data — khởi tạo DDR sơ bộ */
    uint32_t boot_data;     /* Pointer tới boot_data struct */
    uint32_t self;          /* Địa chỉ của chính IVT (self-referential) */
    uint32_t csf;           /* Command Sequence File — HAB/AHAB signature */
    uint32_t reserved2;
};

struct boot_data {
    uint32_t start;   /* Load address trong OCRAM */
    uint32_t size;    /* Size của SPL image */
    uint32_t plugin;  /* 0 = normal boot */
};
```

### 0.3 U-Boot → Kernel handoff (ARM64 boot protocol)

```bash
# U-Boot booti command thực hiện:
# 1. Verify ARM64 Image magic (offset 0x38 = 0x644D5241)
# 2. FDT fixup — patch DTB với runtime info:
#    /memory { reg = <0 0x80000000 0 0x80000000>; }  (actual RAM)
#    /chosen { bootargs = "console=ttymxc1,115200 ..."; }
#    /chosen { linux,initrd-start/end = ...; }  nếu có ramdisk
# 3. Set registers:
#    x0 = FDT physical address
#    x1 = x2 = x3 = 0
# 4. Disable MMU, disable caches
# 5. Jump to Image + text_offset

# Xem bootargs thực tế trên EVK95
adb shell cat /proc/cmdline
```

---

## Chương 0B — MU (Messaging Unit) Hardware — Cơ chế vật lý

> MU là trái tim của mọi giao tiếp inter-core trên i.MX95. Hiểu MU = hiểu tại sao RPMsg và SCMI hoạt động được.

### 0B.1 MU hardware layout

MU là hardware IP block với **4 TX + 4 RX registers** (mỗi cái 32-bit), hoàn toàn symmetric:

```
┌─────────────────────────────────────────────────────────────────┐
│                   MU Hardware Block (MU1)                       │
│                                                                 │
│   Side A (Linux/A55)            Side B (M7/FreeRTOS)           │
│   ┌────────────────────┐        ┌────────────────────┐         │
│   │  MU_TR0..3 (TX)    │──────► │  MU_RR0..3 (RX)   │         │
│   │  write → send      │        │  read ← receive   │         │
│   └────────────────────┘        └────────────────────┘         │
│   ┌────────────────────┐        ┌────────────────────┐         │
│   │  MU_RR0..3 (RX)   │◄────── │  MU_TR0..3 (TX)   │         │
│   │  read ← receive   │        │  write → send      │         │
│   └────────────────────┘        └────────────────────┘         │
│                                                                 │
│   Status Register (MU_SR):                                      │
│   Bit 20-23: TEn[0..3] — TX Empty (channel free to send)        │
│   Bit 24-27: RFn[0..3] — RX Full (data arrived)                 │
│   Bit 28-31: GIPn[0..3] — General Interrupt Pending             │
└─────────────────────────────────────────────────────────────────┘
```

**Quan trọng:** MU chỉ truyền **32-bit word** mỗi lần. Data thực sự đi qua shared DDR — MU chỉ dùng để **báo hiệu (kick)**.

### 0B.2 MU Register Map

```c
/* MU Register offsets — i.MX95 Reference Manual */
#define MU_VER      0x000   /* Version */
#define MU_CR       0x008   /* Control */
#define MU_SR       0x00C   /* Status  */
#define MU_TR(n)    (0x200 + (n)*4)  /* TX Register 0..3 (Side A) */
#define MU_RR(n)    (0x280 + (n)*4)  /* RX Register 0..3 (Side A) */

/* Status bits */
#define MU_SR_TEn(n)  BIT(20 + (n))  /* TX Empty: có thể gửi */
#define MU_SR_RFn(n)  BIT(24 + (n))  /* RX Full: có data đến */
```

### 0B.3 Cơ chế gửi/nhận step-by-step

```c
/* === LINUX SIDE (A55) gửi kick tới M7 === */
/* File: drivers/mailbox/imx-mailbox.c */
static int imx_mu_send_data(struct mbox_chan *chan, void *data)
{
    u32 val = *(u32 *)data;       /* 1 word 32-bit = vring index */
    u32 ch  = chan->con_priv;     /* channel 0..3 */

    /* Chờ TX channel free (TEn=1 = empty = có thể ghi) */
    if (!(readl(base + MU_SR) & MU_SR_TEn(ch)))
        return -EBUSY;

    /* Ghi vào TX register → hardware tự copy sang RX của M7 */
    writel(val, base + MU_TR(ch));

    /* Hardware tự động:
     *  → Set MU_SR.TEn = 0 (TX đang có data)
     *  → Set MU_SR.RFn = 1 ở phía M7
     *  → Trigger interrupt tới M7 nếu RIE enabled
     */
    return 0;
}
```

```c
/* === M7 SIDE (FreeRTOS) nhận interrupt === */
void MU1_B_IRQHandler(void)
{
    uint32_t flags = MU_GetStatusFlags(MU1_B);

    if (flags & kMU_RxFullFlag0) {
        /* Đọc word từ RX register (clear interrupt tự động) */
        uint32_t vring_idx = MU_ReceiveMsgNonBlocking(MU1_B, 0);

        /* Notify RPMsg-Lite: có data trong vring[vring_idx] */
        rpmsg_lite_rx_callback(vring_idx);
    }
}
```

### 0B.4 Tại sao không chỉ dùng shared memory polling?

```
Polling (không dùng MU):                  Interrupt-driven (dùng MU):
  M7 liên tục đọc flag trong DDR            M7 ngủ (WFI) cho đến khi
  → CPU 100% busy ngay cả khi idle           có MU interrupt
  → Không thể sleep → pin power cao          → CPU idle = power save
  → DDR bus contention                       → Latency: ~1μs interrupt
  → Worst-case latency không xác định        → Deterministic RT behavior
```

### 0B.5 Nhiều MU instances trên i.MX95

```
MU1 (0x44220000): A55 ↔ M7   — RPMsg data channel
MU2 (0x44230000): A55 ↔ M33  — SCMI protocol (clock/power requests)
MU3 (0x44240000): A55 ↔ M33  — SCMI notifications (async events)
MU5 (0x44260000): A55 ↔ ELE  — EdgeLock security requests
MU6 (0x44270000): M7  ↔ M33  — direct M7-SM channel (nếu cần)

GIC interrupt lines:
  MU1 → SPI 234
  MU2 → SPI 235
  MU5 → SPI 238
```

---

## Chương 0C — Virtio Layer: Cầu nối giữa Remoteproc và RPMsg

> Layer này thường bị bỏ qua nhưng là chìa khóa để hiểu tại sao toàn bộ cơ chế hoạt động.

### 0C.1 Stack 4 tầng đầy đủ

```
┌───────────────────────────────────────────────────────────────┐
│  Tầng 4: RPMsg Bus & Endpoints                                │
│  rpmsg_create_ept(), rpmsg_send()  →  /dev/rpmsg0             │
├───────────────────────────────────────────────────────────────┤
│  Tầng 3: Virtio RPMsg Driver (virtio_rpmsg_bus.c)             │
│  - Tạo rpmsg_device (mỗi channel = 1 device)                 │
│  - Xử lý Name Service announcements từ M7                    │
│  - Manage endpoint address space                              │
├───────────────────────────────────────────────────────────────┤
│  Tầng 2: Virtio Core + Virtqueue (virtio_ring.c)              │
│  - Virtqueue = abstraction của 1 vring                        │
│  - virtqueue_add_buf() / virtqueue_get_buf()                  │
│  - virtqueue_kick() → gọi transport layer kick               │
├───────────────────────────────────────────────────────────────┤
│  Tầng 1: Remoteproc Virtio Transport (remoteproc_virtio.c)    │
│  - Tạo virtio_device từ resource table                        │
│  - kick() → MU send (1 word = vring index)                   │
│  - Cung cấp vring memory từ reserved-memory DTS              │
└───────────────────────────────────────────────────────────────┘
```

### 0C.2 Lifecycle đầy đủ: firmware load → endpoint ready

```
echo "start" > /sys/class/remoteproc/remoteproc0/state
    │
    ▼  rproc_boot()
    │
    ├─ 1. LOAD ELF
    │     rproc_elf_load_segments()
    │     Copy .text/.data/.resource_table vào M7 memory regions
    │
    ├─ 2. PARSE RESOURCE TABLE
    │     Tìm RSC_VDEV entry (type=7, VIRTIO_ID_RPMSG)
    │     Đọc: vring[0].da = 0xB8000000 (TX, 256 descs)
    │           vring[1].da = 0xB8008000 (RX, 256 descs)
    │     Map addresses (non-cacheable) trong Linux page tables
    │
    ├─ 3. TẠO VIRTIO DEVICE
    │     rproc_add_virtio_devices()
    │     → struct virtio_device { id.device = VIRTIO_ID_RPMSG=7 }
    │     → Register trên virtio bus
    │
    ├─ 4. VIRTIO RPMSG DRIVER PROBE
    │     (bus match: device_id=7 → virtio_rpmsg_bus.c)
    │     rpmsg_probe():
    │     → Tạo virtqueue TX (svq) ↔ vring[0]
    │     → Tạo virtqueue RX (rvq) ↔ vring[1]
    │     → Pre-fill RX vring với empty buffers (sẵn sàng nhận)
    │
    ├─ 5. RELEASE M7
    │     imx_rproc_start() → SCMI → M33 → deassert M7 reset pin
    │
    ▼  [M7 FreeRTOS starts]
    │
    ├─ 6. M7: rpmsg_lite_remote_init()
    │     Đọc resource table (Linux đã điền địa chỉ vào)
    │     Init vring TX/RX với cùng shared memory addresses
    │     Gửi "alive" kick qua MU → Linux
    │
    ├─ 7. LINK UP
    │     Linux nhận MU interrupt → virtio device = DRIVER_OK
    │
    ├─ 8. M7: NAME SERVICE ANNOUNCE
    │     rpmsg_ns_announce(inst, ept, "rpmsg-audio-ch", CREATE)
    │     → Gửi NS message qua vring TX
    │     → MU kick → Linux interrupt
    │
    ├─ 9. LINUX: NAME SERVICE CALLBACK
    │     rpmsg_ns_cb(): name = "rpmsg-audio-ch", addr = 30
    │     → Match với registered rpmsg driver hoặc rpmsg_char
    │     → Tạo rpmsg_device, probe rpmsg_char driver
    │     → mknod /dev/rpmsg0
    │
    └─ 10. READY
          Android HAL open("/dev/rpmsg0") → giao tiếp với M7
```

### 0C.3 Vring — Life of 1 message (A55 → M7)

```
write(fd, "HELLO", 5)
    │
    ▼ rpmsg_char_write → rpmsg_send
    │
    ├─ Lấy TX buffer từ pool (512 bytes pre-allocated)
    │
    ├─ Điền RPMsg header:
    │   [src=0x400][dst=30][reserved][len=5][flags=0]["HELLO"]
    │
    ├─ Add vào vring TX avail ring:
    │   desc[N].addr = phys_addr_of_buf
    │   desc[N].len  = sizeof(hdr) + 5
    │   avail->ring[avail->idx] = N
    │   avail->idx++              ← atomic write
    │
    ├─ virtqueue_kick() → MU_TR(0) = 0  ← 1 word, 1 CPU cycle
    │
    ▼ [M7 MU interrupt, ~1μs later]
    │
    ├─ M7 đọc vring avail ring: thấy desc[N] mới
    ├─ Dereference desc[N].addr → đọc buffer từ DDR
    ├─ Parse RPMsg header: dst=30 → dispatch tới endpoint 30
    ├─ endpoint_30_cb("HELLO", 5)  ← application callback
    │
    └─ M7 add desc[N] vào used ring → kick A55 để recycle buffer
```

---

## Chương 4 — SCMI (System Control & Management Interface)

### 4.1 SCMI là gì và tại sao quan trọng?

Trên i.MX95, Linux kernel **không được phép** trực tiếp thao tác clock/power registers của nhiều peripheral. Thay vào đó, kernel gửi request qua **SCMI** tới **System Manager (SM)** — một firmware chạy trên Cortex-M33, có toàn quyền với hardware.

```
┌─────────────────────────────────────────────────────────┐
│                 Linux (Cortex-A55, EL1)                  │
│                                                         │
│   clk_set_rate(sai3_clk, 49152000)                      │
│         │                                               │
│   Common Clock Framework                                │
│         │                                               │
│   imx95-scmi-clk driver                                 │
│         │  SCMI_CLOCK_RATE_SET message                  │
└─────────┼───────────────────────────────────────────────┘
          │ MU (Messaging Unit) — shared memory channel
┌─────────┼───────────────────────────────────────────────┐
│         ▼                                               │
│   System Manager (Cortex-M33, EL1-S)                   │
│   - Nhận SCMI message                                   │
│   - Validate (có được phép không?)                      │
│   - Thao tác CCM/BLK_CTRL registers trực tiếp           │
│   - Reply SUCCESS/ERROR về Linux                        │
└─────────────────────────────────────────────────────────┘
```

**Tại sao thiết kế vậy?** — Safety & Security partitioning. SM là "gatekeeper" duy nhất có thể thay đổi clock/reset/power của toàn SoC, tránh Linux vô tình corrupt hardware state.

### 4.2 SCMI Protocol Stack

```
Linux SCMI stack (drivers/firmware/arm_scmi/):

  Consumer driver (e.g. imx95-scmi-clk)
       │
  SCMI Protocol layer
  ├── scmi_clock_protocol    (PROTOCOL_ID = 0x14)
  ├── scmi_power_protocol    (PROTOCOL_ID = 0x11)
  ├── scmi_perf_protocol     (PROTOCOL_ID = 0x13)  ← CPU DVFS
  ├── scmi_sensor_protocol   (PROTOCOL_ID = 0x15)
  └── scmi_pinctrl_protocol  (PROTOCOL_ID = 0x19)  ← mới trên i.MX95
       │
  SCMI Transport layer
  └── mailbox transport (dùng MU hardware)
       │
  ARM MU driver (drivers/mailbox/imx-mailbox.c)
       │ shared memory + interrupt
  System Manager firmware (M33)
```

### 4.3 Device Tree — SCMI trên EVK95

```dts
/* arch/arm64/boot/dts/freescale/imx95.dtsi */

/* MU dùng cho SCMI channel */
mu2: mailbox@44230000 {
    compatible = "fsl,imx95-mu";
    reg = <0x0 0x44230000 0x0 0x10000>;
    interrupts = <GIC_SPI 235 IRQ_TYPE_LEVEL_HIGH>;
    clocks = <&scmi_clk IMX95_CLK_MU2_A>;
    #mbox-cells = <2>;
};

/* SCMI firmware interface */
firmware {
    scmi: scmi {
        compatible = "arm,scmi";
        mboxes = <&mu2 0 0>,   /* TX */
                 <&mu2 1 0>;   /* RX */
        mbox-names = "tx", "rx";
        shmem = <&scmi_buf>;   /* shared memory buffer */
        #address-cells = <1>;
        #size-cells = <0>;

        /* Clock protocol */
        scmi_clk: protocol@14 {
            reg = <0x14>;
            #clock-cells = <1>;
        };

        /* Power domain protocol */
        scmi_devpd: protocol@11 {
            reg = <0x11>;
            #power-domain-cells = <1>;
        };

        /* Performance (DVFS) protocol */
        scmi_perf: protocol@13 {
            reg = <0x13>;
            #power-domain-cells = <1>;
        };

        /* Pinctrl protocol (i.MX95 specific) */
        scmi_iomuxc: protocol@19 {
            reg = <0x19>;
        };
    };
};

/* SCMI shared memory buffer trong SRAM */
reserved-memory {
    scmi_buf: scmi_buf@44900000 {
        reg = <0x0 0x44900000 0x0 0x400>;  /* 1KB */
        no-map;
    };
};

/* Cách dùng SCMI clock trong SAI node */
&sai3 {
    clocks = <&scmi_clk IMX95_CLK_SAI3>,         /* bus clock */
             <&scmi_clk IMX95_CLK_SAI3>;          /* audio clock */
    clock-names = "bus", "mclk0";
    assigned-clocks = <&scmi_clk IMX95_CLK_SAI3>;
    assigned-clock-rates = <49152000>;            /* 48kHz × 1024 */
};
```

### 4.4 SCMI Message Format

```c
/*
 * SCMI message header (ARM SCMI Spec, Section 4.2)
 * Ghi vào shared memory buffer trước khi kick MU
 */
struct scmi_msg_hdr {
    uint32_t token    : 10;  /* sequence number, match req/resp */
    uint32_t protocol : 8;   /* e.g. 0x14 = CLOCK */
    uint32_t type     : 2;   /* 0=command, 1=delayed_resp, 2=notify */
    uint32_t msg_id   : 8;   /* command ID trong protocol */
    uint32_t reserved : 4;
};

/* Ví dụ: SCMI_CLOCK_RATE_SET (msg_id=5, protocol=0x14) */
struct scmi_clock_rate_set {
    struct scmi_msg_hdr hdr;
    uint32_t flags;       /* 0=async, 1=sync */
    uint32_t clock_id;    /* e.g. IMX95_CLK_SAI3 */
    uint32_t rate_high;   /* upper 32 bits của rate */
    uint32_t rate_low;    /* lower 32 bits của rate */
                          /* rate = (rate_high << 32) | rate_low */
};
```

### 4.5 Debug SCMI

```bash
# Xem tất cả SCMI clocks Linux biết
adb shell cat /sys/kernel/debug/clk/clk_summary | grep scmi

# Xem clock rate của SAI3
adb shell cat /sys/kernel/debug/clk/sai3/clk_rate
# Output: 49152000

# SCMI error log
adb shell dmesg | grep -i "scmi\|arm-scmi"
# [    1.234] arm-scmi: SCMI Firmware version 0x20000
# [    1.235] arm-scmi: protocol 0x14 version 0x20000

# Dump toàn bộ clock tree (bao gồm SCMI clocks)
adb shell cat /sys/kernel/debug/clk/clk_summary

# Xem power domains qua SCMI
adb shell cat /sys/kernel/debug/pm_genpd/pm_genpd_summary
```

---

## Chương 5 — Cache Coherency & Shared Memory giữa các Core

### 5.1 Vấn đề Cache Coherency trong AMP

Đây là **nguồn bug khó nhất** trong multi-core development. A55 có L1/L2 cache, M7 có TCM (Tightly Coupled Memory) và optional cache. Khi cả hai cùng đọc/ghi shared memory, dữ liệu có thể không nhất quán.

```
┌─────────────────────────────────────────────────────────┐
│  Cortex-A55 Core0                                       │
│  ┌──────────┐  ┌──────────┐                            │
│  │  L1 I$   │  │  L1 D$   │  (32KB each, private)      │
│  └──────────┘  └──────────┘                            │
│  ┌───────────────────────────┐                         │
│  │         L2 Cache          │  (512KB, shared cluster) │
│  └───────────────────────────┘                         │
│  ┌───────────────────────────┐                         │
│  │    CCI / DSU (L3 Cache)   │  (1MB, SoC-level)       │
│  └─────────────┬─────────────┘                         │
└────────────────┼────────────────────────────────────────┘
                 │ AXI bus
┌────────────────┼────────────────────────────────────────┐
│  Cortex-M7     │                                        │
│  ┌─────────────┴──────────┐                            │
│  │  ITCM / DTCM           │  (64KB/128KB, zero-latency) │
│  │  (NO cache, direct)    │                            │
│  └────────────────────────┘                            │
│  ┌────────────────────────┐                            │
│  │  M7 AXI Cache (opt)    │  (16KB, nếu enable)        │
│  └────────────────────────┘                            │
└─────────────────────────────────────────────────────────┘
                 │
         Shared DDR (non-cacheable region)
```

### 5.2 Ba kỹ thuật xử lý Cache Coherency

**Kỹ thuật 1: Non-cacheable memory (đơn giản nhất)**

```c
/*
 * Linux side: khai báo shared buffer là non-cacheable
 * Dùng dma_alloc_coherent() hoặc reserved-memory với no-map
 */

/* Trong driver */
dma_addr_t dma_handle;
void *shared_buf;

/* Allocate non-cacheable, DMA-accessible memory */
shared_buf = dma_alloc_coherent(dev,
                                SHARED_BUF_SIZE,
                                &dma_handle,
                                GFP_KERNEL);
/*
 * dma_alloc_coherent trên ARM64:
 * - Nếu IOMMU present: map qua IOMMU
 * - Nếu không: allocate từ non-cacheable region
 * - CPU access sẽ bypass L1/L2 cache → thẳng xuống DDR
 * - M7 đọc DDR → luôn thấy data mới nhất
 */
```

**Kỹ thuật 2: Cache flush/invalidate thủ công**

```c
/*
 * Dùng khi PHẢI dùng cacheable memory (performance reason)
 * Rule: "Producer flush, Consumer invalidate"
 */

/* A55 producer: ghi data xong → flush cache xuống DDR */
void a55_send_to_m7(void *buf, size_t len)
{
    memcpy(shared_mem, buf, len);

    /* Flush: đẩy dirty cache lines xuống DDR */
    /* M7 sẽ thấy data này khi đọc DDR */
    __flush_dcache_area(shared_mem, len);

    /* Memory barrier: đảm bảo flush hoàn thành trước khi kick M7 */
    dsb(sy);        /* Data Synchronization Barrier */
    dmb(sy);        /* Data Memory Barrier */

    /* Bây giờ mới kick M7 qua MU */
    mbox_send_message(tx_ch, &vqid);
}

/* A55 consumer: trước khi đọc data từ M7 → invalidate cache */
void a55_recv_from_m7(void *buf, size_t len)
{
    /* Invalidate: bỏ stale cache lines, force đọc từ DDR */
    __inval_dcache_area(shared_mem, len);
    dmb(sy);

    memcpy(buf, shared_mem, len);
}
```

**Kỹ thuật 3: Hardware Cache Coherency (CCI/ACE)**

i.MX95 Cortex-A55 cluster kết nối với DSU (DynamIQ Shared Unit) và CCI (Cache Coherent Interconnect). Nhưng **M7 không tham gia coherency domain này** — M7 kết nối qua AHB/AXI thông thường. Do đó kỹ thuật 1 hoặc 2 luôn cần thiết cho A55↔M7.

### 5.3 Memory Barrier cheat sheet

```c
/*
 * ARM64 barrier instructions — phải thuộc lòng khi làm multi-core
 */

dsb(sy);    /* Data Synchronization Barrier — System
             * Chờ TẤT CẢ memory operations trước đó hoàn thành
             * Dùng trước khi: kick remote core, release spinlock */

dmb(sy);    /* Data Memory Barrier — System
             * Đảm bảo ordering của memory accesses
             * Nhẹ hơn DSB, không chờ non-memory ops */

isb();      /* Instruction Synchronization Barrier
             * Flush pipeline, refetch instructions
             * Dùng sau khi: thay đổi MMU/cache config */

/* Ví dụ pattern chuẩn để publish data tới core khác */
shared_data->payload = my_data;     /* ghi data */
shared_data->length  = my_len;
wmb();                              /* write memory barrier */
shared_data->ready   = 1;          /* flag — M7 poll cái này */
dsb(sy);                            /* flush tất cả */
/* kick M7 */
```

### 5.4 Vring và Cache — Cụ thể với RPMsg

```c
/*
 * Remoteproc framework tự xử lý cache coherency cho vring
 * File: drivers/remoteproc/remoteproc_virtio.c
 */

/* Vring được map là non-cacheable (MEMREMAP_WC hoặc dma_coherent) */
static int rproc_virtio_create_virtqueues(...)
{
    /* vrings được allocate từ reserved-memory với no-map
     * → kernel map chúng là non-cacheable tự động
     * → không cần manual flush/invalidate cho vring descriptors */
    vqs[i] = vring_new_virtqueue(...,
                                  rproc_vdev->vdev.dev.parent,
                                  true,   /* weak_barriers */
                                  ...);
}

/*
 * NHƯNG: payload buffers (data thực sự) cần chú ý
 * Nếu dùng zero-copy (pointer trực tiếp vào kernel buffer)
 * → phải flush trước khi add vào vring
 */
```

---

## Chương 6 — ELE (EdgeLock Enclave) — Security Core

### 6.1 ELE là gì?

ELE là **security subsystem độc lập** trên i.MX95 — một Cortex-M33 chuyên dụng chạy firmware của NXP (closed-source), không thể bị Linux/ATF override.

```
┌─────────────────────────────────────────────────────────┐
│                    i.MX95 SoC                           │
│                                                         │
│  ┌──────────────────────────────────────────────────┐  │
│  │           ELE (EdgeLock Enclave)                  │  │
│  │           Cortex-M33 (dedicated)                 │  │
│  │                                                  │  │
│  │  ┌────────────────────────────────────────────┐  │  │
│  │  │  NXP Signed Firmware (ROM + RAM)           │  │  │
│  │  │  - Key management (device keys, OTP)       │  │  │
│  │  │  - Secure boot chain verification          │  │  │
│  │  │  - TRNG (True Random Number Generator)     │  │  │
│  │  │  - Crypto services (AES, RSA, ECC, SHA)    │  │  │
│  │  │  - Lifecycle management (OEM → Closed)     │  │  │
│  │  │  - Attestation                             │  │  │
│  │  └────────────────────────────────────────────┘  │  │
│  │                                                  │  │
│  │  Private: OTP fuses, device keys, secure RAM     │  │
│  │  Linux KHÔNG THỂ đọc trực tiếp                   │  │
│  └─────────────────┬────────────────────────────────┘  │
│                    │ MU (dedicated ELE MU)              │
│  ┌─────────────────▼────────────────────────────────┐  │
│  │           Linux / ATF                            │  │
│  │  imx-ele driver → /dev/ele_mu0_ch0               │  │
│  │  Gửi request: TRNG, crypto, key generation...    │  │
│  └──────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────┘
```

### 6.2 Secure Boot Chain trên EVK95

```
BootROM
  │  1. Đọc SRK (Super Root Key) hash từ OTP fuses
  │  2. Verify signature của SPL/IVT
  │  3. Nếu fail → brick (tùy lifecycle state)
  ▼
SPL (signed bởi OEM key)
  │  HABv4 / AHAB verification
  ▼
ATF BL2 (signed)
  │
  ▼
U-Boot (signed)
  │
  ▼
Linux Kernel + DTB (verified bởi dm-verity / Android Verified Boot)
  │
  ▼
Android (AVB2.0)
```

### 6.3 Linux tương tác với ELE

```c
/*
 * Driver: drivers/firmware/imx/ele_base_msg.c
 * Interface: /dev/ele_mu0_ch0
 */

/* Ví dụ: lấy random number từ ELE TRNG */
#include <linux/firmware/imx/ele_base_msg.h>

int get_random_from_ele(uint8_t *buf, size_t len)
{
    struct ele_msg msg = {
        .version = ELE_VERSION,
        .tag     = ELE_CMD_TAG,
        .size    = 2,
        .command = ELE_GET_TRNG_STATE_REQ,
    };

    /* Gửi request tới ELE qua MU */
    return imx_ele_msg_send_rcv(msg_hdl, &msg);
    /* ELE xử lý trong secure domain, trả về random bytes */
}
```

```bash
# Kiểm tra ELE driver load
adb shell dmesg | grep -i "ele\|edgelock"
# [    1.456] imx-ele: i.MX EdgeLock Enclave (ELE) driver initialized

# Lifecycle state của SoC
adb shell cat /sys/bus/platform/devices/*/lifecycle 2>/dev/null
# Output: OEM_OPEN (development) hoặc OEM_CLOSED (production)

# Kiểm tra secure fuse state
adb shell dmesg | grep -i "fuse\|otp\|hab"
```

---

## Chương 7 — Power Management & Power Domains

### 7.1 Power Domain Architecture trên i.MX95

```
┌─────────────────────────────────────────────────────────────────┐
│                    Power Domain Hierarchy                        │
│                                                                 │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │  SCMI Power Domain (managed by System Manager)           │  │
│  │                                                          │  │
│  │  PD_A55_CORE0  PD_A55_CORE1  PD_A55_CORE2  PD_A55_CORE3 │  │
│  │  PD_A55_CLUSTER (L2/DSU)                                 │  │
│  │  PD_M7                                                   │  │
│  │  PD_DISPLAY  PD_GPU  PD_VPU  PD_AUDIO  PD_PCIE          │  │
│  └──────────────────────────────────────────────────────────┘  │
│                              │                                  │
│  ┌───────────────────────────▼──────────────────────────────┐  │
│  │  Linux Generic Power Domain (genpd)                      │  │
│  │  drivers/pmdomain/imx/scmi-imx95-pd.c                    │  │
│  └──────────────────────────────────────────────────────────┘  │
│                              │                                  │
│  ┌───────────────────────────▼──────────────────────────────┐  │
│  │  Device PM (Runtime PM)                                  │  │
│  │  SAI, GPU, display drivers tự gọi pm_runtime_get/put     │  │
│  └──────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────┘
```

### 7.2 CPU Idle States — Cortex-A55

```dts
/* arch/arm64/boot/dts/freescale/imx95.dtsi */
idle-states {
    entry-method = "psci";

    /* C1: Clock gating — nhanh nhất, tiết kiệm ít nhất */
    CPU_SLEEP: cpu-sleep {
        compatible      = "arm,idle-state";
        arm,psci-suspend-param = <0x0010000>;  /* STANDBYWFI */
        local-timer-stop;
        entry-latency-us  = <10>;
        exit-latency-us   = <10>;
        min-residency-us  = <100>;
    };

    /* C2: Power gating core — tiết kiệm hơn, latency cao hơn */
    CORE_PD: core-pd {
        compatible      = "arm,idle-state";
        arm,psci-suspend-param = <0x0010001>;  /* CORE_PD */
        local-timer-stop;
        entry-latency-us  = <200>;
        exit-latency-us   = <200>;
        min-residency-us  = <2000>;
    };
};
```

### 7.3 Runtime PM — Driver tự quản lý power

```c
/*
 * SAI audio driver — tự power gate khi idle
 * File: sound/soc/fsl/fsl_sai.c
 */

static int fsl_sai_probe(struct platform_device *pdev)
{
    /* ... */

    /* Enable runtime PM */
    pm_runtime_enable(&pdev->dev);
    pm_runtime_get_sync(&pdev->dev);   /* power ON ngay khi probe */

    /* init registers... */

    pm_runtime_put_sync(&pdev->dev);   /* power OFF sau probe */
    return 0;
}

static int fsl_sai_hw_params(struct snd_pcm_substream *substream, ...)
{
    struct fsl_sai *sai = snd_soc_dai_get_drvdata(dai);

    /* Khi audio bắt đầu: power ON SAI domain */
    pm_runtime_get_sync(cpu_dai->dev);
    /* ... configure SAI registers ... */
    return 0;
}

static int fsl_sai_trigger(... SND_SOC_DAPM_POST_PMD)
{
    /* Khi audio dừng: power OFF SAI domain sau timeout */
    pm_runtime_put_autosuspend(cpu_dai->dev);
}

/* Runtime PM callbacks */
static const struct dev_pm_ops fsl_sai_pm_ops = {
    SET_RUNTIME_PM_OPS(fsl_sai_runtime_suspend,
                       fsl_sai_runtime_resume, NULL)
    SET_SYSTEM_SLEEP_PM_OPS(pm_runtime_force_suspend,
                             pm_runtime_force_resume)
};

static int fsl_sai_runtime_suspend(struct device *dev)
{
    struct fsl_sai *sai = dev_get_drvdata(dev);
    clk_disable_unprepare(sai->bus_clk);   /* tắt clock */
    /* SCMI → SM → power gate SAI domain */
    return 0;
}
```

### 7.4 M7 Power Management từ Linux

```bash
# Check M7 power domain state
adb shell cat /sys/kernel/debug/pm_genpd/pm_genpd_summary | grep -i m7

# M7 không thể suspend nếu đang chạy remoteproc
# Linux biết điều này qua power domain dependency:
#   remoteproc0 → PD_M7 → không thể power gate

# Suspend toàn hệ thống (A55 + M7 phải coordinate)
# M7 nhận SUSPEND notification qua RPMsg trước khi A55 suspend
adb shell echo mem > /sys/power/state
```

---

## Chương 8 — Inter-Core Debugging Chuyên sâu

### 8.1 Debug M7 từ Linux (OpenOCD + GDB)

```
┌──────────────────────────────────────────────────────────┐
│  Host PC                                                 │
│  ┌─────────────┐     ┌───────────────┐                  │
│  │  arm-none-  │     │   OpenOCD     │                  │
│  │  eabi-gdb   │◄───►│               │                  │
│  │             │     │  imx95.cfg    │                  │
│  └─────────────┘     └───────┬───────┘                  │
└──────────────────────────────┼───────────────────────────┘
                               │ JTAG/SWD
┌──────────────────────────────┼───────────────────────────┐
│  EVK95 Board                 │                           │
│                              ▼                           │
│                        DAP (Debug Access Port)           │
│                         │           │                    │
│                    A55 CoreSight   M7 DAP                │
│                    (ETM, CTI)      (FPB, DWT)            │
└──────────────────────────────────────────────────────────┘
```

```tcl
# OpenOCD config: imx95_evk.cfg
source [find target/imx95.cfg]

# Connect tới M7 core riêng biệt
target create imx95.m7 cortex_m -chain-position imx95.dap \
    -ap-num 2 -coreid 0

# Set breakpoint trên M7
init
targets imx95.m7
halt
bp 0x20000100 4 hw   ;# breakpoint tại địa chỉ ITCM
resume
```

```bash
# GDB session cho M7 firmware
arm-none-eabi-gdb build/m7_firmware.elf
(gdb) target remote localhost:3333
(gdb) monitor reset halt
(gdb) b rpmsg_task          # breakpoint tại RPMsg task
(gdb) c
(gdb) info registers        # xem registers M7
(gdb) x/16xw 0xB8000000    # xem shared memory (vring)
```

### 8.2 Trace RPMsg Messages Real-time

```bash
# === Kernel ftrace cho RPMsg ===
adb shell "
  cd /sys/kernel/debug/tracing
  echo 0 > tracing_on
  echo > trace
  echo 'rpmsg_*' > set_ftrace_filter
  echo function > current_tracer
  echo 1 > tracing_on
"

# Chạy test gửi message
adb shell "echo 'TEST' > /dev/rpmsg0"

# Đọc trace
adb shell cat /sys/kernel/debug/tracing/trace
# Output:
# kworker/0:1-123  [000] .... rpmsg_send: src=0x400 dst=0x1e len=4
# kworker/0:1-123  [000] .... virtqueue_add_outbuf: idx=5

# === Trace MU interrupts ===
adb shell "
  echo 1 > /sys/kernel/debug/tracing/events/irq/irq_handler_entry/enable
"
adb shell cat /sys/kernel/debug/tracing/trace | grep "mu\|234"
```

### 8.3 Shared Memory Inspector — Tool tự viết

```c
/*
 * File: tools/vring_inspector.c
 * Compile: gcc -o vring_inspector vring_inspector.c
 * Run: adb shell ./vring_inspector /dev/mem 0xB8000000 0x10000
 *
 * Tool đọc vring state trực tiếp từ /dev/mem để debug
 */
#include <stdio.h>
#include <stdlib.h>
#include <fcntl.h>
#include <sys/mman.h>
#include <stdint.h>

/* Virtio vring descriptor */
struct vring_desc {
    uint64_t addr;    /* buffer address */
    uint32_t len;     /* buffer length */
    uint16_t flags;   /* VRING_DESC_F_NEXT, VRING_DESC_F_WRITE */
    uint16_t next;    /* if F_NEXT: index of next descriptor */
};

struct vring_avail {
    uint16_t flags;
    uint16_t idx;      /* producer writes here */
    uint16_t ring[];   /* descriptor indices */
};

struct vring_used_elem {
    uint32_t id;
    uint32_t len;
};

struct vring_used {
    uint16_t flags;
    uint16_t idx;      /* consumer writes here */
    struct vring_used_elem ring[];
};

void dump_vring(volatile void *base, int num_desc)
{
    volatile struct vring_desc  *desc  = base;
    volatile struct vring_avail *avail = base + num_desc * sizeof(*desc);
    volatile struct vring_used  *used  = (void*)((uintptr_t)(avail)
        + sizeof(*avail) + num_desc * sizeof(uint16_t) + sizeof(uint16_t));

    printf("=== VRING STATE (num=%d) ===\n", num_desc);
    printf("Avail idx: %u (producer wrote %u descs)\n",
           avail->idx, avail->idx);
    printf("Used  idx: %u (consumer consumed %u descs)\n",
           used->idx, used->idx);
    printf("Pending: %u messages\n",
           (uint16_t)(avail->idx - used->idx));

    /* Dump last 4 descriptors */
    for (int i = 0; i < 4 && i < num_desc; i++) {
        printf("Desc[%d]: addr=0x%lx len=%u flags=0x%x\n",
               i, desc[i].addr, desc[i].len, desc[i].flags);
    }
}

int main(int argc, char *argv[])
{
    int fd = open("/dev/mem", O_RDONLY | O_SYNC);
    off_t base = strtoul(argv[1], NULL, 16);
    size_t size = strtoul(argv[2], NULL, 16);

    void *mem = mmap(NULL, size, PROT_READ, MAP_SHARED, fd, base);
    dump_vring(mem, 256);

    return 0;
}
```

### 8.4 M7 Coredump Analysis

```bash
# Khi M7 crash, remoteproc tự động lấy coredump

# Kích hoạt coredump
adb shell "echo 'disabled' > /sys/module/remoteproc/parameters/recovery"

# Xem coredump
adb shell ls /sys/class/remoteproc/remoteproc0/coredump
adb pull /sys/class/remoteproc/remoteproc0/coredump /tmp/m7_coredump.elf

# Analyze với GDB
arm-none-eabi-gdb build/m7_firmware.elf /tmp/m7_coredump.elf
(gdb) bt       # backtrace tại thời điểm crash
(gdb) info reg # registers state lúc crash
(gdb) x/32xw $sp  # stack dump

# Tìm nguồn crash từ PC register
(gdb) list *0x20001234   # xem source tại PC = 0x20001234
```

---

## Chương 9 — Cortex-M33 (System Manager) — Core thứ 3 của EVK95

### 9.1 M33 vs M7 — Khác nhau thế nào?

```
┌─────────────────────────────────────────────────────────────┐
│  Cortex-M33 (System Manager)    Cortex-M7 (Real-time App)  │
├─────────────────────────────────────────────────────────────┤
│  Boot:  BootROM boot đầu tiên   Boot: Linux remoteproc     │
│  FW:    NXP System Manager      FW:   OEM FreeRTOS          │
│  Code:  Closed source (NXP)     Code: Open (OEM)            │
│  Role:  Platform controller     Role: Application RT        │
│  SCMI:  Server (provider)       SCMI: không dùng            │
│  Power: ALWAYS ON               Power: Controlled by Linux  │
│  DDR:   NO access               DDR:  YES (shared mem)      │
│  OTP:   YES (via ELE)           OTP:  NO                    │
│  Perf:  240 MHz                 Perf: 800 MHz               │
│  Cache: 16KB I + 16KB D         Cache: 16KB I + 16KB D      │
│         + 256KB TCM                     + 256KB TCM          │
└─────────────────────────────────────────────────────────────┘
```

### 9.2 SCMI Server trên M33 — Những gì SM quản lý

```c
/*
 * System Manager firmware (NXP SM, partial open source)
 * github.com/nxp-imx/imx-sm
 *
 * SM nhận SCMI request từ Linux qua MU, xử lý:
 */

/* Clock management */
int32_t SM_CLOCKRATE_SET(uint32_t clockId, uint64_t rate)
{
    /* Validate: Linux có quyền set clock này không? */
    if (!sm_clock_allowed(clockId, caller_id))
        return SCMI_ERR_DENIED;

    /* Thao tác CCM (Clock Controller Module) registers */
    CCM->CLOCK_ROOT[clockId].CONTROL = compute_dividers(rate);
    return SCMI_SUCCESS;
}

/* Reset management */
int32_t SM_RESET_ASSERT(uint32_t resetId)
{
    /* e.g. reset SAI3, PCIe, USB */
    SRC->CTRL[resetId] |= SRC_CTRL_SW_RESET;
    return SCMI_SUCCESS;
}

/* Power domain */
int32_t SM_POWER_STATE_SET(uint32_t domainId, uint32_t state)
{
    if (state == SCMI_POWER_STATE_ON)
        GPC->PD_CTRL[domainId] |= GPC_PD_CTRL_ON;
    else
        GPC->PD_CTRL[domainId] &= ~GPC_PD_CTRL_ON;
    return SCMI_SUCCESS;
}
```

### 9.3 M33 Boot Sequence

```
EVK95 Power ON
    │
    ▼
BootROM (on-chip)
    ├── Load M33 System Manager từ flash/eMMC
    │   (M33 boot TRƯỚC A55)
    │
    ▼
M33 System Manager starts
    ├── Init clocks (PLL, clock roots)
    ├── Init DDR (M33 controls DDRPHY init sequence)
    ├── Init power domains
    ├── Start SCMI server (wait for requests)
    │
    ▼
M33 releases A55 reset
    │
    ▼
A55 Core0 starts → BootROM → SPL → ATF → U-Boot → Linux
    │
    ▼
Linux gửi SCMI requests → M33 xử lý
(Linux không biết M33 tồn tại — chỉ biết "có SCMI firmware")
```

### 9.4 Debug SCMI/SM từ Linux

```bash
# SM có thể in log qua UART hoặc shared memory log buffer

# Xem SM version
adb shell dmesg | grep "SM\|scmi\|arm-scmi"

# Dump SCMI performance (DVFS) levels
adb shell cat /sys/bus/platform/devices/*/scmi_dvfs/*/opp_table 2>/dev/null

# Test SCMI clock manually
adb shell "
  # Dùng clk_summary để verify SCMI clock operations
  cat /sys/kernel/debug/clk/clk_summary > /tmp/clk_before.txt
  # Thay đổi audio clock
  echo 24576000 > /sys/kernel/debug/clk/sai1/clk_rate
  cat /sys/kernel/debug/clk/clk_summary > /tmp/clk_after.txt
  diff /tmp/clk_before.txt /tmp/clk_after.txt
"

# SCMI error statistics
adb shell cat /sys/bus/platform/devices/*/scmi_info 2>/dev/null
```

---

## Chương 10 — CPU Frequency Scaling (DVFS) — Thực tế với Android

### 10.1 DVFS Flow trên i.MX95

```
Android (cpufreq governor)
    │  "tăng/giảm frequency dựa trên load"
    ▼
Linux CPUFreq Framework
    │  drivers/cpufreq/scmi-cpufreq.c
    ▼
SCMI Performance Protocol (0x13)
    │  PERF_LEVEL_SET request
    ▼
System Manager (M33)
    │  Điều chỉnh PLL + voltage (PMIC)
    ▼
Cortex-A55 chạy ở frequency mới
```

### 10.2 OPP Table trong DTS

```dts
/* Operating Performance Points cho Cortex-A55 */
cpu0_opp_table: opp-table {
    compatible = "operating-points-v2";

    opp-500000000 {
        opp-hz = /bits/ 64 <500000000>;   /* 500 MHz */
        opp-microvolt = <800000>;          /* 0.8V */
        clock-latency-ns = <50000>;
    };

    opp-800000000 {
        opp-hz = /bits/ 64 <800000000>;   /* 800 MHz */
        opp-microvolt = <900000>;          /* 0.9V */
    };

    opp-1200000000 {
        opp-hz = /bits/ 64 <1200000000>;  /* 1.2 GHz */
        opp-microvolt = <1000000>;         /* 1.0V */
    };

    opp-1800000000 {
        opp-hz = /bits/ 64 <1800000000>;  /* 1.8 GHz — max */
        opp-microvolt = <1100000>;         /* 1.1V */
        opp-supported-hw = <0x01>;         /* chỉ chip grade A */
    };
};
```

### 10.3 Debug DVFS

```bash
# Xem frequency hiện tại
adb shell cat /sys/devices/system/cpu/cpu0/cpufreq/scaling_cur_freq
# Output: 1200000  (1.2 GHz, đơn vị kHz)

# Xem available frequencies
adb shell cat /sys/devices/system/cpu/cpu0/cpufreq/scaling_available_frequencies
# Output: 500000 800000 1200000 1800000

# Lock frequency (cho audio latency testing)
adb shell "
  echo performance > /sys/devices/system/cpu/cpu0/cpufreq/scaling_governor
  echo 1800000 > /sys/devices/system/cpu/cpu0/cpufreq/scaling_min_freq
"

# Monitor frequency real-time
adb shell "
  while true; do
    cat /sys/devices/system/cpu/cpu*/cpufreq/scaling_cur_freq | tr '\n' ' '
    echo
    sleep 0.5
  done
"

# Xem DVFS log
adb shell dmesg | grep -i "cpufreq\|opp\|dvfs"
```

---

## Chương 11 — Inter-Core Synchronization Patterns

### 11.1 Atomic Operations qua Shared Memory

```c
/*
 * Scenario: A55 và M7 cùng access shared counter
 * KHÔNG được dùng normal increment — race condition!
 */

/* WRONG ❌ */
shared_mem->counter++;  /* read-modify-write không atomic */

/* CORRECT ✅ — Linux side (A55) */
#include <linux/atomic.h>

/* Đảm bảo atomic trên A55 (nhưng M7 không dùng Linux atomics) */
/* Giải pháp tốt hơn: dùng hardware spinlock */
```

### 11.2 Hardware Spinlock — HSEM trên i.MX95

i.MX95 có **Hardware Semaphore (HSEM)** — đây là cách duy nhất để A55 và M7 cùng protect shared resource một cách thực sự atomic.

```c
/*
 * Linux driver: drivers/hwspinlock/imx_hwspinlock.c
 * M7 SDK: fsl_hsem.c
 */

/* === LINUX SIDE (A55) === */
#include <linux/hwspinlock.h>

struct hwspinlock *hsem;

/* Lấy hardware spinlock #0 */
hsem = hwspin_lock_request_specific(0);

/* Lock (busy-wait hoặc timeout) */
hwspin_lock_timeout(hsem, 1000 /* ms timeout */);

    /* Critical section — an toàn truy cập shared memory */
    shared_mem->value = new_value;
    dsb(sy);

/* Unlock */
hwspin_unlock(hsem);
```

```c
/* === M7 SIDE (FreeRTOS) === */
#include "fsl_hsem.h"

/* Lock HSEM #0 từ M7 */
while (HSEM_TryLock(HSEM, 0, MASTER_ID_M7) != kStatus_Success)
    ; /* spin */

    /* Critical section */
    shared_data->value = new_value;
    __DSB();

/* Unlock */
HSEM_Unlock(HSEM, 0, MASTER_ID_M7);
```

```dts
/* DTS cho HSEM */
hwlock: hwspinlock@44300000 {
    compatible = "fsl,imx95-hwspinlock";
    reg = <0x0 0x44300000 0x0 0x10000>;
    #hwlock-cells = <1>;
    clocks = <&scmi_clk IMX95_CLK_HSEM>;
};

/* Consumer dùng HSEM #0 */
my_driver {
    hwlocks = <&hwlock 0>;
    hwlock-names = "shared_buf_lock";
};
```

### 11.3 Notification Pattern (Polling vs Interrupt)

```c
/*
 * Pattern 1: Polling (M7 side đơn giản hơn)
 * M7 poll một flag trong shared memory
 * A55 set flag sau khi ghi data
 */

/* Shared memory layout */
struct shared_control {
    volatile uint32_t a55_to_m7_flag;    /* A55 set, M7 clear */
    volatile uint32_t m7_to_a55_flag;    /* M7 set, A55 clear */
    volatile uint32_t sequence_num;      /* tránh miss message */
    uint8_t           data[PAGE_SIZE];
};

/* A55: ghi data và set flag */
void a55_notify_m7(struct shared_control *ctrl, void *data, size_t len)
{
    memcpy(ctrl->data, data, len);
    ctrl->sequence_num++;
    wmb();                          /* write barrier */
    __flush_dcache_area(ctrl, sizeof(*ctrl));
    dsb(sy);
    ctrl->a55_to_m7_flag = 1;      /* set flag */
    dsb(sy);
    /* KHÔNG cần kick MU nếu M7 đang poll */
}

/* M7: polling task (FreeRTOS) */
void m7_poll_task(void *param)
{
    volatile struct shared_control *ctrl =
        (volatile struct shared_control *)SHARED_MEM_BASE;
    uint32_t last_seq = 0;

    while (1) {
        /* Poll với yield để không starve other tasks */
        if (ctrl->a55_to_m7_flag && ctrl->sequence_num != last_seq) {
            last_seq = ctrl->sequence_num;
            ctrl->a55_to_m7_flag = 0;

            process_data((void *)ctrl->data);
        }
        taskYIELD();  /* cho FreeRTOS scheduler chạy */
    }
}

/*
 * Pattern 2: Interrupt-driven (hiệu quả hơn cho power)
 * Dùng MU interrupt — đây là cách RPMsg hoạt động
 * Xem chương 3.4 ở trên
 */
```

---

## Chương 12 — Automotive Specific: Safety Core Coordination

### 12.1 Functional Safety với M33 (ASIL-B/D)

Trên automotive i.MX95, M33 còn có vai trò **safety monitor**:

```
┌─────────────────────────────────────────────────────────┐
│  Safety Architecture                                    │
│                                                         │
│  ┌────────────────┐  Heartbeat (500ms)  ┌────────────┐ │
│  │  Android (A55) │ ─────────────────► │    M33     │ │
│  │                │                    │  (Safety   │ │
│  │                │ ◄───────────────── │  Monitor)  │ │
│  └────────────────┘  Watchdog kick     └─────┬──────┘ │
│                                              │         │
│                                   Nếu A55 không kick:  │
│                                   → M33 trigger reset  │
│                                   → hoặc safe state    │
└──────────────────────────────────────────────────────────┘
```

### 12.2 Watchdog coordination qua SCMI

```c
/*
 * Linux Watchdog driver giao tiếp qua SCMI
 * File: drivers/watchdog/scmi_wdt.c (custom)
 */

/* Android Watchdog Service định kỳ kick */
static void scmi_wdt_keepalive(struct watchdog_device *wdd)
{
    struct scmi_wdt *wdt = to_scmi_wdt(wdd);

    /* Gửi SCMI message tới M33 để reset watchdog timer */
    scmi_wdt_ops->keepalive(wdt->handle, wdt->id);
    /*
     * Nếu Linux hang/crash → không kick → M33 timeout
     * M33 có thể:
     * 1. Hard reset SoC
     * 2. Chuyển sang safe state (tắt display, audio, etc.)
     * 3. Notify safety ECU qua CAN
     */
}
```

---

## Tổng kết — Sơ đồ đầy đủ i.MX95 Multi-Core

```
┌──────────────────────────────────────────────────────────────────────┐
│                          i.MX95 EVK Board                            │
│                                                                      │
│  ┌─────────────────────────────────────────────────────────────────┐ │
│  │                     i.MX95 SoC                                  │ │
│  │                                                                 │ │
│  │  ┌──────────────┐  ┌──────────────┐  ┌─────────────────────┐  │ │
│  │  │ Cortex-A55   │  │ Cortex-M7    │  │ Cortex-M33 (SM)     │  │ │
│  │  │ (×4, SMP)    │  │ (AMP)        │  │ (Always-on)         │  │ │
│  │  │              │  │              │  │                     │  │ │
│  │  │ Android 15   │  │ FreeRTOS     │  │ NXP System Manager  │  │ │
│  │  │ EL0/EL1/EL2  │  │ bare-metal   │  │ SCMI Server         │  │ │
│  │  │              │  │              │  │ Safety Monitor      │  │ │
│  │  └──────┬───────┘  └──────┬───────┘  └──────────┬──────────┘  │ │
│  │         │                 │                      │             │ │
│  │  ┌──────▼─────────────────▼──────────────────────▼──────────┐  │ │
│  │  │              Interconnect (AXI/AHB/APB)                  │  │ │
│  │  │                                                          │  │ │
│  │  │  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌────────┐  │  │ │
│  │  │  │ MU1      │  │ MU2      │  │  HSEM    │  │  ELE   │  │  │ │
│  │  │  │ (RPMsg)  │  │ (SCMI)   │  │ (HW Lock)│  │(M33-S) │  │  │ │
│  │  │  │ A55↔M7   │  │ A55↔SM   │  │ A55+M7   │  │        │  │  │ │
│  │  │  └──────────┘  └──────────┘  └──────────┘  └────────┘  │  │ │
│  │  │                                                          │  │ │
│  │  │  ┌────────────────────────────────────────────────────┐  │  │ │
│  │  │  │                DDR Memory                          │  │  │ │
│  │  │  │  ┌──────────┐ ┌──────────┐ ┌────────────────────┐ │  │  │ │
│  │  │  │  │ Linux    │ │ M7 FW   │ │ Shared (vring+buf) │ │  │  │ │
│  │  │  │  │ (cached) │ │         │ │   (non-cacheable)  │ │  │  │ │
│  │  │  │  └──────────┘ └──────────┘ └────────────────────┘ │  │  │ │
│  │  │  └────────────────────────────────────────────────────┘  │  │ │
│  │  └──────────────────────────────────────────────────────────┘  │ │
│  └─────────────────────────────────────────────────────────────────┘ │
└──────────────────────────────────────────────────────────────────────┘

Luồng dữ liệu audio (use case thực tế):
  Android AudioFlinger
    → HAL → /dev/rpmsg_audio
    → RPMsg (vring) → MU interrupt
    → M7 FreeRTOS audio task
    → DSP processing (EQ, ANC, volume)
    → SAI DMA → TAS5828 → Speaker
```

---

## Tham khảo

| Tài liệu | Link |
|---|---|
| NXP i.MX95 Reference Manual | NXP website → i.MX 95 Applications Processor |
| NXP System Manager (SM) source | github.com/nxp-imx/imx-sm |
| Linux Remoteproc docs | `Documentation/staging/remoteproc.rst` |
| RPMsg-Lite (MCU SDK) | github.com/nxp-mcuxpresso/rpmsg-lite |
| ARM PSCI spec | developer.arm.com/documentation/den0022 |
| TF-A source | git.trustedfirmware.org/TF-A/trusted-firmware-a |
| Virtio spec (vring) | docs.oasis-open.org/virtio/virtio/v1.2 |
| ARM SCMI spec | developer.arm.com/documentation/den0056 |
| SMCCC spec | developer.arm.com/documentation/den0028 |
| FreeRTOS SMP docs | freertos.org/symmetric-multiprocessing-introduction |
| NXP MCUXpresso SDK (M7) | mcuxpresso.nxp.com |

---

## Chương 13 — STR (Suspend to RAM) — Multi-Core Coordination

> STR là kịch bản phức tạp nhất: 3 processor (A55, M7, M33) + ATF + ELE phải phối hợp chính xác theo thứ tự, nếu sai bước nào → deadlock hoặc data corruption khi resume.

### 13.1 Tổng quan — Tại sao STR khó trong AMP?

```
Trong SMP đơn giản (chỉ có A55):
  suspend = tắt Core1..3 → Core0 sleep → done

Trong AMP (A55 + M7 + M33):
  Vấn đề 1: M7 đang chạy real-time task (CAN/audio)
             → không thể tắt đột ngột → phải handshake trước
  Vấn đề 2: Shared memory (vring) trong DDR
             → DDR phải được tắt sau cùng
             → Resume phải init lại DDR trước khi ai dùng
  Vấn đề 3: MU registers mất state khi power gate
             → Phải save/restore MU config
  Vấn đề 4: M33 là người duy nhất có thể tắt DDR
             → A55 phải "xin phép" M33 trước khi sleep
  Vấn đề 5: ATF BL31 phải còn resident trong RAM
             → Không thể power gate RAM region chứa ATF
             → Nhưng cần save/restore ATF context
```

### 13.2 Timeline STR đầy đủ — Suspend path

```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
 TRIGGER: echo mem > /sys/power/state
          hoặc Android power button → kernel suspend
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

[Linux, Core0]  pm_suspend(PM_SUSPEND_MEM)
    │
    ├─ BƯỚC 1: Freeze userspace
    │   freeze_processes()
    │   → Stop tất cả Android apps, services
    │   → Chỉ còn kernel threads chạy
    │
    ├─ BƯỚC 2: Suspend devices (theo dependency tree, reverse order)
    │   dpm_suspend_start()
    │   │
    │   ├─ Audio devices:
    │   │   SAI3 runtime suspend → clock off qua SCMI
    │   │   TAS5828 suspend      → I2C shutdown command
    │   │   PCM1808 suspend
    │   │
    │   ├─ RPMsg / Remoteproc suspend:
    │   │   rproc_suspend() ──────────────────────────────────┐
    │   │                                                      │
    │   │   Gửi SUSPEND notification tới M7 qua RPMsg:        │
    │   │   rpmsg_send(ept, "SUSPEND", 7)                      │
    │   │   Chờ ACK từ M7 (timeout 1s)                        │
    │   │                                          [M7 side]  │
    │   │                                          Nhận SUSPEND│
    │   │                                          → Dừng RT  │
    │   │                                            tasks     │
    │   │                                          → Flush     │
    │   │                                            pending   │
    │   │                                            data      │
    │   │                                          → Gửi ACK  │
    │   │                                          → WFI loop │
    │   │                                                      │
    │   │   Linux nhận ACK ◄────────────────────────────────── │
    │   │   → M7 đã ở trạng thái safe để power gate
    │   │
    │   ├─ MU driver suspend:
    │   │   imx_mu_suspend() → save MU_CR, MU_CCR registers
    │   │
    │   ├─ Clock framework suspend:
    │   │   Gửi SCMI CLOCK_CONFIG requests → disable clocks
    │   │
    │   └─ GIC suspend: gic_cpu_pm_notifier → save distributor state
    │
    ├─ BƯỚC 3: Suspend Core1..3 (CPU hotplug down)
    │   cpuhp_tasks_frozen = true
    │   cpu_down(3) → PSCI CPU_OFF → Core3 power gate
    │   cpu_down(2) → Core2 power gate
    │   cpu_down(1) → Core1 power gate
    │   Chỉ còn Core0 chạy
    │
    ├─ BƯỚC 4: Core0 gọi PSCI SYSTEM_SUSPEND
    │   psci_system_suspend() → SMC instruction → trap EL3
    │
    ▼ [ATF BL31, EL3]
    │
    ├─ BƯỚC 5: ATF xử lý SYSTEM_SUSPEND
    │   imx95_system_suspend():
    │   │
    │   ├─ Save CPU context: registers, EL1/EL2 system regs
    │   │   (SCTLR_EL1, TTBR0/1_EL1, TCR_EL1, MAIR_EL1, VBAR_EL1,
    │   │    SP_EL0/1, ELR_EL1, SPSR_EL1...)
    │   │
    │   ├─ Flush L1/L2/L3 cache (clean + invalidate)
    │   │   dcsw_op_all(DCCSW)  ← flush toàn bộ cache xuống DDR
    │   │   Quan trọng: đảm bảo tất cả data trong cache
    │   │   đã xuống DDR trước khi tắt DDR
    │   │
    │   ├─ Gửi SCMI SYSTEM_SUSPEND tới M33 qua MU
    │   │   M33 nhận → sẽ tắt DDR sau khi A55 sleep
    │   │
    │   ├─ Disable A55 cluster power (GPC)
    │   ├─ Disable L3 cache (DSU)
    │   │
    │   └─ Execute WFI (Wait For Interrupt)
    │      A55 Core0 = powered down / clock gated
    │
    ▼ [M33 System Manager]
    │
    ├─ BƯỚC 6: M33 hoàn tất suspend sequence
    │   (Sau khi nhận SCMI SYSTEM_SUSPEND và verify A55 đã sleep)
    │   │
    │   ├─ Tắt DDR self-refresh sequence:
    │   │   → Send MR13 command: enter self-refresh (SR)
    │   │   → DDR data preserved, clock stopped
    │   │   → DDRPHY power gate
    │   │
    │   ├─ Power gate các domain không cần thiết:
    │   │   PD_A55_CLUSTER, PD_GPU, PD_DISPLAY, PD_PCIE...
    │   │
    │   ├─ Switch system clock tới low-power oscillator (32kHz)
    │   │
    │   └─ M33 enters low-power state (chỉ giữ:)
    │       - RTC active (timekeeping)
    │       - PMIC communication (power management)
    │       - Wake source monitoring (GPIO, RTC alarm, CAN wakeup)
    │       - M7 trong WFI (nếu cần wake từ CAN)

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
 STEADY STATE: SUSPENDED
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
  A55 (4 cores):  Powered OFF — state saved trong DDR
  M7:             WFI — có thể wake theo CAN/LIN event
  M33:            Low-power monitor mode
  DDR:            Self-refresh — data preserved, ~1-5mW
  ELE:            Minimal active (RTC, tamper detect)
  ATF context:    Saved trong DDR (trong secure carveout)
  Vring buffers:  Intact trong DDR self-refresh
```

### 13.3 Timeline STR — Resume path

```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
 WAKE EVENT: GPIO interrupt / RTC alarm / CAN frame / USB
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

[M33]  Detect wake source
    │
    ├─ BƯỚC 1: M33 resume sequence
    │   → Switch clock back to high-speed PLL
    │   → Power up DDR domain
    │   → DDR exit self-refresh:
    │       - Power up DDRPHY
    │       - Exit self-refresh (send DDRPHY init sequence)
    │       - DDR training (nếu cần — tùy config)
    │       - DDR fully accessible
    │   → Power up A55 domain
    │   → Release A55 Core0 reset
    │
    ▼ [A55 Core0 wakes — nhảy thẳng vào ATF BL31]
    │   (Không qua BootROM, không qua SPL/U-Boot)
    │   Lý do: resume vector address được lưu trước khi suspend
    │
    ├─ BƯỚC 2: ATF BL31 resume handler
    │   imx95_system_resume():
    │   │
    │   ├─ Restore CPU context từ DDR:
    │   │   Restore SCTLR_EL1, TTBR0/1, TCR, MAIR, VBAR...
    │   │   Restore stack pointers, ELR, SPSR
    │   │
    │   ├─ Re-enable MMU (page tables vẫn còn trong DDR)
    │   │
    │   ├─ Invalidate TLB, I-cache
    │   │   (cache state không còn valid sau khi power gated)
    │   │
    │   └─ Return to Linux (EL1) tại điểm suspend
    │      cpu_resume() trong arch/arm64/kernel/sleep.S
    │
    ▼ [Linux Core0, tiếp tục từ điểm WFI]
    │
    ├─ BƯỚC 3: Resume Core1..3
    │   PSCI CPU_ON → wake Core1..3
    │   Các core join lại SMP cluster
    │
    ├─ BƯỚC 4: Resume devices (reverse of suspend, thuận chiều)
    │   dpm_resume_end()
    │   │
    │   ├─ GIC resume: restore distributor state
    │   │
    │   ├─ MU driver resume:
    │   │   Restore MU_CR, MU_CCR registers
    │   │   (channel config, interrupt enables)
    │   │
    │   ├─ SCMI resume:
    │   │   Re-negotiate với M33 SCMI server
    │   │   Restore clock rates, power domains
    │   │
    │   ├─ Remoteproc resume:
    │   │   rproc_resume()
    │   │   Gửi RESUME notification tới M7 qua RPMsg
    │   │   (Vring buffers vẫn nguyên trong DDR — không cần reinit)
    │   │   Chờ M7 ACK
    │   │                                          [M7 side]
    │   │                                          Nhận RESUME
    │   │                                          → Restart RT tasks
    │   │                                          → Gửi ACK
    │   │
    │   └─ Audio devices resume:
    │       SAI3 runtime resume → clock on qua SCMI
    │       TAS5828 resume → I2C reinit sequence
    │
    └─ BƯỚC 5: Thaw userspace
        thaw_processes()
        → Android apps, services tiếp tục chạy
        → Display on, audio available

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
 FULLY RESUMED — hệ thống hoạt động bình thường
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
 Tổng thời gian resume: ~200-500ms (tùy DDR training + driver resume)
```

### 13.4 CPU Context Save/Restore — ATF làm gì cụ thể

Đây là điểm mấu chốt: **tại sao resume không qua BootROM/SPL/U-Boot** mà nhảy thẳng vào kernel?

```c
/*
 * File: arch/arm64/kernel/sleep.S
 * __cpu_suspend_enter() — Core0 gọi trước khi WFI
 */
ENTRY(__cpu_suspend_enter)
    /* Save callee-saved registers lên stack */
    stp     x19, x20, [sp, #-16]!
    stp     x21, x22, [sp, #-16]!
    stp     x23, x24, [sp, #-16]!
    stp     x25, x26, [sp, #-16]!
    stp     x27, x28, [sp, #-16]!
    stp     x29, lr,  [sp, #-16]!

    /* Save stack pointer */
    mov     x9, sp
    str     x9, [x0]           /* x0 = &cpu_suspend_ctx[cpu_id] */

    /* Gọi __cpu_suspend_exit_address để save resume pointer */
    adr     x9, cpu_resume     /* địa chỉ resume function */
    str     x9, [x0, #8]

    /* Flush stack ra DDR (cache clean) */
    bl      __flush_dcache_area

    ret                        /* return tới PSCI call → WFI */
ENDPROC(__cpu_suspend_enter)
```

```c
/*
 * ATF save system registers (EL1/EL2 context)
 * File: lib/el3_runtime/aarch64/context.S (TF-A)
 */
func cm_el1_sysregs_context_save
    /* Save tất cả EL1 system registers */
    mrs  x9,  spsr_el1
    mrs  x10, elr_el1
    stp  x9,  x10, [x0, #CTX_SPSR_EL1]  /* save suspend point */

    mrs  x9,  sctlr_el1   /* system control (MMU enable, cache...) */
    mrs  x10, cpacr_el1
    stp  x9,  x10, [x0, #CTX_SCTLR_EL1]

    mrs  x9,  ttbr0_el1   /* page table base 0 */
    mrs  x10, ttbr1_el1   /* page table base 1 */
    stp  x9,  x10, [x0, #CTX_TTBR0_EL1]

    mrs  x9,  tcr_el1     /* translation control */
    mrs  x10, esr_el1
    stp  x9,  x10, [x0, #CTX_TCR_EL1]

    mrs  x9,  mair_el1    /* memory attribute indirection */
    mrs  x10, vbar_el1    /* vector base address */
    stp  x9,  x10, [x0, #CTX_MAIR_EL1]

    /* ... và nhiều reg khác ... */
    ret
endfunc cm_el1_sysregs_context_save
```

```c
/*
 * Resume vector — nơi Core0 nhảy vào sau khi M33 release reset
 * ATF đã setup resume_addr trong GPC (General Power Controller)
 * trước khi WFI
 */
func imx95_suspend
    /* Save context như trên */
    bl  cm_el1_sysregs_context_save

    /* Ghi resume vector vào GPC_CPU_PD_CTRL
     * M33 sẽ đọc địa chỉ này để biết phải jump về đâu */
    ldr  x0, =RESUME_ENTRY     /* = địa chỉ của plat_resume */
    str  x0, [GPC_BASE, GPC_CPU0_WAKEUP_RESUME_ADDR]

    dsb  sy
    isb

    wfi   /* ← Core0 stop ở đây */
    /* Sau resume: tiếp tục từ đây */
    b    plat_resume
endfunc imx95_suspend

func plat_resume
    bl  cm_el1_sysregs_context_restore  /* restore registers */
    bl  el3_exit                         /* return to EL1 kernel */
endfunc plat_resume
```

### 13.5 DDR Self-Refresh — M33 làm gì với DDR

```
DDR states trong STR:

ACTIVE (normal):
  DDR clock: 3200 MHz (LPDDR5)
  Power:     ~800mW
  Latency:   ~10ns

                    suspend trigger
                         │
                         ▼
SELF-REFRESH:
  Sequence M33 thực hiện:
  1. Flush A55 cache (ATF đã làm ở bước trước)
  2. Gửi DDR command: MR13[6]=1 (enter self-refresh)
  3. Wait tACPDEN (tPDEN)
  4. Assert CKE low (disable DDR clock)
  5. Power gate DDRPHY
  6. Optionally: gate DRAM VDD2 (deeper power save)

  DDR clock: OFF
  Data:      Preserved bởi DRAM capacitors (refresh internally)
  Power:     ~1-5mW
  Duration:  Up to hours/days (tREFI self-managed by DRAM)

                    wake event
                         │
                         ▼
EXIT SELF-REFRESH:
  1. Power up DDRPHY, restore VDD2
  2. Assert CKE high (enable clock)
  3. Wait tXS (exit self-refresh latency ~200ns)
  4. Send DRAM MRS commands (restore mode registers)
  5. Optional: DDRPHY re-training (nếu temperature drift)
  6. DDR ACTIVE — accessible by A55

ACTIVE (resumed):
  Identical to before suspend
  Vring buffers, kernel data, ATF context: all intact
```

### 13.6 M7 Suspend/Resume — 3 kịch bản thực tế

**Kịch bản 1: M7 suspend hoàn toàn (không cần wake từ M7)**

```c
/* Linux suspend: thông báo M7 suspend */
static int rproc_suspend_notifier(struct notifier_block *nb,
                                   unsigned long action, void *data)
{
    if (action == PM_SUSPEND_PREPARE) {
        /* Gửi SUSPEND command xuống M7 */
        struct rpmsg_suspend_msg msg = { .cmd = RPMSG_CMD_SUSPEND };
        rpmsg_send(suspend_ept, &msg, sizeof(msg));

        /* Chờ M7 ACK — timeout 1 giây */
        if (!wait_for_completion_timeout(&m7_suspended,
                                          msecs_to_jiffies(1000))) {
            dev_err(dev, "M7 suspend timeout!\n");
            return NOTIFY_BAD;  /* block suspend nếu M7 không phản hồi */
        }
    }
    return NOTIFY_OK;
}

/* M7 FreeRTOS suspend handler */
void handle_suspend_cmd(void)
{
    /* Dừng tất cả RT tasks */
    vTaskSuspendAll();

    /* Flush pending CAN/audio data */
    can_flush_tx_queue();
    audio_drain_buffer();

    /* Gửi ACK về Linux */
    struct rpmsg_ack_msg ack = { .cmd = RPMSG_CMD_SUSPEND_ACK };
    rpmsg_lite_send(inst, ept, LINUX_EPT_ADDR, &ack, sizeof(ack), RL_BLOCK);

    /* M7 enters WFI — clock gated by M33 sau đó */
    __WFI();
    /* Resume: M33 sẽ assert interrupt để wake M7 */
}
```

**Kịch bản 2: M7 giữ tỉnh để wake system (ví dụ CAN wakeup)**

```c
/*
 * Automotive use case: xe đang suspend, nhận CAN frame "remote start"
 * M7 cần xử lý CAN và quyết định có wake toàn hệ thống không
 */

/* Linux thông báo M7: "suspend nhưng giữ CAN monitor" */
struct rpmsg_suspend_msg msg = {
    .cmd   = RPMSG_CMD_SUSPEND,
    .flags = SUSPEND_FLAG_KEEP_CAN_ACTIVE,  /* M7 không sleep hoàn toàn */
};
rpmsg_send(suspend_ept, &msg, sizeof(msg));

/* M7 FreeRTOS — CAN wakeup task */
void m7_can_wakeup_task(void *param)
{
    CAN_msg_t frame;

    while (1) {
        /* M7 vẫn chạy CAN task dù A55 đang suspend */
        if (CAN_receive(&frame, portMAX_DELAY) == pdTRUE) {

            if (is_wakeup_frame(&frame)) {
                /* Quyết định wake toàn hệ thống */
                /* Báo M33 via HSEM hoặc MU M7-M33 */
                M33_notify_system_wakeup();

                /* M33 sẽ release A55 reset */
                /* A55 resume từ ATF resume vector */
            }
        }
    }
}
```

**Kịch bản 3: Audio wakeup (wake word detection trên M7)**

```
Automotive: hệ thống suspend, nhưng M7 chạy low-power audio DSP
  → Capture từ mic (PCM1808) liên tục
  → Run wake word detection (nhỏ gọn, ~10mW)
  → Detect "Hey Car" → wake toàn hệ thống
  → Android tiếp quản xử lý voice command

Power breakdown:
  A55 suspended:   0mW
  DDR self-refresh: 3mW
  M33 monitor:     1mW
  M7 audio DSP:    15mW
  MEMS mic + ADC:  2mW
  Total:           ~21mW vs 800mW khi full active
```

### 13.7 Source code Linux — suspend/resume hooks

```c
/*
 * Platform suspend ops — i.MX95
 * File: arch/arm64/mach-imx/pm-imx95.c
 */
static int imx95_pm_prepare(void)
{
    /* Thông báo tất cả driver chuẩn bị suspend */
    /* Verify M7 đã acknowledge */
    return imx95_check_m7_suspended();
}

static int imx95_pm_enter(suspend_state_t state)
{
    switch (state) {
    case PM_SUSPEND_MEM:  /* STR */
        /* Bước cuối trên Linux trước khi gọi ATF */
        cpu_pm_enter();           /* notify CPU PM framework */
        cpu_suspend(0,            /* arg = power state */
                    imx95_cpu_pm_finish);  /* resume callback */
        cpu_pm_exit();
        break;
    }
    return 0;
}

static void imx95_pm_finish(void)
{
    /* Chạy ngay sau khi resume từ ATF */
    /* Restore GIC state, timers */
}

static const struct platform_suspend_ops imx95_suspend_ops = {
    .prepare     = imx95_pm_prepare,
    .enter       = imx95_pm_enter,
    .finish      = imx95_pm_finish,
    .valid       = suspend_valid_only_mem,  /* chỉ support MEM (STR) */
};

/* Đăng ký trong init */
suspend_set_ops(&imx95_suspend_ops);
```

```c
/*
 * Driver suspend/resume example — SAI audio driver
 * File: sound/soc/fsl/fsl_sai.c
 */
static int fsl_sai_suspend(struct device *dev)
{
    struct fsl_sai *sai = dev_get_drvdata(dev);

    /* Save register state trước khi power gate */
    regcache_cache_only(sai->regmap, true);
    regcache_mark_dirty(sai->regmap);

    /* Disable clocks */
    clk_disable_unprepare(sai->bus_clk);
    clk_disable_unprepare(sai->mclk_clk[0]);

    return 0;
}

static int fsl_sai_resume(struct device *dev)
{
    struct fsl_sai *sai = dev_get_drvdata(dev);

    /* Re-enable clocks trước */
    clk_prepare_enable(sai->bus_clk);
    clk_prepare_enable(sai->mclk_clk[0]);

    /* Sync lại registers từ cache xuống hardware */
    regcache_cache_only(sai->regmap, false);
    regcache_sync(sai->regmap);  /* write lại toàn bộ regs */

    return 0;
}

static const struct dev_pm_ops fsl_sai_pm_ops = {
    .suspend = fsl_sai_suspend,
    .resume  = fsl_sai_resume,
    /* Runtime PM (idle-time) */
    SET_RUNTIME_PM_OPS(fsl_sai_runtime_suspend,
                       fsl_sai_runtime_resume, NULL)
};
```

### 13.8 Common bugs trong STR — những lỗi hay gặp

```
BUG 1: M7 không ACK suspend trong timeout
───────────────────────────────────────────
Triệu chứng: dmesg "M7 suspend timeout", system không suspend
Nguyên nhân: M7 đang xử lý CAN burst / audio frame lớn
Fix: tăng timeout, hoặc M7 task check suspend flag thường xuyên hơn

BUG 2: Kernel panic khi resume — NULL pointer
──────────────────────────────────────────────
Triệu chứng: crash tại driver X khi resume
Nguyên nhân: driver không restore registers đúng thứ tự
             (clock chưa on nhưng đã access registers)
Fix: đảm bảo clock enable TRƯỚC khi regcache_sync()

BUG 3: Audio glitch sau resume
────────────────────────────────
Triệu chứng: tiếng pop/click khi audio resume
Nguyên nhân: TAS5828 power sequence sai sau resume
             (SAI clock chạy trước khi TAS5828 init xong)
Fix: thêm delay hoặc sync point giữa SAI resume và codec resume

BUG 4: RPMsg message mất sau resume
─────────────────────────────────────
Triệu chứng: HAL không nhận được reply từ M7 sau wake
Nguyên nhân: MU interrupt bị mất khi MU power gate
             → M7 đã gửi message nhưng Linux không nhận interrupt
Fix: trong MU resume, check và drain RX registers trước khi
     enable interrupts (có thể có pending data từ trước suspend)

BUG 5: DDR data corruption sau resume
───────────────────────────────────────
Triệu chứng: random kernel crash, memory errors
Nguyên nhân: Cache không được flush hoàn toàn trước khi DDR SR
             → cache còn dirty lines khi DDR vào self-refresh
             → Resume: DDR data cũ, không match với cache state
Fix: Verify ATF flush sequence: dcsw_op_all(DCCSW) phải complete
     trước khi M33 được phép đưa DDR vào self-refresh

BUG 6: Wakeup source không hoạt động
──────────────────────────────────────
Triệu chứng: board suspend xong không thể wake bằng GPIO
Nguyên nhân: wakeup GPIO chưa được enable_irq_wake() trước suspend
Fix:
  enable_irq_wake(gpio_irq);   /* trước khi suspend */
  /* hoặc trong DTS: */
  wakeup-source;               /* property trên GPIO node */
```

### 13.9 Debug STR trên EVK95

```bash
# === Trigger suspend ===
adb shell "echo mem > /sys/power/state"

# === Xem wake sources được enable ===
adb shell cat /sys/power/wakeup_count
adb shell cat /sys/kernel/debug/wakeup_sources

# === Xem suspend stats ===
adb shell cat /sys/kernel/debug/suspend_stats
# Output:
# success: 5         ← số lần suspend thành công
# fail: 1            ← số lần fail
# failed_freeze: 0
# failed_prepare: 0
# failed_suspend: 1  ← fail ở device suspend phase
# failed_resume: 0

# === Xem driver nào chặn suspend ===
adb shell dmesg | grep -i "suspend\|resume\|pm:"
# [  123.456] PM: suspend entry (deep)
# [  123.457] PM: Syncing filesystems
# [  123.458] Freezing user space processes
# [  123.600] PM: suspend devices
# [  123.700] imx-rproc: waiting for M7 suspend ACK...
# [  123.800] imx-rproc: M7 suspended OK

# === Wakeup reason sau resume ===
adb shell dmesg | grep -i "wake\|wakeup" | tail -20
# [  456.123] PM: resume from suspend
# [  456.124] imx-mu: MU1 wakeup triggered
# [  456.125] gpio-keys: power button pressed

# === Enable PM debug verbose ===
adb shell "echo 1 > /sys/module/pm_debug_messages/parameters/pm_debug_messages"
adb shell "echo 1 > /sys/power/pm_print_times"
# Sau đó suspend/resume → log sẽ show thời gian từng driver suspend/resume

# === Xem thời gian resume từng driver ===
adb shell dmesg | grep "PM: resume" | head -30
# PM: resume of devices complete after 245.123 msecs

# === Test STR loop (stress test) ===
adb shell "
for i in $(seq 1 10); do
  echo mem > /sys/power/state
  sleep 3
  echo Resume $i
done
"
```

### 13.10 Sơ đồ tổng hợp STR

```
SUSPEND:                              RESUME:
                                       (wake event)
Android Apps         frozen ─────────► thawed
     │                                     ▲
     ▼                                     │
Linux Drivers        suspend ────────► resume
(SAI,TAS5828,MU,     (reverse           (forward
 SCMI,RPMsg...)       order)             order)
     │                                     ▲
     ▼                                     │
M7 FreeRTOS   ─── SUSPEND msg ──────► RESUME msg
                   ACK ◄───               ACK ──►
     │             WFI                restart tasks
     ▼                                     ▲
A55 Core1..3         CPU_OFF ───────► CPU_ON (PSCI)
     │                                     ▲
     ▼                                     │
A55 Core0            WFI ───────────► cpu_resume()
     │               (via ATF)             ▲
     ▼                                     │
ATF BL31      save ctx + flush cache ─► restore ctx
     │         ghi resume vector           ▲
     ▼                                     │
M33 SM        DDR → self-refresh ───► DDR exit SR
              power gate domains       power up
              32kHz clock              PLL restart
     │                                     ▲
     ▼                                     │
DDR            ACTIVE → SR ─────────► SR → ACTIVE
               (data preserved)        (data intact)
     │                                     ▲
WAKEUP SOURCE ─────────────────────────────┘
(GPIO/RTC/CAN)        M33 detects wake event
```

---
-e 
*Tài liệu tổng hợp cho NXP i.MX95 EVK + Android Automotive AOSP.*
*Version 3.0 — Bổ sung: STR full multi-core coordination, Boot timeline, MU hardware, Virtio layer.*
*Địa chỉ memory và clock IDs cần verify với i.MX95 Reference Manual.*
