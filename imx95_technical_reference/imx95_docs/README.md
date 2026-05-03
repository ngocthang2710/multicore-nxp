# NXP i.MX95 EVK — Technical Reference Library

> **Platform:** NXP i.MX95 EVK · ARMv9-A (Cortex-A55 × 6) + Cortex-M33 + EdgeLock Enclave  
> **Android:** AOSP Android 15 (API 35) / Android Automotive OS  
> **Kernel:** Linux 6.6.x GKI  
> **Author:** Senior Embedded Linux / AAOS Engineer  

---

## Cấu Trúc Tài Liệu

| File | Nội Dung | Phần | Dung Lượng |
|------|----------|------|-----------|
| `01_boot_sequence.md` | Toàn cảnh chuỗi khởi động + Source Code Map | §1–2 | ~53 KB |
| `02_build_and_artifacts.md` | Modular Build + Output Artifacts & Flashing | §3–4 | ~30 KB |
| `03_ab_ota_thermal_power.md` | A/B OTA Seamless Update + Power & Thermal | §5–6 | ~38 KB |
| `04_iomuxc_pinmuxing.md` | IOMUXC Pin Muxing & PAD Configuration | §7 | ~21 KB |
| `05_debug_tracing_binder.md` | Debug End-to-End, ftrace, Perfetto, Binder IPC | §8–9 | ~22 KB |
| `06_memory_security.md` | Memory Management (DMA-BUF, LMKD) + Security (AVB, dm-verity, SELinux) | §10–11 | ~20 KB |
| `07_performance_multimedia.md` | Performance Optimization + Multimedia Pipeline | §12–13 | ~13 KB |
| `08_networking_system_design.md` | Networking, CAN, SOME/IP + System Design Trade-off | §14–15 | ~16 KB |

---

## Quick Navigation

### Boot Chain
```
BootROM → ELE Auth → SPL → DDR PHY Train → ATF BL31 → M33 SM → U-Boot → Kernel → Android Init → Zygote
```
→ Chi tiết: `01_boot_sequence.md`

### Build một thành phần nhanh
```bash
m bootloader        # U-Boot + ATF + flash.bin
m bootimage         # GKI boot.img
m vendorimage       # vendor.img (HAL, modules, configs)
m android.hardware.audio.service.nxp  # chỉ Audio HAL
```
→ Chi tiết: `02_build_and_artifacts.md`

### Debug audio silent
```bash
adb shell getprop ro.android.car.audio.enableaudiopatch  # phải = true
adb shell logcat | grep fmqByteCount                     # phải > 0
adb shell tinyplay test.wav -D 2 -d 0                    # isolate kernel layer
```
→ Chi tiết: `05_debug_tracing_binder.md` + `08_networking_system_design.md §15.4`

### A/B Rollback verify
```bash
adb shell bootctl get-current-slot
adb shell dd if=/dev/block/by-name/misc bs=1 skip=2048 count=32 | xxd
```
→ Chi tiết: `03_ab_ota_thermal_power.md`

### IOMUXC PAD value decode
```
0x40000b9e → SION=1, DSE=65Ω, SLEW=SLOW, HYS=1, ODE=1(I2C!), PUE+PUS
```
→ Chi tiết: `04_iomuxc_pinmuxing.md`

---

## Môi Trường Tham Chiếu

| Component | Version / Details |
|-----------|-------------------|
| SoC | NXP i.MX9592 (i.MX95), TSMC N4P |
| CPU | 6× Cortex-A55 @ 1.8 GHz |
| MCU | Cortex-M33 (System Manager) + ELE (EdgeLock Enclave) |
| RAM | LPDDR5-6400, 8 GB |
| Storage | eMMC 5.1 HS400, 64 GB |
| Android | AOSP Android 15 (AP3A) |
| Kernel | android15-6.6.23 GKI |
| U-Boot | 2024.04-lf-6.6.23-2.0.0 |
| ATF | v2.10.0 |
| Toolchain | LLVM/Clang 17 (Android), GCC 12.x (M33) |

