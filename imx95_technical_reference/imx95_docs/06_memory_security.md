# NXP i.MX95 EVK — Memory Management & Security
> Phần 10–11 của tài liệu NXP i.MX95 Deep-Dive

---

## 10. Memory Management (Android + Linux)

### 10.1 Memory Architecture Overview

```
Android Memory Layers:

┌─────────────────────────────────────────────────────────────────┐
│  Application (Java/Kotlin)                                       │
│  ├── Java Heap: ART GC managed (dalvik.vm.heapsize)            │
│  └── Native Heap: malloc/free via jemalloc/scudo               │
├─────────────────────────────────────────────────────────────────┤
│  Android Shared Memory                                           │
│  ├── ashmem/memfd: shared between processes (Binder FD passing) │
│  └── ASharedMemory (NDK): /dev/ashmem or memfd_create()        │
├─────────────────────────────────────────────────────────────────┤
│  Graphics / Media Memory                                         │
│  ├── DMA-BUF: kernel-managed, shareable across subsystems       │
│  ├── ION (deprecated): allocator for GPU/Camera/DSP buffers     │
│  └── gralloc HAL: Android graphics buffer allocation            │
├─────────────────────────────────────────────────────────────────┤
│  Linux Kernel Memory                                             │
│  ├── Page Cache: file-backed (VFS, ext4, erofs)                 │
│  ├── Anonymous memory: stack, heap, mmap private                │
│  ├── Slab/SLUB: kernel object allocator                         │
│  └── CMA: Contiguous Memory Allocator (for DMA)                 │
├─────────────────────────────────────────────────────────────────┤
│  Physical Memory (LPDDR5, 8GB on i.MX95 EVK)                   │
│  ├── Kernel reserved (BL31, DTB, CMA): ~512MB                  │
│  └── Available to userspace/page cache: ~7.5GB                  │
└─────────────────────────────────────────────────────────────────┘
```

### 10.2 DMA-BUF: Multimedia Buffer Sharing

DMA-BUF là cơ chế chia sẻ buffer giữa các subsystems (Camera, GPU, Display, Video decoder) mà **không** cần memcpy.

```c
/* drivers/dma-buf/dma-buf.c */

/* Exporter: Camera driver tạo buffer */
struct dma_buf *dma_buf_export(const struct dma_buf_export_info *exp_info)
{
    struct dma_buf *dmabuf;
    
    /* Tạo anonymous file (fd) đại diện cho buffer */
    dmabuf->file = anon_inode_getfile("dmabuf", &dma_buf_fops, dmabuf, O_RDWR);
    
    /* exp_info->ops: exporter-specific operations */
    /* Camera exporter ops:
       .map_dma_buf  = camera_dma_map  (tạo sg_table từ physical pages)
       .unmap_dma_buf = camera_dma_unmap
       .mmap         = camera_dma_mmap  (map vào userspace)
    */
    return dmabuf;
}

/* Importer: GPU driver nhận buffer để render */
struct dma_buf_attachment *dma_buf_attach(struct dma_buf *dmabuf,
                                           struct device *dev)
{
    /* Attach GPU device vào buffer */
    /* Allows GPU to map the buffer for DMA */
    return attach;
}

struct sg_table *dma_buf_map_attachment(struct dma_buf_attachment *attach,
                                         enum dma_data_direction direction)
{
    /* Get scatter-gather list of physical pages */
    /* GPU MMU được cấu hình để access các pages này */
    return attach->dmabuf->ops->map_dma_buf(attach, direction);
}

/* ─── Userspace flow (Camera → GPU → Display): ─── */
/* 1. Camera driver: dma_buf_export() → fd1 */
/* 2. Camera HAL passes fd1 over AIDL to GPU */
/* 3. GPU driver: dma_buf_attach(fd1) → renders into buffer */
/* 4. GPU passes fd1 to SurfaceFlinger */
/* 5. Display driver: dma_buf_attach(fd1) → DMA to display controller */
/* ZERO memcpy throughout the pipeline */

/* ─── Userspace DMA-BUF API: ─── */
/* AHardwareBuffer_allocate() → gralloc HAL → ION/DMA-BUF */
/* AHardwareBuffer_sendHandleToUnixSocket() → share across processes */
```

**ION Allocator (legacy, Android < 12) vs DMA-BUF Heaps (current):**

```c
/* DMA-BUF Heaps (drivers/dma-buf/heaps/): */
/* /dev/dma_heap/system    → physically non-contiguous (vmalloc) */
/* /dev/dma_heap/cma       → physically contiguous (CMA) for DMA */
/* /dev/dma_heap/system-uncached → non-cached, for DMA coherency */

/* Allocation from userspace: */
int heap_fd = open("/dev/dma_heap/system", O_RDONLY);
struct dma_heap_allocation_data alloc = {
    .len = buffer_size,
    .fd_flags = O_RDWR | O_CLOEXEC,
};
ioctl(heap_fd, DMA_HEAP_IOCTL_ALLOC, &alloc);
int buf_fd = alloc.fd;  /* DMA-BUF file descriptor */

/* Share với process khác: */
/* Pass buf_fd qua AIDL (ParcelFileDescriptor) hoặc SCM_RIGHTS socket */
```

### 10.3 LMKD (Low Memory Killer Daemon)

LMKD quyết định process nào bị kill khi hệ thống hết memory. Android 11+ dùng userspace LMKD thay kernel LMK.

```c
/* system/memory/lmkd/lmkd.cpp */

/* LMKD monitors memory pressure qua PSI (Pressure Stall Information): */
/* /proc/pressure/memory — Linux 4.20+ */
/* "some avg10=X.XX avg60=Y.YY avg300=Z.ZZ total=NNN" */
/* avg10 > threshold → trigger kill */

/* Memory pressure levels: */
/* LOW: some avg60 > 10%  → kill cached processes */
/* MEDIUM: some avg10 > 30% → kill service processes */  
/* CRITICAL: full avg10 > 25% → kill foreground processes (last resort) */

struct lmk_proc_prio_t {
    pid_t pid;
    int oom_score_adj;  /* -1000 to 1000, higher = more killable */
    /* OOM scores: */
    /* Native daemons:        -1000 (never kill) */
    /* SystemServer:          -900 */
    /* Persistent services:   -800 */
    /* Foreground app:        0    */
    /* Visible app:           100  */
    /* Perceptible:           200  */
    /* Service:               500  */
    /* Cached/background:     900  */
};

static void kill_one_process(struct proc *procp, int min_score_adj,
                               enum kill_reasons kill_reason)
{
    /* Find highest oom_score_adj process above min_score_adj */
    /* Send SIGKILL */
    kill(procp->pid, SIGKILL);
    
    /* Log kill event */
    ALOGI("Kill '%s' (%d), uid %d, oom_score_adj %d to free %" PRId64 "kB",
          procp->cmdline, procp->pid, procp->uid,
          procp->oom_score_adj, tasksize * PAGE_SIZE / 1024);
}

/* ─── Debug LMKD ─── */
adb shell logcat | grep "lmkd\|lowmemorykill\|Kill"
# lmkd: Kill 'com.example.app' (3847), uid 10045, oom_score_adj 900 to free 51200kB
```

**ZRAM (Swap Compression):**

```bash
# ZRAM là block device với LZ4/LZO compression trong RAM
# Effective ratio: ~2-3x (100MB app compressed thành ~35MB ZRAM)

# Verify ZRAM trên Android:
adb shell cat /proc/swaps
# Filename    Type    Size    Used    Priority
# /dev/block/zram0  partition  4194300  123456  100

adb shell cat /proc/sys/vm/swappiness
# 100 (aggressive swap on Android, để free RAM cho active processes)

adb shell zramctl
# NAME         ALGORITHM DISKSIZE   DATA   COMPR  TOTAL STREAMS MOUNTPOINT
# /dev/zram0   lz4          4G   456.2M  156.3M  180M       6 [SWAP]

# Compression ratio:
adb shell cat /sys/block/zram0/stat
# mm_compr_data_size / mm_orig_data_size = compression ratio

# OOM scenario debug:
adb shell cat /proc/meminfo
# MemTotal:       8145392 kB
# MemFree:          23456 kB  ← CRITICAL
# MemAvailable:    145678 kB
# Cached:          456789 kB
# SwapTotal:      4194300 kB
# SwapFree:        123456 kB  ← ZRAM mostly full → system under pressure
```

### 10.4 Memory Leak Detection

```bash
# ─── Native memory leak (C/C++) ─── 

# 1. malloc_debug (Android built-in)
adb shell setprop libc.debug.malloc.options backtrace=8
adb shell setprop libc.debug.malloc.program android.hardware.audio.service
adb shell stop audioserver && adb shell start audioserver

# 2. Address Sanitizer (build-time)
# In Android.bp:
# sanitize: { address: true }
# → Detects: heap overflow, use-after-free, double-free

# 3. procmem (Android tool)
adb shell procmem $(pidof audioserver)
# Shows: VSS, RSS, PSS, USS per mapping
# USS = memory unique to process (leaked memory shows here)

# 4. dumpsys meminfo
adb shell dumpsys meminfo audioserver
# App Summary:
#   Java Heap:    0K
#   Native Heap:  45,234K   ← watch this over time
#   Code:         12,345K
#   Stack:         1,234K
#   Graphics:          0K
#   Total PSS:    67,890K

# 5. Valgrind (via emulator atau custom build)
# adb shell valgrind --leak-check=full /vendor/bin/hw/audioserver

# ─── Java heap leak ─── 
# adb shell am dumpheap com.example.app /sdcard/heap.hprof
# adb pull /sdcard/heap.hprof
# Analyze in Android Studio → Memory Profiler

# ─── Kernel memory leak ─── 
adb shell cat /proc/slabinfo | sort -k3 -rn | head -20
# Tìm slab entry nào có active_objs tăng liên tục

# kmemleak detector:
adb shell cat /sys/kernel/debug/kmemleak  # Requires CONFIG_DEBUG_KMEMLEAK=y
```

---

## 11. Security (Advanced)

### 11.1 AVB 2.0 (Android Verified Boot)

```
AVB 2.0 Chain of Trust:

  BootROM (immutable, on-chip)
      │ SHA384 verify
      ▼
  ELE Firmware + SPL  (signed with OEM key in Boot Container)
      │ RSA-4096 / ECDSA-P384 verify
      ▼
  ATF + U-Boot        (signed, in Boot Container)
      │
      ▼
  U-Boot: avb_verify_partition()
      │ vbmeta_a: RSA-4096 signature
      │ Checks: boot_a SHA256, system_a SHA256, vendor_a SHA256
      ▼
  Kernel + initramfs  (verified by vbmeta chain)
      │
      ▼
  dm-verity (kernel)  (system/vendor mounted as verified block devices)
      │ SHA256 hash tree over partition
      ▼
  Android userspace   (SELinux enforcing)

Verified Boot States:
  GREEN  : OEM-locked, all signatures verify    (production)
  YELLOW : OEM-locked, user-supplied key verify (custom key)
  ORANGE : OEM-unlocked (fastboot oem unlock)   (developer)
  RED    : Verification failure → halt or warn  (corrupted)

androidboot.verifiedbootstate=[green|yellow|orange|red]
```

**vbmeta Structure:**

```c
/* external/avb/libavb/avb_vbmeta_image.h */
typedef struct AvbVBMetaImageHeader {
    uint8_t  magic[4];              /* "AVB0" */
    uint32_t required_libavb_version_major;  /* 1 */
    uint32_t required_libavb_version_minor;
    uint64_t authentication_data_block_size;
    uint64_t auxiliary_data_block_size;
    uint32_t algorithm_type;        /* AVB_ALGORITHM_TYPE_SHA256_RSA4096 */
    uint64_t hash_offset;           /* Offset of hash within auth block */
    uint64_t hash_size;             /* 32 bytes (SHA256) */
    uint64_t signature_offset;      /* Offset of RSA/ECDSA signature */
    uint64_t signature_size;        /* 512 bytes (RSA-4096) */
    uint64_t public_key_offset;     /* Public key in auxiliary block */
    uint64_t public_key_size;
    uint64_t public_key_metadata_offset;
    uint64_t public_key_metadata_size;
    uint64_t descriptors_offset;    /* → AvbHashDescriptor, AvbHashtreeDescriptor */
    uint64_t descriptors_size;
    uint64_t rollback_index;        /* Anti-rollback: >= stored in ELE OTP */
    uint32_t flags;
    uint8_t  release_string[48];    /* "avbtool 1.2.0" */
    uint8_t  reserved[80];
} AVB_ATTR_PACKED AvbVBMetaImageHeader;

/* AvbHashtreeDescriptor: describes dm-verity hash tree */
typedef struct AvbHashtreeDescriptor {
    AvbDescriptor parent_descriptor;  /* tag=AVB_DESCRIPTOR_TAG_HASHTREE */
    uint32_t dm_verity_version;       /* 1 */
    uint64_t image_size;              /* Partition data size */
    uint64_t tree_offset;             /* Hash tree starts at this offset */
    uint64_t tree_size;
    uint32_t data_block_size;         /* 4096 bytes */
    uint32_t hash_block_size;         /* 4096 bytes */
    uint32_t fec_num_roots;           /* Forward Error Correction (0 if disabled) */
    uint64_t fec_offset;
    uint64_t fec_size;
    uint8_t  hash_algorithm[32];      /* "sha256" */
    uint32_t partition_name_len;      /* "system" */
    uint32_t salt_len;                /* 32 bytes random salt */
    uint32_t root_digest_len;         /* 32 bytes (SHA256 of hash tree root) */
    uint8_t  reserved[60];
} AVB_ATTR_PACKED AvbHashtreeDescriptor;
```

**U-Boot AVB verification:**

```c
/* boot/android/avb2.c (U-Boot) */
AvbSlotVerifyResult avb_slot_verify(...)
{
    /* 1. Read vbmeta_a from eMMC */
    avb_io_manager->read_from_partition(io_manager, "vbmeta_a",
                                         0, AVB_VBMETA_IMAGE_HEADER_SIZE,
                                         vbmeta_buf, &num_read);
    
    /* 2. Verify vbmeta signature */
    avb_vbmeta_image_verify(vbmeta_buf, vbmeta_size, &pk_data, &pk_len);
    /* → SHA256(header + auxiliary) == signature_RSA4096_decrypt(sig, pk) */
    
    /* 3. Verify rollback index */
    avb_ops->read_rollback_index(avb_ops, rollback_index_location,
                                  &stored_rollback);
    if (vbmeta->rollback_index < stored_rollback) {
        return AVB_SLOT_VERIFY_RESULT_ERROR_ROLLBACK_INDEX;
        /* Anti-rollback: stored in ELE OTP, eFuse bank */
    }
    
    /* 4. Verify each partition hash (boot, system, vendor) */
    /* Read hash descriptor → verify SHA256 of partition data */
    
    /* 5. Setup dm-verity cmdline */
    /* Appends to bootargs: */
    /* "dm=1 vroot none ro 1,0 N verity 1 /dev/mmcblk0p5 /dev/mmcblk0p5 
          4096 4096 N N sha256 <roothash> <salt>" */
    cmdline_append(slot_data->cmdline);
    
    return AVB_SLOT_VERIFY_RESULT_OK;
}
```

### 11.2 dm-verity (Kernel Side)

```c
/* drivers/md/dm-verity-target.c */

/* dm-verity creates a device mapper target over system/vendor partition */
/* Read: verifies each 4KB block against SHA256 hash tree on-the-fly */

static int verity_map(struct dm_target *ti, struct bio *bio)
{
    struct dm_verity *v = ti->private;
    
    /* For each read bio: */
    /* 1. Get block number from bio->bi_iter.bi_sector */
    /* 2. Look up leaf hash in hash tree */
    /* 3. Verify parent hashes up to root */
    /* 4. Compare root with stored root_digest (from vbmeta) */
    
    if (hash_mismatch) {
        if (v->mode == DM_VERITY_MODE_EIO) {
            bio->bi_status = BLK_STS_IOERR;
            bio_endio(bio);
        } else if (v->mode == DM_VERITY_MODE_RESTART) {
            /* Trigger kernel panic → reboot → A/B rollback */
            kernel_restart("dm-verity device corrupted");
        } else if (v->mode == DM_VERITY_MODE_LOGGING) {
            /* Log only, allow read (development mode) */
            DMERR_LIMIT("dm-verity: %s: data block %llu is corrupted",
                        v->data_dev->name, (unsigned long long)cur_block);
        }
    }
    
    /* Forward bio to actual block device */
    bio_set_dev(bio, v->data_dev->bdev);
    submit_bio(bio);
    return DM_MAPIO_SUBMITTED;
}

/* ─── Debug dm-verity ─── */
adb shell dmctl table system_a
# 0-3145727: verity 1 /dev/block/mmcblk0p12 /dev/block/mmcblk0p12
#            4096 4096 3145728 3145728 sha256
#            <roothash_hex> <salt_hex>
#            1 ignore_zero_blocks

# Disable verity (development only):
adb disable-verity   # → modifies fstab to use 'noverity' flag
adb reboot
```

### 11.3 SELinux Policy Tuning

```
SELinux Architecture trên Android:

  Process context: "u:r:hal_audio_default:s0"
  File context:    "u:object_r:vendor_file:s0"
  
  Access decision: allow/deny based on:
    type_enforcement (TE) rules
    
  Policy files:
    /system/etc/selinux/plat_sepolicy.cil    ← AOSP platform policy
    /vendor/etc/selinux/vendor_sepolicy.cil  ← OEM/vendor policy
    /vendor/etc/selinux/precompiled_sepolicy ← Compiled (shipped on device)
```

**Workflow thêm SELinux policy cho HAL mới:**

```bash
# 1. Identify AVC denial:
adb shell dmesg | grep "avc: denied"
# avc: denied { read } for pid=1234 comm="audioserver"
#   path="/vendor/etc/audio/tas5828_config.json"
#   scontext=u:r:audioserver:s0
#   tcontext=u:object_r:vendor_configs_file:s0
#   tclass=file permissive=0

# 2. Generate policy from audit log:
audit2allow -i avc_denial.txt
# #============= audioserver ==============
# allow audioserver vendor_configs_file:file read;

# 3. Add to vendor policy:
# device/nxp/imx95_evk/sepolicy/audioserver.te
echo 'allow audioserver vendor_configs_file:file { read open getattr };' \
    >> device/nxp/imx95_evk/sepolicy/audioserver.te

# 4. Label file correctly:
# device/nxp/imx95_evk/sepolicy/file_contexts
echo '/vendor/etc/audio(/.*)?  u:object_r:vendor_configs_file:s0' \
    >> device/nxp/imx95_evk/sepolicy/file_contexts

# 5. Rebuild sepolicy:
m selinux_policy

# 6. Phân biệt permissive vs enforcing:
# permissive: log denials, allow access (development)
# enforcing: deny access, log (production)
adb shell getenforce  # Enforcing / Permissive
adb shell setenforce 0  # Temporary permissive (debug only, không survive reboot)

# 7. Domain-specific permissive (KHÔNG dùng global):
# device/nxp/imx95_evk/sepolicy/hal_audio.te
# permissive hal_audio_default;  ← BAD PRACTICE in production
# Thay vào đó: fix the actual denial
```

**SELinux Context trong Init RC:**

```rc
# /vendor/etc/init/android.hardware.audio.service.rc
service vendor.audio-hal /vendor/bin/hw/android.hardware.audio.service
    class hal
    user audioserver
    group audio
    seclabel u:r:hal_audio_default:s0  ← Explicit context

# Type declaration phải có trong .te file:
# type hal_audio_default, domain;
# hal_client_domain(hal_audio_default, hal_audio)
# hal_server_domain(hal_audio_default, hal_audio)
```

### 11.4 TrustZone / TEE (OP-TEE trên i.MX95)

```
ARM TrustZone Memory Separation:

 Non-Secure World (EL0/EL1)          Secure World (EL0/EL1)
┌──────────────────────────┐         ┌────────────────────────┐
│  Android Userspace       │         │  Trusted Applications  │
│  (EL0)                   │ ◄──SMC──│  (TA) (TEE EL0)       │
├──────────────────────────┤         ├────────────────────────┤
│  Linux Kernel (EL1)      │         │  OP-TEE OS (TEE EL1)  │
│  TEE driver:             │         │  - AES/RSA ops         │
│  /dev/tee0               │         │  - Key derivation      │
│  /dev/teepriv0           │         │  - Secure storage      │
└──────────────────────────┘         └────────────────────────┘
                │                               │
                └───────────── ATF BL31 (EL3) ──┘
                               SMC dispatcher
                               - PSCI (power)
                               - SCMI (clock/power)
                               - TEE (OP-TEE entry)

SCM flow:
  Android TEE client → /dev/tee0 (ioctl) → optee_driver.ko
  → SMC to ATF EL3 → ATF routes to OP-TEE EL1(S)
  → OP-TEE loads TA → executes crypto op → returns

Use cases on Android Automotive:
  - Keymaster/Keymint TA: hardware-backed key generation
  - Gatekeeper TA: PIN/pattern verification  
  - DRM TA: Widevine L1 key handling
  - Secure Boot TA: rollback index storage
```

---

