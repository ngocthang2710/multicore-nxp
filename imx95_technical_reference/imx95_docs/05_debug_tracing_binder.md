# NXP i.MX95 EVK — Debug, Tracing & Binder IPC
> Phần 8–9 của tài liệu NXP i.MX95 Deep-Dive

---

## 8. Debug End-to-End & Tracing (Cross-layer)

### 8.1 Kiến Trúc Tracing Stack

```
User Space App
     │  atrace / Trace.beginSection()
     ▼
Android Trace System (libcutils/Trace.cpp)
     │  writes to /sys/kernel/tracing/trace_marker
     ▼
Linux ftrace (kernel/trace/)
     │  ring buffer per-cpu
     ▼
Hardware PMU (ARM CoreSight, ETM)

Tool mapping:
  systrace / perfetto  → atrace (Android events) + ftrace (kernel events)
  perf                 → hardware PMU + software counters
  ftrace (tracefs)     → raw kernel function / event tracing
  simpleperf           → Android wrapper for perf
  Perfetto             → successor of systrace, protobuf trace format
```

### 8.2 ftrace: Kernel Function & Event Tracing

**Setup tracefs:**

```bash
# tracefs mount point
adb shell mount | grep tracefs
# tracefs on /sys/kernel/tracing type tracefs

# Hoặc: /sys/kernel/debug/tracing (debugfs)
# Trên Android 12+: /sys/kernel/tracing trực tiếp

# Available tracers
adb shell cat /sys/kernel/tracing/available_tracers
# blk function function_graph irqsoff mmiotrace nop preemptoff wakeup

# Available events (có ~1500+ events)
adb shell ls /sys/kernel/tracing/events/
# audio  binder  block  clk  cpufreq  dma  ext4  filemap
# irq    kmem    mmc    net  power    sched  scmi  thermal
```

**Trace Audio HAL Latency (ftrace function_graph):**

```bash
# Mục tiêu: trace toàn bộ call chain khi AudioFlinger write đến ALSA

adb root
adb shell

# 1. Set tracer
echo function_graph > /sys/kernel/tracing/current_tracer

# 2. Filter chỉ audio-relevant functions (tránh noise)
echo 'fsl_sai_*' > /sys/kernel/tracing/set_ftrace_filter
echo 'snd_pcm_*' >> /sys/kernel/tracing/set_ftrace_filter
echo 'dmaengine_*' >> /sys/kernel/tracing/set_ftrace_filter
echo 'sdma_*' >> /sys/kernel/tracing/set_ftrace_filter

# 3. Enable ALSA PCM events
echo 1 > /sys/kernel/tracing/events/alsa/enable

# 4. Tăng buffer (default 4KB/cpu, tăng lên 64MB)
echo 65536 > /sys/kernel/tracing/buffer_size_kb

# 5. Set trace clock (global time, không phải per-cpu local)
echo global > /sys/kernel/tracing/trace_clock

# 6. Bật trace
echo 1 > /sys/kernel/tracing/tracing_on

# 7. Run audio playback (1 giây)
tinyplay /sdcard/test_48k.wav -D 2 -d 0 -p 1024 -n 4 &
sleep 1

# 8. Stop và lấy trace
echo 0 > /sys/kernel/tracing/tracing_on
cat /sys/kernel/tracing/trace > /sdcard/audio_trace.txt
adb pull /sdcard/audio_trace.txt

# Output mẫu:
# # tracer: function_graph
# #
# # CPU DURATION            FUNCTION CALLS
# |   |   |                 |   |   |   |
#  2) + 23.5 us   |  fsl_sai_trigger() {
#  2)   0.8 us    |    clk_enable();
#  2) + 15.2 us   |    dmaengine_submit() {
#  2)   8.1 us    |      sdma_prep_slave_sg();
#  2)   6.5 us    |      dma_async_issue_pending();
#  2)            }
#  2)            }
```

**Trace Scheduler (EAS task placement):**

```bash
# Trace scheduler wakeup + migration events
echo 1 > /sys/kernel/tracing/events/sched/sched_wakeup/enable
echo 1 > /sys/kernel/tracing/events/sched/sched_migrate_task/enable
echo 1 > /sys/kernel/tracing/events/sched/sched_switch/enable
echo 1 > /sys/kernel/tracing/events/power/cpu_frequency/enable

# Trace only AudioFlinger thread
AFPID=$(adb shell pidof audioserver)
echo ${AFPID} > /sys/kernel/tracing/set_event_pid

# Output mẫu:
# audioserver-456 [002] 1234.567: sched_switch:
#   prev_comm=audioserver prev_pid=456 prev_prio=99
#   next_comm=swapper next_pid=0 next_prio=120
# audioserver-456 [002] 1234.789: cpu_frequency:
#   state=1200000 cpu_id=2
```

**ftrace: IRQ Disable Latency (irqsoff tracer):**

```bash
# Đây là tracer cực hữu ích cho audio underrun debug
# Tìm khoảng thời gian IRQ bị disabled dài nhất

echo irqsoff > /sys/kernel/tracing/current_tracer
echo 1 > /sys/kernel/tracing/tracing_on
# ... reproduce audio underrun ...
echo 0 > /sys/kernel/tracing/tracing_on
cat /sys/kernel/tracing/trace

# Output: trace sẽ show stack trace của hàm nào hold IRQ lâu nhất
# Ví dụ phát hiện bug: spinlock trong DMA completion handler
# held IRQ for 380μs → audio xrun (DMA period = 1024/48000 ≈ 21ms → margin ok)
# Nhưng nếu IRQ disabled > 5ms → burst underrun
```

### 8.3 Perfetto: End-to-End System Trace

Perfetto là công cụ chính thức thay thế systrace từ Android 10.

```bash
# ─── Capture Perfetto trace (30 giây) ───
adb shell perfetto \
  --config :test \
  --txt \
  -o /data/misc/perfetto-traces/trace.pb \
  --time 30s

# Config file chi tiết hơn (ví dụ cho audio debug):
cat << 'EOF' > /tmp/audio_trace_config.pbtx
buffers {
    size_kb: 65536
    fill_policy: RING_BUFFER
}
data_sources {
    config {
        name: "linux.ftrace"
        ftrace_config {
            ftrace_events: "sched/sched_switch"
            ftrace_events: "sched/sched_wakeup"
            ftrace_events: "power/cpu_frequency"
            ftrace_events: "audio/audio_state"
            ftrace_events: "dma/dma_fence_init"
            atrace_categories: "audio"
            atrace_categories: "hal"
            atrace_categories: "view"
            atrace_apps: "android.hardware.audio.service"
            atrace_apps: "audioserver"
            buffer_size_kb: 16384
        }
    }
}
data_sources {
    config {
        name: "linux.process_stats"
        process_stats_config {
            scan_all_processes_on_start: true
            proc_stats_poll_ms: 1000
        }
    }
}
duration_ms: 10000
EOF

adb push /tmp/audio_trace_config.pbtx /data/local/tmp/
adb shell perfetto --config /data/local/tmp/audio_trace_config.pbtx \
    -o /data/misc/perfetto-traces/audio.pb

adb pull /data/misc/perfetto-traces/audio.pb
# Mở tại: https://ui.perfetto.dev (upload .pb file)
```

**Source code của atrace marker:**

```cpp
/* libs/cutils/include/cutils/trace.h */
#define ATRACE_BEGIN(name) atrace_begin(ATRACE_TAG, name)
#define ATRACE_END() atrace_end(ATRACE_TAG)

/* libs/cutils/trace-dev.cpp */
void atrace_begin_body(const char* name) {
    if (CC_UNLIKELY(atrace_is_tag_enabled(ATRACE_TAG))) {
        char buf[ATRACE_MESSAGE_LENGTH];
        ssize_t len = snprintf(buf, sizeof(buf), "B|%d|%s", getpid(), name);
        /* Write to tracefs marker → kernel ring buffer */
        write(atrace_marker_fd, buf, len);
        /* atrace_marker_fd = open("/sys/kernel/tracing/trace_marker", O_WRONLY) */
    }
}

/* AudioFlinger sử dụng: */
/* frameworks/av/services/audioflinger/Threads.cpp */
void AudioFlinger::PlaybackThread::threadLoop_write() {
    ATRACE_BEGIN("write");    /* → "B|PID|write" in trace */
    mOutput->write(mMixBuffer, mNormalFrameCount * mFrameSize);
    ATRACE_END();             /* → "E" in trace */
}
```

### 8.4 Debug Boot Time

```bash
# ─── Phân tích boot time ───

# 1. Kernel init timing (từ dmesg)
adb shell dmesg | grep "initcall" | sort -t'+' -k2 -n | tail -20
# Tìm initcall nào chậm nhất:
# [    2.345] initcall fsl_sai_driver_init returned 0 after 125000 usecs

# 2. Android boot event timestamps
adb shell logcat -d -b events | grep "boot_progress"
# 01-01 00:00:02.100 I boot_progress_start: 2100
# 01-01 00:00:03.200 I boot_progress_preload_start: 3200
# 01-01 00:00:04.500 I boot_progress_preload_end: 4500
# 01-01 00:00:06.100 I boot_progress_system_run: 6100
# 01-01 00:00:08.300 I boot_progress_pms_start: 8300
# 01-01 00:00:10.100 I boot_progress_pms_ready: 10100
# 01-01 00:00:12.400 I boot_progress_ams_ready: 12400
# 01-01 00:00:13.500 I boot_progress_enable_screen: 13500

# 3. bootstat phân tích
adb shell bootstat -p 2>&1
# Boot reason: reboot,userrequested
# Absolute boot time: 13.5s
# Time after kernel: 11.2s
# Zygote start: +3.1s
# System server ready: +9.3s

# 4. Kernel cmdline optimization
# Thêm vào bootargs:
# initcall_debug    → in mỗi initcall và thời gian
# no_console_suspend → giữ console active trong suspend

# 5. Find top 10 slowest drivers:
adb shell dmesg | grep -oP "initcall \K\S+ returned 0 after \K[0-9]+" \
  | sort -rn | head -10
```

### 8.5 Debug Audio Issue End-to-End

**Scenario: Silent Audio / No Sound**

```bash
# ─── Layer-by-layer isolation ───

# Layer 1: ALSA (kernel)
adb shell cat /proc/asound/cards
adb shell tinyplay /sdcard/test.wav -D 2 -d 0  # bypass HAL hoàn toàn
# Nếu tinyplay OK nhưng Android silent → vấn đề ở HAL hoặc Framework

# Layer 2: HAL
adb shell logcat | grep -E "AudioHAL|IModule|openOutputStream|fmqByteCount"
# Tìm: fmqByteCount=0 → AudioFlinger KHÔNG gửi data xuống HAL
# Tìm: openOutputStream failed → HAL init lỗi

# Layer 3: AudioFlinger
adb shell dumpsys media.audio_flinger | grep -E "Output|Mixer|State"
# "Output thread 0x... type 0 (MIXER)" → thread type
# "standby" → thread đang sleep (không có active stream)

# Layer 4: AudioPolicyService
adb shell dumpsys media.audio_policy | grep -E "output|route|active"

# Layer 5: CarAudioService
adb shell dumpsys car_service | grep -E "gain|port|patch|AudioPatch"

# ─── Lỗi thường gặp và root cause ───
# 1. fmqByteCount=0:
#    RC: ro.android.car.audio.enableaudiopatch=false
#    Fix: set true → CarAudioService gọi setPortGain() → ALSA volume unmuted

# 2. openOutputStream EINVAL:
#    RC: audio_policy_configuration.xml sai sample rate hoặc format
#    Fix: match với ALSA PCM hw params

# 3. ALSA xrun (underrun):
#    RC: IRQ latency quá cao (xem irqsoff tracer)
#    RC: Period size quá nhỏ
#    Fix: tăng period_count, check IRQ affinity cho DMA

# 4. SAI clock not running:
#    RC: audio_blk_ctrl clock không được enable
#    Fix: thêm assigned-clocks vào SAI DTS node
```

### 8.6 ANR Debug (Application Not Responding)

```bash
# ANR xảy ra khi:
# - Input event không được xử lý trong 5 giây
# - Broadcast không complete trong 10 giây (foreground)
# - Service không start/bind trong 20 giây

# ─── ANR trace file ───
adb pull /data/anr/anr_*.txt  # hoặc
adb pull /data/anr/anr_latest.txt

# Phân tích: tìm thread "main" bị block
# "main" prio=5 tid=1 Blocked
# | waiting to lock <0x0abc1234> held by tid=15
# tid=15 "Binder:456_3" — binder call đang pending

# ─── Binder timeout detection ───
adb shell dumpsys activity | grep -E "ANR|timeout|blocked"

# ─── watchdog framework ───
# com/android/server/Watchdog.java
# Monitors: foreground thread, AMS, WMS, InputDispatcher
# Timeout: 30s default

# ─── ftrace để catch ANR root cause ───
echo 1 > /sys/kernel/tracing/events/binder/binder_transaction/enable
echo 1 > /sys/kernel/tracing/events/binder/binder_transaction_received/enable
echo 1 > /sys/kernel/tracing/tracing_on
# ... reproduce ANR ...
# Tìm binder transaction nào không có matching reply
```

---

## 9. Binder IPC Deep Dive

### 9.1 Kiến Trúc Tổng Quan

```
Client Process                    Server Process
┌─────────────────┐               ┌─────────────────┐
│  IBinder proxy  │               │  BBinder impl   │
│  (stub-gen AIDL)│               │  (real service) │
│                 │               │                 │
│  transact()     │               │  onTransact()   │
└────────┬────────┘               └────────▲────────┘
         │ ioctl(BC_TRANSACTION)            │ ioctl(BR_TRANSACTION)
         ▼                                 │
┌────────────────────────────────────────────────────┐
│           /dev/binder (kernel driver)               │
│   drivers/android/binder.c                         │
│                                                     │
│   binder_transaction()                              │
│   ┌─────────────────────────────────────────────┐   │
│   │  binder_node (server BBinder object)        │   │
│   │  binder_ref  (client handle → node mapping) │   │
│   │  binder_proc (per-process state)            │   │
│   │  binder_thread (per-thread state)           │   │
│   └─────────────────────────────────────────────┘   │
│                                                     │
│   Memory: mmap() shared region (1MB default)        │
│   Client VA ──→ same physical pages ←── Server VA   │
└────────────────────────────────────────────────────┘

Zero-copy mechanism:
  Client writes to its mmap region
  Kernel maps SAME physical pages into server's address space
  No memcpy between processes — O(1) data transfer
```

### 9.2 `binder_transaction()`: Chi Tiết Thực Thi

```c
/* drivers/android/binder.c */

static void binder_transaction(struct binder_proc *proc,
                                struct binder_thread *thread,
                                struct binder_transaction_data *tr,
                                int reply,
                                binder_size_t extra_buffers_size)
{
    struct binder_transaction *t;
    struct binder_work *tcomplete;
    binder_size_t buffer_offset = 0;
    
    /* ─── 1. Tìm target process/thread ─── */
    if (reply) {
        /* BC_REPLY: target = thread đang waiting cho reply này */
        in_reply_to = thread->transaction_stack;
        target_thread = in_reply_to->from;
        target_proc = target_thread->proc;
    } else {
        /* BC_TRANSACTION: target = server process */
        if (tr->target.handle) {
            /* Lookup handle → binder_ref → binder_node */
            ref = binder_get_ref_olocked(proc, tr->target.handle, true);
            target_node = ref->node;
        } else {
            /* handle=0 → ServiceManager itself */
            target_node = context->binder_context_mgr_node;
        }
        target_proc = target_node->proc;
        
        /* Nếu server có thread đang menunggu: wake it */
        /* Nếu không: enqueue vào proc->todo */
        if (!list_empty(&target_proc->waiting_threads)) {
            target_thread = container_of(
                target_proc->waiting_threads.next,
                struct binder_thread, waiting_thread_node);
        }
    }
    
    /* ─── 2. Allocate buffer trong server's mmap region ─── */
    /* binder_alloc_new_buf() allocates from the shared mmap area */
    /* Buffer physically SHARED between client and server — zero copy */
    t->buffer = binder_alloc_new_buf(
        &target_proc->alloc,
        tr->data_size + tr->offsets_size + extra_buffers_size,
        !reply && (t->flags & TF_ONE_WAY));
    
    /* ─── 3. Copy data từ client user space vào shared buffer ─── */
    /* copy_from_user() → client vaddr → shared physical pages */
    /* Server có thể đọc ngay từ server vaddr của cùng physical pages */
    if (copy_from_user(t->buffer->data,
                       (const void __user *)(uintptr_t)tr->data.ptr.buffer,
                       tr->data_size)) {
        return_error = BR_FAILED_REPLY;
        goto err_copy_data_failed;
    }
    
    /* ─── 4. Fixup object references ─── */
    /* Scan through binder objects (IBinder, file descriptors) in data */
    /* Translate references from client space → server space */
    for (buffer_offset = ...; n < offsets_end; n++) {
        switch (hdr->type) {
        case BINDER_TYPE_BINDER:
            /* Client passing its own BBinder → create node in kernel */
            ret = binder_translate_binder(fp, t, thread);
            break;
        case BINDER_TYPE_HANDLE:
            /* Client passing a handle → translate to server's handle */
            ret = binder_translate_handle(fp, t, thread);
            break;
        case BINDER_TYPE_FD:
            /* File descriptor passing: dup fd in target process */
            ret = binder_translate_fd(fp->fd, t, thread, in_reply_to);
            break;
        }
    }
    
    /* ─── 5. Enqueue transaction ─── */
    t->work.type = BINDER_WORK_TRANSACTION;
    if (target_thread) {
        list_add_tail(&t->work.entry, &target_thread->todo);
        wake_up_interruptible_sync(&target_thread->wait);  /* Wake server */
    } else {
        list_add_tail(&t->work.entry, &target_proc->todo);
        wake_up_interruptible_all(&target_proc->wait);
    }
    
    /* ─── 6. Client waits for reply (synchronous call) ─── */
    if (!reply && !(t->flags & TF_ONE_WAY)) {
        /* Client thread blocks here until server replies */
        binder_thread_read(proc, thread, bwr.read_buffer,
                           bwr.read_size, &bwr.read_consumed, filp->f_flags);
    }
}
```

**`binder_thread_read()` — Server Side:**

```c
/* drivers/android/binder.c */
static int binder_thread_read(struct binder_proc *proc,
                               struct binder_thread *thread,
                               binder_uintptr_t binder_buffer,
                               size_t size,
                               binder_size_t *consumed,
                               int non_block)
{
    /* Server thread blocks waiting for work */
    /* Called from: ioctl(BINDER_WRITE_READ) with BC_ENTER_LOOPER */
    
retry:
    wait_for_proc_work = binder_available_for_proc_work_ilocked(thread);
    
    if (wait_for_proc_work) {
        /* Server thread is idle — wait for new transaction */
        ret = wait_event_freezable_exclusive(
            proc->wait,
            binder_has_proc_work(proc, thread));
    }
    
    /* Process work items from todo list */
    while (1) {
        struct binder_work *w = NULL;
        
        /* Get next work item */
        if (!list_empty(&thread->todo))
            w = list_first_entry(&thread->todo, struct binder_work, entry);
        else if (!list_empty(&proc->todo))
            w = list_first_entry(&proc->todo, struct binder_work, entry);
        
        switch (w->type) {
        case BINDER_WORK_TRANSACTION: {
            struct binder_transaction *t = container_of(w, ...);
            
            /* Write BR_TRANSACTION to server's read buffer */
            /* Server userspace receives this via ioctl result */
            cmd = BR_TRANSACTION;
            put_user(cmd, (uint32_t __user *)ptr);
            
            /* Write transaction data pointer */
            /* tr.data.ptr.buffer points INTO server's mmap region */
            /* (same physical memory client wrote to — zero copy) */
            put_user_preempt_disabled(&tr, (struct binder_transaction_data __user *)ptr);
            
            /* Server will call onTransact() then BC_REPLY */
            break;
        }
        }
    }
}
```

### 9.3 Thread Pool Management

```cpp
/* frameworks/native/libs/binder/ProcessState.cpp */

/* Android process tạo Binder thread pool khi start */
void ProcessState::startThreadPool() {
    AutoMutex _l(mLock);
    if (!mThreadPoolStarted) {
        mThreadPoolStarted = true;
        spawnPooledThread(true);  /* Spawn main pool thread */
    }
}

void ProcessState::spawnPooledThread(bool isMain) {
    sp<Thread> t = new PoolThread(isMain);
    t->run(makeBinderThreadName().c_str());
    /* Thread name: "Binder:PID_N" (N = thread index) */
}

/* PoolThread::threadLoop() → IPCThreadState::joinThreadPool() */
/* → ioctl(BINDER_WRITE_READ) with BC_ENTER_LOOPER */
/* → Blocks in binder_thread_read() waiting for work */

/* Kernel side: max threads per process */
/* Default: 15 threads (BINDER_DEFAULT_MAX_THREADS) */
/* Configurable via: ioctl(BINDER_SET_MAX_THREADS, &max) */
/* AudioServer: typically 16-32 threads */
/* ServiceManager: 1 thread (single-threaded by design) */

/* Thread pool exhaustion detection: */
adb shell cat /proc/$(pidof audioserver)/status | grep Threads
# Threads: 28  ← nếu tăng liên tục → binder thread leak
```

**Transaction Tracing:**

```bash
# Xem binder transaction statistics
adb shell cat /sys/kernel/debug/binder/stats
# proc: 456 (audioserver)
#   incoming transactions: 12847
#   outgoing transactions: 8932
#   peak threads: 12
#   requested threads: 4

# Per-process binder state
adb shell cat /sys/kernel/debug/binder/proc/456
# proc 456
#  threads: 12
#  requested threads: 0+2/15
#  ready threads 8
#  free async space 520192B
#  nodes: 47
#  refs: 23

# Binder deadlock detection:
# Nếu A calls B and B calls A → BC_TRANSACTION from A→B sits in B's todo
# Nhưng B is blocked calling A → A's thread sleeping in binder_thread_read
# → Deadlock: kernel binder detects với transaction chain analysis
adb shell dmesg | grep "binder: deadlock"
```

### 9.4 AIDL vs HIDL vs AIDL-CPP: Trade-off

```
                HIDL (deprecated)      AIDL-C++ (transitional)   AIDL-NDK (current)
──────────────────────────────────────────────────────────────────────────────────
Transport       hwbinder (/dev/hwbinder)  binder (/dev/binder)   binder
Language        HIDL IDL                  AIDL                   AIDL
Runtime         libhidlbase               libbinder              libbinder_ndk
Thread model    Passthrough/Binderized    Binderized only        Binderized only
Stability       HAL stable ABI            versioned AIDL         versioned AIDL
Android version 8.0-12                    13 transitional        13+
Performance     ~same                     ~same                  ~same (ndk overhead smaller)

Khi nào dùng cái gì:
  HIDL: Legacy, Android 11 → đang migrate sang AIDL
  AIDL-NDK: New HAL interfaces (Android 13+), e.g., Audio HAL
  AIDL-Java: Framework → App (e.g., IActivityManager)
  AIDL-C++: Stable C++ API cho vendor code
```

---

