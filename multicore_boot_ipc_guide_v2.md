# Multi-Core Boot & IPC — Tài liệu chuyên sâu
## Dành cho: Embedded Android Automotive / NXP i.MX95 / Renesas R-Car

---

## Mục lục

**Phần cốt lõi:**
1. [Hướng 1 — SMP Boot Flow (ARM64 Kernel)](#hướng-1--smp-boot-flow-arm64-kernel)
2. [Hướng 2 — ATF/TF-A & PSCI](#hướng-2--atftf-a--psci)
3. [Hướng 3 — AMP với Remoteproc + RPMsg](#hướng-3--amp-với-remoteproc--rpmsg)
4. [Kiến trúc tổng quan i.MX95](#kiến-trúc-tổng-quan-imx95)
5. [Practical Lab](#practical-lab)

**Phần nâng cao (i.MX95 chuyên sâu):**

6. [Chương 4 — SCMI: Linux giao tiếp với System Manager](#chương-4--scmi-system-control--management-interface)
7. [Chương 5 — Cache Coherency & Shared Memory](#chương-5--cache-coherency--shared-memory-giữa-các-core)
8. [Chương 6 — ELE (EdgeLock Enclave) — Security Core](#chương-6--ele-edgelock-enclave--security-core)
9. [Chương 7 — Power Management & Power Domains](#chương-7--power-management--power-domains)
10. [Chương 8 — Inter-Core Debugging Chuyên sâu](#chương-8--inter-core-debugging-chuyên-sâu)
11. [Chương 9 — Cortex-M33 (System Manager)](#chương-9--cortex-m33-system-manager--core-thứ-3-của-evk95)
12. [Chương 10 — CPU Frequency Scaling (DVFS)](#chương-10--cpu-frequency-scaling-dvfs--thực-tế-với-android)
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

*Tài liệu tổng hợp cho NXP i.MX95 EVK + Android Automotive AOSP.*
*Version 2.0 — Bổ sung: SCMI, Cache Coherency, ELE, Power Domains, M33, HSEM, DVFS, Safety.*
*Địa chỉ memory và clock IDs cần verify với i.MX95 Reference Manual (Rev.1, Chapter 3-5).*
