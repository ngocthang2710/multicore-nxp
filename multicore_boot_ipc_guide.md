# Multi-Core Boot & IPC — Tài liệu chuyên sâu
## Dành cho: Embedded Android Automotive / NXP i.MX95 / Renesas R-Car

---

## Mục lục

1. [Hướng 1 — SMP Boot Flow (ARM64 Kernel)](#hướng-1--smp-boot-flow-arm64-kernel)
2. [Hướng 2 — ATF/TF-A & PSCI](#hướng-2--atftf-a--psci)
3. [Hướng 3 — AMP với Remoteproc + RPMsg (Quan trọng nhất)](#hướng-3--amp-với-remoteproc--rpmsg)
4. [Kiến trúc tổng quan i.MX95](#kiến-trúc-tổng-quan-imx95)
5. [Practical Lab — Code thực tế](#practical-lab)

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

## Tham khảo

| Tài liệu | Link |
|---|---|
| NXP i.MX95 Reference Manual | NXP website → i.MX 95 |
| Linux Remoteproc docs | `Documentation/staging/remoteproc.rst` |
| RPMsg-Lite (MCU SDK) | github.com/nxp-mcuxpresso/rpmsg-lite |
| ARM PSCI spec | developer.arm.com/documentation/den0022 |
| TF-A source | git.trustedfirmware.org/TF-A/trusted-firmware-a |
| Virtio spec (vring) | docs.oasis-open.org/virtio/virtio/v1.2 |

---

*Tài liệu này được tổng hợp cho NXP i.MX95 + Android Automotive AOSP.*
*Các địa chỉ memory, clock IDs cần verify với Reference Manual của từng board cụ thể.*
