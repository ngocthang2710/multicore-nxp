# NXP i.MX95 EVK — A/B OTA, Power & Thermal Management
> Phần 5–6 của tài liệu NXP i.MX95 Deep-Dive

---

## 5. Cơ Chế Cập Nhật A/B (Seamless OTA & Fallback Rollback)

### 5.1 Kiến Trúc Phân Vùng A/B

Android A/B (còn gọi là Seamless Update) nhân đôi tất cả partition có thể boot/update. Không có partition `recovery` riêng — recovery mode chạy từ chính `boot_a`/`boot_b` với `androidboot.force_normal_boot=0`.

```
eMMC A/B Partition Layout (i.MX95 EVK):

  Physical Partition   | Slot A           | Slot B           | Shared
  ─────────────────────┼──────────────────┼──────────────────┼─────────────
  eMMC boot0           | flash.bin (A)    | flash.bin (B)    | (same bin)
  GPT User Area:       |                  |                  |
    boot               | boot_a           | boot_b           |
    vendor_boot        | vendor_boot_a    | vendor_boot_b    |
    dtbo               | dtbo_a           | dtbo_b           |
    vbmeta             | vbmeta_a         | vbmeta_b         |
    vbmeta_system      | vbmeta_system_a  | vbmeta_system_b  |
    vbmeta_vendor      | vbmeta_vendor_a  | vbmeta_vendor_b  |
    super (LP)         | system_a         | system_b         |
                       | vendor_a         | vendor_b         |
                       | product_a        | product_b        |
                       | system_ext_a     | system_ext_b     |
  ─────────────────────┼──────────────────┼──────────────────┼─────────────
    misc               |                  |                  | BCB (shared)
    metadata           |                  |                  | LP metadata
    userdata           |                  |                  | /data
    persist            |                  |                  | /mnt/vendor/persist

Slot State Machine:
  UNBOOTABLE  ← new slot trước khi OTA
  BOOTABLE    ← OTA hoàn thành ghi xong
  ACTIVE      ← đang được chọn để boot
  SUCCESSFUL  ← đã boot thành công lên userspace + set_active

  Transitions:
    OTA ghi xong     → set_unbootable(B) → mark_bootable(B)
    Reboot OTA       → set_active(B)     [tries_remaining = 7]
    Boot B thành công → mark_successful(B)
    Boot B thất bại  → tries_remaining-- → khi = 0: B = UNBOOTABLE
    Rollback         → set_active(A)
```

**Boot Control Block (BCB) & Misc Partition:**

```
Misc partition layout (64KB, offset: GPT partition "misc"):

Byte offset  Field                 Size    Description
──────────────────────────────────────────────────────────────
0x000        bootloader_message    2048B   BCB v1 (legacy)
0x800        slot_suffix           32B     "\0" = slot A, "_b" = slot B  ← U-Boot reads
0x820        update_channel        128B    OTA channel name
0x900        stage                 32B     Recovery stage counter
0x920        reserved              ...

BCB (Boot Control Block) structure tại offset 0:
  struct bootloader_message {
      char command[32];      // "boot-recovery", "update-radio", ""
      char status[32];       // "OKAY", "failed-update"
      char recovery[1024];   // Recovery args
      char stage[32];        // multi-stage install: "1/3", "2/3"
      char reserved[1184];
  };

BCBV2 (Android 11+ boot_control v2) — tại offset 0x800:
  struct bootloader_control {
      char magic[4];           // "BCTV" (0x42435456)
      uint8_t version;         // 1 hoặc 2
      uint8_t nb_slot;         // 2
      uint16_t crc32_le;       // CRC của struct này
      struct slot_metadata {
          uint8_t priority   : 4;  // 0-15, 15=highest priority
          uint8_t tries_remaining: 3; // 0-7, 7=fresh OTA slot
          uint8_t successful_boot: 1; // 1=đã boot thành công
          uint8_t verity_corrupted: 1;
          uint8_t reserved: 7;
      } slot_info[4];          // slots A, B, C, D (max 4)
      uint8_t recovery_tries_remaining; // cố gắng boot recovery
      uint8_t reserved[7];
  };
```

### 5.2 Luồng Giao Tiếp U-Boot ↔ BCB ↔ boot_control HAL

**Android side (Java/C++ HAL → misc partition):**

```
hardware/interfaces/boot/
├── aidl/
│   └── android/hardware/boot/
│       ├── IBootControl.aidl       ← AIDL interface (Android 13+)
│       └── MergeStatus.aidl
└── default/
    ├── BootControl.cpp             ← Reference implementation
    └── libboot_control/
        └── boot_control.cpp        ← Core logic: read/write BCB

bootable/recovery/
└── boot_control/
    └── BootControlImx.cpp          ← NXP i.MX implementation

system/update_engine/
├── update_engine_daemon.cc         ← OTA engine main (download + apply)
├── payload_consumer/
│   └── delta_performer.cc          ← Apply payload: write partitions
├── boot_control_android.cc         ← Calls boot_control HAL
└── update_attempter_android.cc     ← State machine: Idle→Downloading→Applying
```

**Luồng OTA từ đầu đến cuối:**

```
1. [update_engine] Download payload.bin từ OTA server
   → Verify signature (ECDSA P-256 của Google/OEM)
   → Parse payload manifest (protobuf)

2. [update_engine] Ghi partition:
   for each partition in manifest:
       if partition == "boot":
           open /dev/block/by-name/boot_b  ← slot inactive
           write delta/full image
       if partition == "system":
           lp: resize/create system_b in super
           write via block device

3. [boot_control HAL] Mark slot B bootable:
   BootControl::setSlotAsUnbootable(1)       // clear B first
   BootControl::markBootSuccessful()         // not yet
   
   /* Write BCB to misc: */
   android::bootable::SetMiscPartitionContents(
       "misc", &bootloader_message_raw);
   
   /* struct bootloader_control update: */
   abc.slot_info[1].priority = 15;           // B = highest priority
   abc.slot_info[1].tries_remaining = 7;     // 7 attempts
   abc.slot_info[1].successful_boot = 0;     // not yet successful
   abc.slot_info[0].priority = 14;           // A slightly lower
   
   /* Write to /dev/block/by-name/misc at offset 0x800 */
   WriteMiscPartitionControlFields(abc);

4. [update_engine] Trigger reboot:
   PowerManager.reboot("update")
   → Android reboots → U-Boot
```

**U-Boot BCB Parsing (Critical Path):**

```c
/* boot/android/ab.c (U-Boot android_ab support) */

/* Source: u-boot/boot/android/ab.c */
int ab_select_slot(struct blk_desc *dev_desc,
                   struct disk_partition *misc_part_info,
                   bool normal_boot)
{
    struct bootloader_control abc;
    
    /* 1. Đọc BCB từ misc partition */
    /* misc_part_info->start = LBA start của "misc" partition */
    /* Offset trong misc: A/B control tại byte 0x800 → LBA offset = 4 */
    ret = blk_dread(dev_desc,
                    misc_part_info->start + BCB_OFFSET_LBA,
                    1,            /* 1 sector = 512 bytes */
                    &abc);
    
    /* 2. Validate magic và CRC32 */
    if (abc.magic != BOOT_CTRL_MAGIC) {
        /* "BCTV" magic không khớp → corrupt hoặc fresh device */
        /* Fallback: boot slot A */
        return 0;
    }
    crc = crc32(0, (uint8_t*)&abc, offsetof(typeof(abc), crc32_le));
    if (crc != abc.crc32_le) {
        printf("WARNING: BCB CRC mismatch, defaulting to slot A\n");
        return 0;
    }
    
    /* 3. Chọn slot có priority cao nhất và còn tries_remaining > 0 */
    int slot = -1;
    int highest_priority = -1;
    for (int i = 0; i < abc.nb_slot; i++) {
        if (abc.slot_info[i].tries_remaining == 0 &&
            !abc.slot_info[i].successful_boot) {
            continue;  /* UNBOOTABLE: skip */
        }
        if (abc.slot_info[i].priority > highest_priority) {
            highest_priority = abc.slot_info[i].priority;
            slot = i;
        }
    }
    if (slot < 0) {
        printf("ERROR: No bootable slot found!\n");
        return -ENOENT;  /* Both slots unbootable → fastboot mode */
    }
    
    /* 4. Decrement tries_remaining nếu chưa successful */
    if (!abc.slot_info[slot].successful_boot) {
        if (abc.slot_info[slot].tries_remaining > 0) {
            abc.slot_info[slot].tries_remaining--;
        }
        /* Ghi lại BCB với tries_remaining đã giảm */
        blk_dwrite(dev_desc,
                   misc_part_info->start + BCB_OFFSET_LBA,
                   1, &abc);
    }
    
    /* 5. Set suffix trong bootargs */
    /* slot=0 → "_a", slot=1 → "_b" */
    env_set("slot_suffix", slot == 0 ? "_a" : "_b");
    
    /* 6. Set active partition names */
    /* U-Boot boot command sẽ dùng: boot${slot_suffix} */
    printf("## Booting slot %s (priority=%d, tries_remaining=%d)\n",
           slot == 0 ? "A" : "B",
           abc.slot_info[slot].priority,
           abc.slot_info[slot].tries_remaining);
    
    return slot;
}

/* Hook trong U-Boot android bootflow: */
/* cmd/android_bootloader.c → android_boot_flow() */
int android_boot_flow(...)
{
    slot = ab_select_slot(dev_desc, &misc_partinfo, true);
    /* slot = 0 → boot boot_a, dtbo_a, vbmeta_a */
    /* slot = 1 → boot boot_b, dtbo_b, vbmeta_b */
    
    snprintf(boot_partition, sizeof(boot_partition),
             "boot%s", slot == 0 ? "_a" : "_b");
    /* → "boot_a" hoặc "boot_b" */
    
    /* Pass slot suffix to kernel: */
    /* androidboot.slot_suffix=_a (hoặc _b) */
    env_set_hex("androidboot.slot_suffix", slot == 0 ? "_a" : "_b");
}
```

**U-Boot Serial Log (A/B selection):**

```
## Android A/B Slot Selection:
   Reading BCB from misc partition (LBA 0x1400, offset 4)...
   BCB magic: BCTV OK, CRC: OK
   Slot A: priority=14, tries=7, successful=1
   Slot B: priority=15, tries=6, successful=0   ← OTA slot, tries-- (7→6)
   ## Booting slot B (priority=15, tries_remaining=6)
   slot_suffix = _b
   Booting partition: boot_b
   Loading boot_b from eMMC...
```

### 5.3 Watchdog & Fallback Rollback Mechanism

Đây là phần quan trọng nhất của A/B safety. Hệ thống cần đảm bảo nếu slot mới không boot được, slot cũ luôn có thể fallback.

#### Cơ Chế Watchdog Hardware (i.MX95 WDOG)

```
i.MX95 Watchdog instances:
  WDOG1: base 0x42490000  ← System watchdog (U-Boot + Kernel)
  WDOG2: base 0x424A0000  ← Secondary (cho M33 SM)
  WDOG3: base 0x424B0000  ← (unused / reserved)

WDOG1 Registers:
  WDOGx_WCR   (0x00): Control Register
    Bit[2] WDE:  Watchdog Enable (write-once!)
    Bit[3] WDT:  Watchdog Timeout (0=16s, 1=reset tạm thời)
    Bit[6] WT[3:0]: timeout period (0x0=0.5s, 0xF=128s)
  WDOGx_WSR   (0x02): Service Register → write 0x5555 then 0xAAAA to feed
  WDOGx_WRSR  (0x04): Reset Status Register
    Bit[0] SFTW: Software reset triggered this boot
    Bit[1] TOUT: Watchdog timeout triggered this boot ← Kiểm tra cái này!
  WDOGx_WICR  (0x06): Interrupt Control
  WDOGx_WMCR  (0x08): Misc Control (power-down counter)
```

**WDOG trong U-Boot — phát hiện watchdog reset:**

```c
/* arch/arm/mach-imx/imx9/soc.c */
void check_wdog_reset_source(void)
{
    uint16_t wrsr = readw(WDOG1_BASE_ADDR + WDOG_WRSR);
    
    if (wrsr & WDOG_WRSR_TOUT) {
        /* Watchdog timeout xảy ra trong lần boot trước */
        printf("WARNING: Previous boot terminated by WDOG1 timeout!\n");
        
        /* Đây là dấu hiệu slot hiện tại không boot được */
        /* U-Boot có thể chủ động mark slot current là failed */
        /* và switch sang slot kia */
        ab_mark_boot_failed();
    }
}

/* board/freescale/imx95_evk/imx95_evk.c */
int board_init(void)
{
    check_wdog_reset_source();
    
    /* Start WDOG với timeout 128s cho U-Boot phase */
    /* Kernel sẽ tiếp quản và rút ngắn timeout */
    wdog_start(WDOG1_BASE_ADDR, 128);  /* 128 giây */
    return 0;
}
```

**Linux Kernel Watchdog Driver (imx2_wdt):**

```c
/* drivers/watchdog/imx2_wdt.c */

/* IMX2 WDT driver — dùng cho WDOG1 trên i.MX95 */
struct imx2_wdt_device {
    struct clk *clk;
    struct regmap *regmap;
    struct watchdog_device wdog;
    bool ext_reset;
    bool clk_is_on;
    bool no_ping;
};

static int imx2_wdt_set_timeout(struct watchdog_device *wdog,
                                  unsigned int new_timeout)
{
    /* WCR WT field: timeout = (WT + 1) * 0.5 seconds */
    /* Max 128s, default 60s cho Android boot phase */
    u16 secs = clamp_t(u16, (new_timeout * 2 - 1), 0, 0xFF);
    regmap_update_bits(wdt->regmap, IMX2_WDT_WCR,
                       IMX2_WDT_WCR_WT, secs << IMX2_WDT_WCR_WT_SHIFT);
    return 0;
}

/* /dev/watchdog0 → userspace daemon phải ping thường xuyên */
/* Android: /system/bin/watchdogd */
```

**Android Watchdog Daemon (software layer):**

```
Có 2 tầng watchdog trên Android:

Tầng 1: Hardware WDOG (kernel /dev/watchdog0)
  → Được mở bởi: /system/bin/watchdogd
  → Feed interval: mỗi 30 giây
  → Timeout: 60 giây
  → Nếu watchdogd chết → WDOG reset → tries_remaining-- trong BCB

Tầng 2: Android Framework Watchdog (software)
  → com.android.server.Watchdog (Java, trong SystemServer)
  → Monitor: AMS, WMS, InputDispatcher threads
  → Timeout: 30 giây per service
  → Nếu service hang → trigger reboot (PowerManager.reboot)
  → Sau reboot: boot_control HAL set tries_remaining--
```

**Luồng Rollback Đầy Đủ:**

```
Scenario: OTA cài slot B, slot B bị Kernel Panic

Boot 1 (slot B, tries=7):
  U-Boot reads BCB → slot B priority=15, tries=7
  U-Boot decrements → tries=6, writes BCB
  U-Boot boots boot_b
  Kernel Panic tại board_init() hoặc trong driver init
  → WDOG1 timeout (không ai feed watchdog)
  → Hardware reset!

Boot 2 (slot B, tries=6):
  U-Boot checks WDOG_WRSR.TOUT bit → set!
  U-Boot reads BCB → slot B priority=15, tries=6
  U-Boot decrements → tries=5, writes BCB
  U-Boot boots boot_b lần nữa
  ...Kernel Panic lại...
  → WDOG timeout → reset

[Lặp lại đến khi tries=0]

Boot N (slot B, tries=0):
  U-Boot reads BCB:
    slot B: priority=15, tries=0, successful=0
    → tries_remaining=0 && !successful → UNBOOTABLE!
  U-Boot chọn slot A (priority=14, tries=7, successful=1)
  U-Boot boots boot_a ← Rollback thành công!
  Android boots → watchdogd starts → WDOG fed
  [No mark_successful needed, slot A already successful]

Serial log của rollback:
  ## Android A/B Slot Selection:
     Slot A: priority=14, tries=N/A (successful=1)
     Slot B: priority=15, tries=0, successful=0 → UNBOOTABLE, skip
     ## Falling back to slot A (last known good)
     WARNING: Slot B exhausted all boot attempts!
```

**Scenario 2: Boot Thành Công → mark_successful:**

```
Android User-space boot thành công:
  1. update_verifier chạy sau zygote start
     → Verify dm-verity cho system_b, vendor_b
     → Nếu OK: tiếp tục

  2. update_engine_client --mark_boot_successful
     → Gọi IBootControl::markBootSuccessful()
     → Ghi BCB: slot_info[1].successful_boot = 1
                slot_info[1].tries_remaining = 7  (reset)

  3. BCB sau khi successful:
     Slot A: priority=7,  tries=7,  successful=1  (demoted)
     Slot B: priority=15, tries=7,  successful=1  ← Active, Successful

Source:
  system/update_engine/update_attempter_android.cc
  → BootControlAndroid::MarkBootSuccessful()
  → hardware/interfaces/boot/aidl/default/BootControl.cpp
  → bootable/recovery/boot_control/BootControlImx.cpp
```

**Scenario 3: Kernel Panic sớm trước khi WDOG active:**

```c
/* Kernel Panic Handler */
/* kernel/panic.c */
void panic(const char *fmt, ...)
{
    /* 1. Print panic message */
    pr_emerg("Kernel panic - not syncing: %s\n", buf);
    
    /* 2. Dump stack, kmsg */
    dump_stack();
    
    /* 3. Reboot after panic_timeout (default 5s) */
    /* Settable via: echo 5 > /proc/sys/kernel/panic */
    /* Hoặc cmdline: panic=5 */
    
    if (panic_timeout > 0) {
        pr_emerg("Rebooting in %d seconds...\n", panic_timeout);
        mdelay(panic_timeout * 1000);
        
        /* Trigger hardware reset qua PSCI SYSTEM_RESET */
        arm_pm_restart(REBOOT_WARM, NULL);
        /* → ATF SMC: PSCI_SYSTEM_RESET */
        /* → M33 SM via SCMI executes SRC software reset */
    }
    /* Nếu panic_timeout = 0: hang forever → WDOG timeout sau 60s */
}
```

**Verify A/B state từ Android:**

```bash
# Kiểm tra slot hiện tại
adb shell getprop ro.boot.slot_suffix     # _a hoặc _b

# Kiểm tra trạng thái boot control
adb shell bootctl get-number-slots        # 2
adb shell bootctl get-current-slot        # 0 (=A) hoặc 1 (=B)
adb shell bootctl get-active-boot-slot    # slot được chọn lần tới
adb shell bootctl is-slot-bootable 0      # true/false
adb shell bootctl is-slot-marked-successful 0  # true/false
adb shell bootctl get-suffix 1            # _b

# Force rollback thủ công (test):
adb shell bootctl set-active-boot-slot 0  # Set A làm active cho boot tới
adb reboot

# Xem raw BCB:
adb shell dd if=/dev/block/by-name/misc bs=1 skip=2048 count=32 2>/dev/null | xxd
# offset 2048 (0x800) = bootloader_control struct start
# Byte 0-3: magic "BCTV" = 0x42 0x43 0x54 0x56
# Byte 6-7: CRC16
# Byte 8: slot A (priority[3:0] | tries[6:4] | successful[7])
# Byte 9: slot B

# update_engine log:
adb shell logcat -d | grep -i "update_engine\|bootcontrol\|slot\|rollback"
```

---

## 6. Quản Lý Năng Lượng & Nhiệt (Advanced Power & Thermal Management)

### 6.1 Kiến Trúc Linux Thermal Framework

```
Linux Thermal Framework Stack (kernel/drivers/thermal/):

  ┌──────────────────────────────────────────────────────────┐
  │           User Space / Android Thermal HAL               │
  │   /vendor/bin/hw/android.hardware.thermal-service.nxp   │
  │   Reads: /sys/class/thermal/thermal_zone*/temp           │
  │   Writes: /sys/class/thermal/thermal_zone*/policy        │
  └─────────────────┬────────────────────────────────────────┘
                    │ sysfs / ioctl (THERMAL_GENL_EVENT_*)
  ┌─────────────────▼────────────────────────────────────────┐
  │           Thermal Core (drivers/thermal/thermal_core.c)  │
  │   thermal_zone_device_register()                         │
  │   thermal_cooling_device_register()                      │
  │   Manages: trip points, governor state machine           │
  └──────┬────────────────────────┬────────────────────────--┘
         │                        │
  ┌──────▼──────┐         ┌───────▼──────────────────────────┐
  │  Thermal    │         │  Thermal Governors               │
  │  Zones      │         │  ├── step_wise (default Android) │
  │  (sensors)  │         │  ├── power_allocator (EAS-aware) │
  │             │         │  ├── user_space (HAL-driven)     │
  │  tz0: CPU0  │         │  └── bang_bang (hysteresis)      │
  │  tz1: CPU1  │         └──────────────────────────────────┘
  │  tz2: SoC   │
  │  tz3: DDR   │         ┌──────────────────────────────────┐
  │  tz4: GPU   │         │  Cooling Devices                 │
  │  tz5: NPU   │         │  ├── cpufreq_cooling (cpufreq)   │
  └──────┬──────┘         │  ├── devfreq_cooling (GPU/NPU)   │
         │                │  ├── thermal_pm_qos (memory BW)  │
  ┌──────▼──────┐         │  └── nxp_scmi_thermal_cooling    │
  │  Sensor     │         │      (M33 SM SCMI protocol)      │
  │  Drivers    │         └──────────────────────────────────┘
  │  SCMI Sensor│
  │  (via MU)   │
  └─────────────┘
```

#### Thermal Sensor trên i.MX95: SCMI Sensor Protocol

i.MX95 không có TMPSNS IP trực tiếp accessible từ A55 — thay vào đó, **M33 System Manager** sở hữu tất cả temperature sensors và expose qua SCMI Sensor protocol.

```c
/* drivers/thermal/scmi_thermal.c (Linux kernel) */
/* Kernel đọc nhiệt độ qua SCMI Sensor protocol 0x15 */

struct scmi_sensor_info {
    uint32_t id;
    uint32_t attr;
    char name[SCMI_MAX_STR_SIZE];   /* "CPU_TEMP", "SOC_TEMP", etc. */
    int64_t scale;                  /* milli-degrees */
};

/* SCMI message để đọc sensor: */
struct scmi_msg_sensor_reading_get {
    __le32 sensor_id;       /* ID của sensor, từ SCMI sensor_list */
    __le32 flags;           /* bit[0]: async, bit[1]: extended */
};

/* Response: */
struct scmi_resp_sensor_reading_get {
    __le32 sensor_value_low;    /* Lower 32 bits */
    __le32 sensor_value_high;   /* Upper 32 bits (sign extended) */
    /* Giá trị: millidegree Celsius */
    /* Ví dụ: 45000 = 45.0°C */
};

/* Thermal zone driver sử dụng SCMI: */
static int scmi_thermal_get_temp(struct thermal_zone_device *tz, int *temp)
{
    struct scmi_thermal_zone *stz = tz->devdata;
    u64 reading;
    
    /* Gửi SCMI message đến M33 SM qua MU1 (Messaging Unit) */
    ret = scmi_sensor_reading_get(stz->scmi_sensor, stz->id, &reading);
    
    *temp = (int)(reading / 1000);  /* micro → milli-degrees */
    return ret;
}
```

**M33 System Manager — Sensor Management:**

```c
/* imx-sm firmware: sm/dev/dev_sm_sensor.c */
/* M33 đọc trực tiếp từ Temperature Sensor hardware registers */

/* i.MX95 Temperature Sensor (ANATOP domain, controlled by M33): */
/* Base: 0x44480000 (ANATOP) */
/* TEMPSENSE0: 0x44480300 */

uint32_t SM_SENSOR_TempGet(uint32_t sensor_id)
{
    uint32_t raw;
    int32_t temp_mc;  /* milli-celsius */
    
    /* Enable temperature sensor */
    ANATOP->TEMPSENSE0 |= TEMPSENSE_CTRL_ENABLED;
    
    /* Wait for conversion (typical 100us) */
    while (!(ANATOP->TEMPSENSE0 & TEMPSENSE_STATUS_FINISHED));
    
    /* Read raw ADC value */
    raw = (ANATOP->TEMPSENSE0 & TEMPSENSE_STATUS_TEMP_CNT_MASK)
          >> TEMPSENSE_STATUS_TEMP_CNT_SHIFT;
    
    /* Convert to temperature using calibration from eFuse: */
    /* temp (°C) = HOT_TEMP - (raw - HOT_COUNT) * (HOT_TEMP - 25) 
                              / (ROOM_COUNT - HOT_COUNT)          */
    /* Calibration values burned in OTP bank 3 during NXP factory test */
    int32_t hot_temp  = otp_hot_temp;    /* typically 105°C */
    int32_t hot_count = otp_hot_count;   /* raw ADC at hot_temp */
    int32_t room_count = otp_room_count; /* raw ADC at 25°C */
    
    temp_mc = (hot_temp * 1000) -
              (int32_t)(raw - hot_count) * (hot_temp - 25) * 1000
              / (room_count - hot_count);
    
    return (uint32_t)temp_mc;
}
```

#### Device Tree: Thermal Zones & Trip Points

```dts
/* arch/arm64/boot/dts/freescale/imx95-thermal.dtsi */

/* Thermal zones sử dụng SCMI sensor */
thermal_zones: thermal-zones {

    /*─── Zone 0: CPU Cluster Temperature ───*/
    cpu_thermal: cpu-thermal {
        polling-delay-passive = <250>;   /* ms: passive cooling interval */
        polling-delay = <2000>;          /* ms: idle monitoring interval */
        
        /* SCMI sensor #0 = CPU temp (từ M33 sensor list) */
        thermal-sensors = <&scmi_sensor 0>;
        
        trips {
            /* Trip point 1: Passive cooling bắt đầu */
            cpu_alert0: trip0 {
                temperature = <85000>;   /* 85°C */
                hysteresis  = <2000>;    /* ±2°C hysteresis */
                type = "passive";
                /* → cpufreq_cooling: giảm max freq */
            };
            
            /* Trip point 2: Aggressive throttling */
            cpu_alert1: trip1 {
                temperature = <95000>;   /* 95°C */
                hysteresis  = <2000>;
                type = "passive";
                /* → cpufreq_cooling: min freq */
                /* → devfreq: giảm DDR bandwidth */
            };
            
            /* Trip point 3: Critical — shutdown */
            cpu_crit: trip2 {
                temperature = <105000>;  /* 105°C = TJ max của i.MX95 */
                hysteresis  = <2000>;
                type = "critical";
                /* → thermal_core triggers emergency_poweroff() */
                /* → SCMI: SM_PWR_DOMAIN_OFF cho CPU domain */
            };
        };
        
        /* Cooling devices map */
        cooling-maps {
            /* Trip0 → CPU frequency scaling */
            map0 {
                trip = <&cpu_alert0>;
                cooling-device = <&cpu0           /* Cortex-A55 CPU0 */
                                   THERMAL_NO_LIMIT
                                   THERMAL_NO_LIMIT>;
                /* cpufreq_cooling: state 0=1800MHz, state N=300MHz */
            };
            
            /* Trip1 → Also throttle GPU */
            map1 {
                trip = <&cpu_alert1>;
                cooling-device = <&gpu 0 4>;  /* GPU freq states 0-4 */
            };
        };
    };

    /*─── Zone 1: SoC/Die Temperature ───*/
    soc_thermal: soc-thermal {
        polling-delay-passive = <500>;
        polling-delay = <5000>;
        thermal-sensors = <&scmi_sensor 1>;  /* SCMI sensor #1 = SoC */
        
        trips {
            soc_alert: trip0 {
                temperature = <90000>;    /* 90°C */
                hysteresis  = <5000>;
                type = "passive";
            };
            soc_crit: trip1 {
                temperature = <110000>;   /* 110°C */
                hysteresis  = <5000>;
                type = "critical";
            };
        };
        cooling-maps {
            map0 {
                trip = <&soc_alert>;
                /* Throttle via SCMI performance domain */
                cooling-device = <&scmi_dvfs 0 THERMAL_NO_LIMIT THERMAL_NO_LIMIT>;
            };
        };
    };

    /*─── Zone 2: NPU (Neural Processing Unit) Temperature ───*/
    npu_thermal: npu-thermal {
        polling-delay-passive = <500>;
        polling-delay = <5000>;
        thermal-sensors = <&scmi_sensor 4>;  /* SCMI sensor #4 = NPU */
        trips {
            npu_alert: trip0 {
                temperature = <80000>;
                hysteresis = <2000>;
                type = "passive";
            };
        };
        cooling-maps {
            map0 {
                trip = <&npu_alert>;
                cooling-device = <&npu 0 4>;  /* NPU devfreq states */
            };
        };
    };
};
```

**Thermal Throttling Sequence (step_wise governor):**

```
Nhiệt độ tăng dần:
  T < 85°C  → Không throttle, CPU @ 1800MHz max
  T ≥ 85°C  → trip0 "passive" triggered:
              step_wise governor: giảm 1 cooling state mỗi polling interval
              cpufreq_cooling state 0→1→2→3... (1800→1400→1200→800MHz)
  T ≥ 95°C  → trip1 "passive" triggered:
              Thêm cooling device (DDR DVFS, GPU)
              CPU forced to minimum: 300MHz (emergency throttle)
  T ≥ 105°C → trip2 "critical":
              thermal_emergency_poweroff(0) → system shutdown trong 10s
              hoặc SCMI system reset nếu PM không available

sysfs monitoring:
  /sys/class/thermal/thermal_zone0/temp         # millidegree
  /sys/class/thermal/thermal_zone0/trip_point_0_temp
  /sys/class/thermal/thermal_zone0/policy       # step_wise / power_allocator
  /sys/class/thermal/cooling_device0/cur_state  # current cooling state
  /sys/class/thermal/cooling_device0/max_state  # max cooling state
```

**dmesg log throttling:**

```
[   45.231000] thermal thermal_zone0: critical temperature reached (105C), shutting down
[   45.232000] thermal thermal_zone0: Initiating system poweroff
[  123.456000] imx_scmi_perf scmi_perf.0: SCMI performance domain 0: set level 300000 (was 1800000)
[  123.457000] cpu cpu0: cpufreq: set_cur_freq: 300000 kHz
```

### 6.2 SCMI Performance Domain: Thermal → CPUFreq Path

```c
/* drivers/cpufreq/scmi-cpufreq.c */
/* CPUFreq driver trên i.MX95 dùng SCMI Performance Domain (0x13) */

static int scmi_cpufreq_set_target(struct cpufreq_policy *policy,
                                    unsigned int index)
{
    struct scmi_data *priv = policy->driver_data;
    struct scmi_perf_domain *domain = priv->domain;
    
    /* Gửi SCMI message PERF_LEVEL_SET đến M33 SM */
    /* M33 SM nhận → điều chỉnh PLL/CCM → thay đổi CPU frequency */
    return scmi_perf_level_set(domain, freq_table[index].frequency,
                               false /* synchronous */);
}

/* SCMI Performance message format: */
/* Protocol 0x13, Message ID 0x07 (PERF_LEVEL_SET) */
struct scmi_msg_perf_set_level {
    __le32 domain_id;    /* 0 = A55 cluster */
    __le32 performance_level;  /* frequency in kHz hoặc OPP index */
};

/* M33 SM xử lý PERF_LEVEL_SET: */
/* imx-sm/sm/dev/dev_sm_perf.c */
int32_t SM_PERF_LevelSet(uint32_t domain_id, uint32_t level)
{
    /* Translate OPP level → PLL configuration */
    /* CCM (Clock Controller Module) PLL1 → A55_CLK_ROOT */
    /* CCM base: 0x44450000 */
    /* A55_CLK_ROOT: CCM_ROOT_CTRL(32) */
    
    /* Example: 1800MHz → ARM PLL1 = 3600MHz / 2 */
    CCM_ANALOG->ARM_PLL1_CTRL =
        PLL_CTRL_ENABLE | PLL_CTRL_CLKE |
        PLL_CTRL_DIV_SELECT(90);  /* Fvco = 24MHz * 90 = 2160MHz */
    
    /* Wait PLL lock */
    while (!(CCM_ANALOG->ARM_PLL1_CTRL & PLL_CTRL_LOCK));
    
    /* Switch A55_CLK_ROOT source */
    CCM->CLOCK_ROOT[A55_CLK_ROOT].CONTROL =
        CCM_CLOCK_ROOT_MUX(SYS_PLL1_800M) | CCM_CLOCK_ROOT_DIV(1);
    
    return SM_ERR_SUCCESS;
}
```

### 6.3 EAS (Energy Aware Scheduling) trên Cortex-A55

**Điều kiện tiên quyết để EAS hoạt động trên i.MX95:**

```
EAS Requirements:
  1. schedutil CPUFreq governor (KHÔNG dùng ondemand/performance)
  2. Energy Model (EM) registered cho mỗi CPU
  3. CFS tasks (EAS chỉ hoạt động với CFS, không phải RT/DL)
  4. CONFIG_ENERGY_MODEL=y, CONFIG_CPU_FREQ_GOV_SCHEDUTIL=y
  5. Single performance domain (tất cả A55 cùng cluster = ok)
  6. Thermal pressure tracking

i.MX95: 6x Cortex-A55 @ 1.8GHz (1 cluster, homogeneous)
  → Single performance domain → EAS fully applicable
  → Nếu sau này có big.LITTLE: EAS sẽ manage inter-cluster migration
```

**Energy Model Registration:**

```c
/* drivers/cpufreq/scmi-cpufreq.c hoặc arm_scmi_energy_model.c */

/* Energy Model per CPU phải được đăng ký khi driver probe */
static int imx95_register_energy_model(struct device *cpu_dev,
                                        struct cpufreq_policy *policy)
{
    /* OPP table (từ Device Tree cpu0 operating-points-v2): */
    /* freq(kHz)    power(μW)    */
    /* 300000       10000        */
    /* 600000       25000        */
    /* 800000        40000        */
    /* 1000000       65000        */
    /* 1200000       95000        */
    /* 1400000      135000        */
    /* 1600000      185000        */
    /* 1800000      245000        */
    
    /* Power values đo bằng cách: */
    /* P_dyn = C * V^2 * f  (dynamic) */
    /* P_sta = V * I_leakage (static, phụ thuộc nhiệt độ) */
    /* Giá trị calibrated từ silicon characterization */
    
    ret = dev_pm_opp_of_register_em(cpu_dev, policy->cpus);
    /* → em_pd_get(cpu_dev) → returns Energy Model */
    return ret;
}
```

**Device Tree: OPP + Energy Model:**

```dts
/* arch/arm64/boot/dts/freescale/imx95.dtsi */

cpus {
    #address-cells = <1>;
    #size-cells    = <0>;

    cpu-map {
        cluster0 {
            core0 { cpu = <&cpu0>; };
            core1 { cpu = <&cpu1>; };
            core2 { cpu = <&cpu2>; };
            core3 { cpu = <&cpu3>; };
            core4 { cpu = <&cpu4>; };
            core5 { cpu = <&cpu5>; };
        };
    };

    cpu0: cpu@0 {
        compatible = "arm,cortex-a55";
        device_type = "cpu";
        reg = <0x000>;
        enable-method = "psci";
        
        /* SCMI-based DVFS */
        clocks = <&scmi_clk IMX95_CLK_A55>;
        operating-points-v2 = <&a55_opp_table>;
        
        /* CPU idle states (PSCI CPU_SUSPEND) */
        cpu-idle-states = <&CPU_SLEEP &CLUSTER_SLEEP>;
        
        /* Capacity-aware scheduling (schedutil EAS) */
        capacity-dmips-mhz = <530>;  /* Cortex-A55: ~530 DMIPS/MHz */
    };
    
    /* cpu1-cpu5: identical, same cluster */
    cpu1: cpu@100 { compatible = "arm,cortex-a55"; reg = <0x100>; ... };
    cpu2: cpu@200 { compatible = "arm,cortex-a55"; reg = <0x200>; ... };
    cpu3: cpu@300 { compatible = "arm,cortex-a55"; reg = <0x300>; ... };
    cpu4: cpu@400 { compatible = "arm,cortex-a55"; reg = <0x400>; ... };
    cpu5: cpu@500 { compatible = "arm,cortex-a55"; reg = <0x500>; ... };
};

/* OPP Table với Energy Model data */
a55_opp_table: opp-table-0 {
    compatible = "operating-points-v2";
    opp-shared;  /* ← Tất cả cpu0-cpu5 share cùng table */
    
    opp-300000000 {
        opp-hz = /bits/ 64 <300000000>;
        opp-microvolt = <700000>;       /* 0.7V */
        opp-microwatt = <10000>;        /* 10mW total cluster */
        clock-latency-ns = <150000>;
    };
    opp-800000000 {
        opp-hz = /bits/ 64 <800000000>;
        opp-microvolt = <800000>;       /* 0.8V */
        opp-microwatt = <40000>;        /* 40mW */
        clock-latency-ns = <150000>;
    };
    opp-1200000000 {
        opp-hz = /bits/ 64 <1200000000>;
        opp-microvolt = <900000>;       /* 0.9V */
        opp-microwatt = <95000>;        /* 95mW */
        clock-latency-ns = <150000>;
    };
    opp-1800000000 {
        opp-hz = /bits/ 64 <1800000000>;
        opp-microvolt = <1000000>;      /* 1.0V */
        opp-microwatt = <245000>;       /* 245mW */
        clock-latency-ns = <150000>;
        opp-suspend;                    /* Wake-up OPP */
    };
};

/* CPU Idle States */
idle-states {
    entry-method = "psci";
    
    CPU_SLEEP: cpu-sleep {
        compatible = "arm,idle-state";
        arm,psci-suspend-param = <0x0010000>;  /* State 1: WFI + retention */
        local-timer-stop;
        entry-latency-us  = <100>;
        exit-latency-us   = <250>;
        min-residency-us  = <1000>;
        wakeup-latency-us = <350>;
    };
    
    CLUSTER_SLEEP: cluster-sleep {
        compatible = "arm,idle-state";
        arm,psci-suspend-param = <0x1010000>;  /* State 1: Full cluster off */
        local-timer-stop;
        entry-latency-us  = <300>;
        exit-latency-us   = <1200>;
        min-residency-us  = <5000>;
        wakeup-latency-us = <1500>;
    };
};
```

**EAS Decision Flow trong Scheduler:**

```c
/* kernel/sched/fair.c */
/* EAS task placement: tìm CPU nào có energy cost thấp nhất */

static int find_energy_efficient_cpu(struct task_struct *p,
                                      int prev_cpu, int sync)
{
    struct perf_domain *pd;
    int best_cpu = prev_cpu;
    unsigned long best_energy = ULONG_MAX;
    
    rcu_read_lock();
    /* Iterate over performance domains (1 domain = A55 cluster) */
    for_each_perf_domain(pd, task_rq(p)) {
        
        /* Tính energy nếu place task p trên mỗi CPU trong domain */
        for_each_cpu(cpu, pd->cpus) {
            
            /* compute_energy(): */
            /* 1. Estimate new utilization nếu p chạy trên cpu */
            /* 2. Map utilization → OPP frequency via schedutil */
            /* 3. Lookup energy từ Energy Model (em_cpu_energy) */
            /* 4. Sum active CPUs energy + idle CPUs static power */
            
            energy = compute_energy(p, cpu, pd);
            
            if (energy < best_energy) {
                best_energy = energy;
                best_cpu = cpu;
            }
        }
    }
    rcu_read_unlock();
    return best_cpu;
}

/* Verify EAS đang hoạt động: */
/* dmesg | grep -i "EAS\|energy\|sched" */
```

**sysfs: Verify EAS & CPUFreq:**

```bash
# Kiểm tra CPUFreq governor
adb shell cat /sys/devices/system/cpu/cpu0/cpufreq/scaling_governor
# schedutil  ← EAS cần schedutil

# Kiểm tra OPP table
adb shell cat /sys/devices/system/cpu/cpu0/cpufreq/scaling_available_frequencies
# 300000 600000 800000 1000000 1200000 1400000 1600000 1800000

# Kiểm tra current frequency
adb shell cat /sys/devices/system/cpu/cpu0/cpufreq/scaling_cur_freq
# 1200000  (= 1.2GHz khi load vừa phải)

# Kiểm tra Energy Model
adb shell ls /sys/devices/system/cpu/cpu0/cpuidle/
# state0/  state1/  ← CPU_SLEEP, CLUSTER_SLEEP

# Thermal pressure
adb shell cat /sys/kernel/debug/sched/sched_thermal_pressure 2>/dev/null

# Power stats
adb shell dumpsys batterystats | grep -A5 "CPU"

# Idle stats (verify CPUIdle hoạt động)
adb shell cat /sys/devices/system/cpu/cpu0/cpuidle/state1/usage
adb shell cat /sys/devices/system/cpu/cpu0/cpuidle/state1/time  # microseconds
```

---

