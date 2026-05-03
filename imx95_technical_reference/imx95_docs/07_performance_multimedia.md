# NXP i.MX95 EVK — Performance Optimization & Multimedia Pipeline
> Phần 12–13 của tài liệu NXP i.MX95 Deep-Dive

---

## 12. Performance Optimization

### 12.1 Boot Time Optimization

**Phân tích và optimization theo layer:**

```bash
# ─── Kernel Boot Optimization ─── 

# 1. Loại bỏ initcall chậm: xác định thủ phạm
# Thêm vào bootargs: initcall_debug
adb shell dmesg | grep "initcall" | awk '{print $NF, $0}' | sort -rn | head -10

# 2. Compile built-in thay vì modules (cho critical path):
# CONFIG_FSL_SAI=y   (thay vì =m)
# CONFIG_SND_SOC_TAS5828M=y
# → Giảm module loading time ~50-100ms mỗi module

# 3. Async probe (parallel driver init):
# In DTS:
# &sai3 { status = "okay"; ... };
# Thêm: fsl,dma-name = "sdma"; /* triggers async probe */
# Hoặc dùng: probe_type = PROBE_PREFER_ASYNCHRONOUS;

# 4. Reduce kernel log (loglevel=0 trong production):
# bootargs: loglevel=0 quiet

# ─── Android Init Optimization ─── 

# 5. Preloaded classes trimming
# /system/etc/preloaded-classes: 6000+ classes
# Profiling: adb shell am start -S com.example.app
# Xem dalvik: "Loading class ..." trong logcat -t 3000 --pid=$(pidof app)
# Remove unused classes → giảm Zygote preload time

# 6. dex2oat (ahead-of-time compilation):
# Boot time bị ảnh hưởng lớn lần đầu sau OTA (dex2oat chạy background)
# Verify: adb shell ps -A | grep dex2oat
# Tắt dex2oat speed-profile để giảm first-boot time:
# ro.sys.fw.dex2oat_thread_count=4   ← tăng CPU để compile nhanh hơn
# pm.dexopt.boot-after-ota=verify    ← chỉ verify, không recompile hết

# 7. SystemServer service parallelism:
# frameworks/base/services/java/com/android/server/SystemServer.java
# Service parallelism được control bởi:
# SystemServerInitThreadPool (4 threads trong Android 12+)
# Xem: adb shell dumpsys activity | grep "SystemServer timing"
```

**Boot time benchmark trước/sau:**

```
Optimization                          Before   After   Saving
──────────────────────────────────────────────────────────────
SAI/SND built-in (not module)          450ms    50ms    400ms
loglevel=0                             200ms    50ms    150ms
dex2oat threads 1→4                    3500ms  1200ms  2300ms
SELinux permissive→enforcing precomp.   800ms   150ms   650ms
async driver probe                      600ms   200ms   400ms
──────────────────────────────────────────────────────────────
Total                                  5550ms  1650ms  3900ms
```

### 12.2 CPU Scheduler Tuning

```bash
# ─── IRQ Affinity (Critical cho Audio) ─── 

# Problem: DMA completion IRQ share CPU với heavy tasks → audio xrun
# Solution: Pin audio DMA IRQ to dedicated CPU

# Xem IRQ numbers
adb shell cat /proc/interrupts | grep -i "sdma\|sai\|edma"
#  45:    0    0    0    2856    0    0    GIC-0  45  sdma-0
#  47:    0    0    0  125467    0    0    GIC-0  47  fsl-sai3

# Pin SAI3 DMA IRQ (47) đến CPU2 (isolated)
adb shell echo 4 > /proc/irq/47/smp_affinity  # CPU2 (bitmask: 0x4 = CPU2)
adb shell echo 4 > /proc/irq/45/smp_affinity  # SDMA IRQ → CPU2

# Isolate CPU2 để không bị scheduler dùng cho normal tasks:
# Thêm vào bootargs: isolcpus=2 nohz_full=2 rcu_nocbs=2

# Verify:
adb shell cat /proc/irq/47/smp_affinity_list  # "2"
adb shell cat /sys/devices/system/cpu/cpu2/online  # 1

# ─── Real-Time Priority cho AudioFlinger ─── 
# AudioFlinger mixer thread: SCHED_FIFO, prio 3 (mặc định)
# Verify:
adb shell ps -eT -o pid,tid,pri,pcls,name | grep audioserver | head -10
# 456  458  98 FF  audioserver  ← FIFO priority 98

# Tăng rt priority nếu cần (audioflinger_priority):
adb shell setprop af.fast_track_multiplier 1  # Fast track mode

# ─── schedutil tunables ─── 
# Rate limit cho frequency changes:
adb shell cat /sys/devices/system/cpu/cpufreq/policy0/schedutil/rate_limit_us
# 20000  (20ms default)
# Giảm xuống để response nhanh hơn với transient loads:
adb shell echo 4000 > /sys/devices/system/cpu/cpufreq/policy0/schedutil/rate_limit_us

# Hispeed frequency (boost khi load tăng đột biến):
adb shell echo 1200000 > \
    /sys/devices/system/cpu/cpufreq/policy0/schedutil/hispeed_freq
```

### 12.3 perf & simpleperf: CPU Profiling

```bash
# ─── simpleperf (Android wrapper for perf) ─── 

# Profile audioserver for 10 seconds:
adb shell simpleperf record \
    -p $(pidof audioserver) \
    -g \                     # call graph
    -e cpu-cycles,cache-misses \
    --duration 10 \
    -o /data/local/tmp/perf.data

adb pull /data/local/tmp/perf.data
simpleperf report -i perf.data --sort dso,symbol | head -40

# Output mẫu:
# Cmdline: simpleperf record -p 456 ...
# Arch: arm64
# Event: cpu-cycles (type 0, config 0)
#
# Overhead  DSO                      Symbol
# 15.23%    /system/lib64/libc.so   memcpy
# 12.45%    [kernel]                _raw_spin_lock_irqsave
# 8.92%     libaudioflinger.so      AudioMixer::process
# 6.34%     libaudioflinger.so      RecordThread::inputStandby
# 4.21%     [kernel]                fsl_sai_dai_trigger

# ─── Flame graph ─── 
simpleperf report-sample -i perf.data --protobuf -o perf.trace
# → Mở trong Android Studio Profiler hoặc speedscope.app

# ─── perf stat: hardware counter summary ─── 
adb shell simpleperf stat \
    -p $(pidof audioserver) \
    -e cpu-cycles,instructions,cache-references,cache-misses \
    --duration 5

# Output:
# Performance counter statistics:
# 1,234,567,890    cpu-cycles
# 2,345,678,901    instructions              #    1.90 insn per cycle
#     12,345,678    cache-references
#        345,678    cache-misses              #    2.80% of all cache refs
```

---

## 13. Multimedia Pipeline

### 13.1 Audio Pipeline: ALSA → HAL → AudioFlinger

```
Full Audio Pipeline (Playback):

App (MediaPlayer / AudioTrack)
    │ AudioTrack::write()
    │ Shared memory (anonymous mmap, 1MB default)
    ▼
AudioFlinger (audioserver process)
    │
    ├── MixerThread::threadLoop()      [SCHED_FIFO prio 3]
    │       │ Mix all active tracks
    │       │ Apply effects (equalizer, resampler)
    │       ▼
    │   MixerThread::writePeriodUs()  [period: 21ms @ 48kHz/1024 frames]
    │       │ write to HAL output stream
    │       ▼
    ▼
Audio HAL (AIDL: android.hardware.audio.core)
    │ IStreamOut::write(fmq_buffer)   [Fast Message Queue]
    │ FMQ: shared ring buffer between audioserver and HAL
    ▼
StreamOutImpl (vendor HAL, NXP)
    │ tinyalsa: pcm_write()
    │ snd_pcm_writei() → ALSA interface
    ▼
ASoC / ALSA (kernel)
    │ fsl_sai_trigger(SNDRV_PCM_TRIGGER_START)
    │ DMA: sdma_prep_slave_sg() → scatter-gather DMA transfer
    ▼
SAI3 DMA Controller (i.MX95 SDMA)
    │ DMA from DRAM buffer → SAI3 FIFO (32 words)
    ▼
SAI3 Hardware (Synchronous Audio Interface)
    │ I2S serial output: BCLK=3.072MHz, LRCLK=48kHz, DATA=24-bit
    ▼
TAS5828M (I2C config: sample rate, volume, DSP filters)
    │ Digital amplifier
    ▼
Speaker Output

Key timing numbers @ 48kHz, period=1024 frames:
  Period duration: 1024/48000 = 21.33ms
  DMA IRQ: every 21ms (SDMA completion)
  Mix latency: < 10ms (AudioFlinger processing)
  Total HAL latency: ~45ms (typical Android Automotive)
```

**AudioFlinger MixerThread Core:**

```cpp
/* frameworks/av/services/audioflinger/Threads.cpp */

bool AudioFlinger::MixerThread::threadLoop()
{
    while (!exitPending()) {
        
        /* 1. Sleep until next period */
        sleepTime = mNormalFrameCount * 1e9 / mSampleRate;  /* 21ms */
        
        /* 2. Mix all active tracks into mMixBuffer */
        mActiveTracks.process();
        mAudioMixer->process();  /* AudioMixer::process() */
        
        /* 3. Write to HAL */
        ATRACE_BEGIN("write");
        
        /* FMQ (Fast Message Queue) write */
        /* FMQ is shared memory between audioserver and HAL process */
        mOutput->write((char *)mMixBuffer, mixBufferSize);
        
        ATRACE_END();
        
        /* 4. Detect underrun (xrun) */
        if (mBytesWritten == 0) {
            /* HAL returned 0 bytes written → xrun! */
            ALOGW("MixerThread: write returned 0, possible xrun");
        }
    }
    return false;
}

/* Fast Message Queue (FMQ) — AudioFlinger ↔ HAL IPC */
/* frameworks/av/media/libaudiohal/impl/StreamHalAidl.cpp */
status_t StreamOutHalAidl::write(const void* buffer, size_t bytes, size_t* written)
{
    /* FMQ write: lock-free, shared memory ring buffer */
    /* No Binder overhead — critical for audio latency */
    if (!mFmqSendAtomic->write(static_cast<const int8_t*>(buffer),
                                bytes / sizeof(int8_t))) {
        ALOGE("FMQ write failed: ring buffer full (xrun)");
        *written = 0;
        return OK;  /* HAL will report xrun */
    }
    *written = bytes;
    
    /* Signal HAL that data is available */
    mEffectiveMQFlagsMask |= static_cast<uint32_t>(MQFlagsDataReady);
    return OK;
}
```

### 13.2 DRM/KMS & SurfaceFlinger (Display Pipeline)

```
Display Pipeline (Android Automotive: 2 screens common):

App (View/Canvas)
    │ Surface → SurfaceFlinger via Binder
    ▼
SurfaceFlinger (surfaceflinger process)
    │
    ├── Layer composition
    │   ├── GPU composition (RenderEngine/Skia/GLES)
    │   └── HWComposer offload (hardware overlays)
    │
    ▼
HWComposer HAL (android.hardware.graphics.composer)
    │ IComposerClient::presentDisplay()
    │
    ▼
DRM/KMS Driver (kernel)
    │ drm_atomic_commit()  [kernel/drivers/gpu/drm/]
    │
    │ CRTC (display controller)
    │ Plane (overlay hardware)
    │ Encoder (LVDS, MIPI-DSI, HDMI)
    │ Connector (physical output)
    ▼
i.MX95 LCDIF / MIPI DSI Controller
    │ DMA to display FIFO
    ▼
Display Panel (MIPI DSI) / LVDS / HDMI

```

**DRM/KMS atomic commit:**

```c
/* drivers/gpu/drm/drm_atomic_helper.c */

int drm_atomic_helper_commit(struct drm_device *dev,
                              struct drm_atomic_state *state,
                              bool nonblock)
{
    /* Validate atomic state: planes, CRTCs, connectors */
    ret = drm_atomic_helper_prepare_planes(dev, state);
    
    /* Wait for previous commit completion (vblank sync) */
    ret = drm_atomic_helper_wait_for_fences(dev, state, false);
    
    /* Program hardware: CRTC, planes, mode */
    drm_atomic_helper_commit_modeset_disables(dev, state);
    drm_atomic_helper_commit_planes(dev, state, flags);
    drm_atomic_helper_commit_modeset_enables(dev, state);
    
    /* Flip happens at next VBLANK interrupt */
    /* VBLANK IRQ: LCDIF IRQ → drm_handle_vblank() → wake waiters */
    return 0;
}

/* i.MX95 LCDIF driver: */
/* drivers/gpu/drm/imx/lcdif/lcdif-drv.c */
static void lcdif_crtc_atomic_flush(struct drm_crtc *crtc,
                                     struct drm_atomic_state *state)
{
    struct lcdif_crtc *lcdif_crtc = to_lcdif_crtc(crtc);
    
    /* Update framebuffer address registers */
    writel(fb_addr_lo, lcdif_crtc->base + LCDIF_CUR_BUF);  /* base: 0x4AE40000 */
    writel(fb_addr_hi, lcdif_crtc->base + LCDIF_CUR_BUF_HI);
    
    /* Shadow register: take effect at next VBLANK */
    writel(LCDIF_CTRL_SFTRST, lcdif_crtc->base + LCDIF_CTRL);
}
```

### 13.3 V4L2 (Video For Linux): Camera Pipeline

```c
/* Camera pipeline trên i.MX95: */
/* ISP → ISI → V4L2 → Camera HAL → Android Camera2 API */

/* drivers/media/platform/nxp/imx8-isi/ (ISI: Image Sensing Interface) */
/* drivers/media/v4l2-core/v4l2-dev.c */

/* V4L2 buffer flow (DMABUF type): */
/* 1. App/HAL: VIDIOC_REQBUFS → allocate N buffers */
/* 2. Export as DMA-BUF fds */
/* 3. VIDIOC_QBUF → enqueue buffer for camera to fill */
/* 4. Camera DMA fills buffer (zero-copy into DMA-BUF) */
/* 5. VIDIOC_DQBUF → dequeue filled buffer */
/* 6. Pass DMA-BUF fd to GPU for processing (ISP pipeline) */

/* Key ioctl flow: */
struct v4l2_requestbuffers req = {
    .count  = 4,
    .type   = V4L2_BUF_TYPE_VIDEO_CAPTURE,
    .memory = V4L2_MEMORY_DMABUF,
};
ioctl(video_fd, VIDIOC_REQBUFS, &req);

/* Queue buffer: */
struct v4l2_buffer buf = {
    .type   = V4L2_BUF_TYPE_VIDEO_CAPTURE,
    .memory = V4L2_MEMORY_DMABUF,
    .m.fd   = dma_buf_fd,  /* DMA-BUF fd from DMA Heap */
};
ioctl(video_fd, VIDIOC_QBUF, &buf);

/* Debug V4L2: */
adb shell v4l2-ctl --list-devices
adb shell v4l2-ctl -d /dev/video0 --all  /* capabilities, formats */
adb shell v4l2-ctl -d /dev/video0 --set-fmt-video=width=1920,height=1080,pixelformat=UYVY
adb shell v4l2-ctl -d /dev/video0 --stream-mmap=4 --stream-to=/sdcard/capture.raw
```

---

