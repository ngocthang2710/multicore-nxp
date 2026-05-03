# NXP i.MX95 EVK — Build Ecosystem & Output Artifacts
> Phần 3–4 của tài liệu NXP i.MX95 Deep-Dive

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

