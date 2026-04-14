# Multi-Container Runtime

## 1. Team Information

| Name | SRN |
|---|---|
| Nishant Hegde | PES1UG24CS681 |
| MD Zubed | PES1UG24CS674 |

---

## 2. Build, Load, and Run Instructions

### Prerequisites

```bash
sudo apt update
sudo apt install -y build-essential linux-headers-$(uname -r)
```

### Build

```bash
cd boilerplate
make
```

This builds all targets in one step: `engine`, `cpu_hog`, `memory_hog`, `io_pulse`, and `monitor.ko`.

### Prepare Root Filesystem

```bash
cd boilerplate
mkdir -p rootfs-base
wget https://dl-cdn.alpinelinux.org/alpine/v3.20/releases/x86_64/alpine-minirootfs-3.20.3-x86_64.tar.gz
sudo tar -xzf alpine-minirootfs-3.20.3-x86_64.tar.gz -C rootfs-base

# Create per-container writable rootfs copies
sudo cp -a rootfs-base rootfs-alpha
sudo cp -a rootfs-base rootfs-beta
sudo cp -a rootfs-base rootfs-memtest
sudo cp -a rootfs-base rootfs-high
sudo cp -a rootfs-base rootfs-low

# Copy workload binaries into each rootfs before launch
sudo cp cpu_hog memory_hog io_pulse rootfs-alpha/
sudo cp cpu_hog memory_hog io_pulse rootfs-beta/
sudo cp memory_hog rootfs-memtest/
sudo cp cpu_hog rootfs-high/
sudo cp cpu_hog rootfs-low/
```

### Load Kernel Module

```bash
sudo insmod monitor.ko

# Verify control device was created
ls -l /dev/container_monitor
```

### Start Supervisor (Terminal 1)

```bash
sudo ./engine supervisor ./rootfs-base
```

The supervisor stays alive in this terminal. All container management happens here.

### Launch Containers (Terminal 2)

```bash
# Start two containers in the background
sudo ./engine start alpha ./rootfs-alpha '/cpu_hog 30' --soft-mib 48 --hard-mib 80
sudo ./engine start beta  ./rootfs-beta  '/cpu_hog 30' --soft-mib 64 --hard-mib 96

# List all tracked containers and their metadata
sudo ./engine ps

# View log output from a container
sudo ./engine logs alpha

# Stop a container
sudo ./engine stop alpha
sudo ./engine stop beta
```

### Run a Container in the Foreground

```bash
# Blocks until the container exits, then prints exit status
sudo ./engine run mycontainer ./rootfs-alpha '/cpu_hog 10'
```

### Memory Limit Test

```bash
sudo ./engine start memtest ./rootfs-memtest '/memory_hog 4 500' --soft-mib 20 --hard-mib 40

# Watch kernel logs for soft/hard limit events
dmesg | grep container_monitor
```

### Scheduling Experiment

```bash
# Launch both simultaneously with different priorities
sudo ./engine start highrun ./rootfs-high '/cpu_hog 30' --nice -10
sudo ./engine start lowrun  ./rootfs-low  '/cpu_hog 30' --nice 10

# After ~15 seconds compare log line counts
sudo ./engine logs highrun | wc -l
sudo ./engine logs lowrun  | wc -l
```

### Teardown

```bash
# Stop all containers
sudo ./engine stop alpha
sudo ./engine stop beta

# Stop supervisor with Ctrl+C in Terminal 1

# Verify no zombie processes
ps aux | grep defunct

# Inspect kernel logs
dmesg | tail -10

# Unload the kernel module
sudo rmmod monitor
```

To run helper binaries inside a container, copy them into the container's rootfs before launch:

```bash
cp workload_binary ./rootfs-alpha/
```

---

## 3. Demo with Screenshots

### Screenshot 1 — Multi-container supervision
![multi-container](Screenshots/01_multi_container.png)
Two cpu_hog containers (alpha, beta) running simultaneously under one supervisor process, confirmed by `ps aux` showing both child PIDs alive.

### Screenshot 2 — Metadata tracking
![ps output](Screenshots/02_ps.png)
Output of `engine ps` showing tracked container metadata: ID, PID, state, soft/hard memory limits, exit code, and termination reason.

### Screenshot 3 — Bounded-buffer logging
![logs](Screenshots/03_logs.png)
Log file contents captured through the producer-consumer pipeline via `engine logs alpha`, showing cpu_hog progress lines written by the container and routed through the bounded buffer to disk.

### Screenshot 4 — CLI and IPC
![cli ipc](Screenshots/04_cli_ipc.png)
A `engine stop alpha` CLI command sent to the supervisor over the UNIX domain socket (Path B), with the supervisor printing its acknowledgement response back to the client.

### Screenshot 5 — Soft-limit warning
![soft limit](Screenshots/05_soft_limit.png)
`dmesg` output showing the `[container_monitor] SOFT LIMIT` warning event emitted by the kernel module when the memtest container's RSS first exceeded its soft threshold.

### Screenshot 6 — Hard-limit enforcement
![hard limit](Screenshots/06_hard_limit.png)
`dmesg` showing the `[container_monitor] HARD LIMIT` kill event, and `engine ps` showing the memtest container with state `killed` and termination reason `hard_limit_killed`.

### Screenshot 7 — Scheduling experiment
![scheduling](Screenshots/07_scheduling.png)
`highrun` (nice -10) vs `lowrun` (nice +10) running concurrently — highrun accumulates more CPU ticks per wall-clock second, demonstrating CFS weight-based scheduling under contention.

### Screenshot 8 — Clean teardown
![teardown](Screenshots/08_teardown.png)
`ps aux | grep defunct` returning no results after supervisor shutdown, and the supervisor printing `[supervisor] exited cleanly.`, confirming all children were reaped and threads joined.

---

## 4. Engineering Analysis

### 4.1 Isolation Mechanisms

Each container is created with `clone()` using three namespace flags: `CLONE_NEWPID`, `CLONE_NEWUTS`, and `CLONE_NEWNS`. `CLONE_NEWPID` gives the container its own PID namespace so processes inside see themselves starting from PID 1, completely isolated from the host PID tree. `CLONE_NEWUTS` gives the container its own hostname so it can be set independently with `sethostname()`. `CLONE_NEWNS` gives the container its own mount namespace so filesystem mounts inside do not propagate to the host.

After namespace creation, `chroot()` is called to restrict the container's filesystem view to its assigned rootfs directory. Inside the new root, `/proc` is mounted fresh with `mount("proc", "/proc", "proc", 0, NULL)` so that tools like `ps` work correctly within the container.

The host kernel still shares the network namespace, IPC namespace, and the kernel itself with all containers. Containers are not VMs — they run on the same kernel, which is why namespace isolation is a software boundary rather than a hardware one.

### 4.2 Supervisor and Process Lifecycle

A long-running supervisor is necessary because container processes are children of the process that called `clone()`. If that process exits immediately after launching a container, the child becomes an orphan adopted by init (PID 1), and the runtime loses the ability to track metadata, collect exit status via `waitpid()`, and route log output.

The supervisor installs a `SIGCHLD` handler that sets a volatile flag. The main event loop checks this flag and calls `waitpid(-1, &status, WNOHANG)` to reap all exited children without blocking. This prevents zombie processes from accumulating. Each child's exit status is used to classify the termination reason: if `stop_requested` was set before the signal, the reason is `stopped`; if the exit signal was `SIGKILL` without a stop request, the reason is `hard_limit_killed`. The supervisor also handles `SIGINT` and `SIGTERM` for orderly shutdown by sending `SIGTERM` to all running containers, waiting up to 3 seconds, then sending `SIGKILL` before joining the logger thread and freeing all resources.

### 4.3 IPC, Threads, and Synchronization

The project uses two distinct IPC mechanisms:

**Path A — Logging (pipes):** Each container's stdout and stderr are connected to a pipe created before `clone()`. A dedicated producer thread per container reads from the pipe's read end and pushes `log_item_t` chunks into a shared bounded buffer. A single consumer thread (the logging thread) pops chunks and writes them to per-container log files under `logs/`. The bounded buffer is protected by a `pthread_mutex_t` with two `pthread_cond_t` condition variables (`not_empty` and `not_full`). Without the mutex, two producers could simultaneously read the item count, both see space available, both advance the head index, and one would silently overwrite the other's data. The `not_full` condition variable prevents producers from busy-waiting when the buffer is full; `not_empty` prevents the consumer from busy-waiting when it is empty.

**Path B — Control (UNIX domain socket):** CLI client processes connect to the supervisor's UNIX domain socket at `/tmp/mini_runtime.sock`, send a `control_request_t` struct, and receive a `control_response_t`. This is a separate IPC path from logging so that control commands are never mixed with log data.

The container metadata linked list is protected by a separate `pthread_mutex_t` (`metadata_lock`). Without this lock, the SIGCHLD handler reaper and the main event loop could concurrently read and write the same `container_record_t`, causing torn reads of the state field and incorrect termination classification.

A mutex is preferred over a spinlock for all user-space shared structures because the critical sections involve blocking I/O and condition variable waits. Spinlocks are appropriate only for very short critical sections where sleeping is not permitted.

### 4.4 Memory Management and Enforcement

RSS (Resident Set Size) measures the number of physical memory pages currently mapped into a process's address space and resident in RAM. It does not measure virtual memory that has been `mmap`'d but not yet accessed (demand paging means pages are only allocated on first touch), memory-mapped files that have been paged out to disk, or memory shared with other processes such as shared libraries (each process counts shared pages independently even though they physically share one copy).

Soft and hard limits serve different purposes. The soft limit is a warning threshold — when RSS exceeds it, the kernel module logs a `printk` warning but does not kill the process. This lets the runtime observe memory growth trends before they become critical. The hard limit is an enforcement threshold — when RSS exceeds it, the process receives `SIGKILL` immediately.

Enforcement belongs in kernel space for two reasons. First, a user-space monitor is subject to scheduling delays: if the monitored process is allocating memory rapidly, the user-space poller may not be scheduled in time to stop it before it exceeds its limit significantly. The kernel timer fires on a hardware timer interrupt independent of user-space scheduling. Second, sending `SIGKILL` from kernel space via `send_sig()` is immediate and cannot be blocked or ignored, whereas a user-space `kill()` call requires a round trip through the kernel and depends on scheduling the signal delivery.

### 4.5 Scheduling Behavior

The Linux CFS (Completely Fair Scheduler) uses nice values to adjust each process's scheduling weight. A process with `nice -10` receives a higher weight in the CFS weight table than a process with `nice +10`, meaning it is allocated proportionally more CPU time per scheduling period when both are runnable.

When two containers are run simultaneously on a single core with different nice values, the higher-weight process (nice -10) completes more CPU work per wall-clock second. This is visible in the log output: `highrun` accumulates more cpu_hog progress lines than `lowrun` in the same elapsed time, because it receives more CPU ticks from the scheduler.

When a process runs alone on an idle system, nice values have no visible effect — the scheduler gives an uncontested process 100% of the CPU regardless of its weight. Priority only matters under contention. This is a fundamental property of CFS: it enforces fairness proportional to weight, not absolute time guarantees.

---

## 5. Design Decisions and Tradeoffs

### Namespace Isolation

**Choice:** `CLONE_NEWPID | CLONE_NEWUTS | CLONE_NEWNS` with `chroot()`.

**Tradeoff:** `chroot()` is simpler than `pivot_root()` but a privileged process inside the container could potentially escape by calling `chroot()` again. `pivot_root()` would be more secure because it changes the root mount point at the VFS level and does not allow escape via `..` traversal.

**Justification:** For a course project running trusted workloads, `chroot()` provides sufficient isolation with significantly simpler implementation. The security difference is irrelevant when the workloads are `cpu_hog` and `memory_hog` rather than adversarial processes.

### Supervisor Architecture

**Choice:** Single-threaded event loop using `select()` with a 100 ms timeout, checking signal flags at the top of each iteration.

**Tradeoff:** Introduces up to 100 ms latency on CLI commands. An `epoll()`-based loop with `signalfd` would eliminate this latency and scale to many more simultaneous connections.

**Justification:** Simplicity. The 100 ms latency is imperceptible for interactive use and the supervisor handles at most a handful of CLI connections. The complexity of `epoll` + `signalfd` is not justified for this workload.

### IPC and Logging

**Choice:** UNIX domain socket for control (Path B), pipes for logging (Path A), bounded buffer with mutex and condition variables.

**Tradeoff:** The bounded buffer adds memory overhead and synchronization cost compared to writing directly to disk from each container. Direct writes would require per-file locks and would block container output if the disk is slow, coupling container performance to I/O latency.

**Justification:** The bounded buffer decouples fast producers (containers writing to their pipe) from the slow consumer (disk writes). A container can never block waiting for disk I/O — it only blocks if the buffer is completely full, which requires all 16 slots to be occupied simultaneously.

### Kernel Monitor

**Choice:** `struct mutex` to protect the monitored list, polling timer at 1-second intervals.

**Tradeoff:** A 1-second polling interval means a process could exceed its hard limit by up to 1 second of memory growth before being killed. A finer interval (100 ms) would be more accurate but would increase kernel timer interrupt overhead.

**Justification:** Memory growth for typical workloads is not instantaneous. A 1-second reaction window is acceptable for catching runaway allocation and keeps kernel overhead minimal. A mutex is used instead of a spinlock because `get_rss_bytes()` calls `get_task_mm()` which acquires `mm->mmap_lock`, a sleeping lock — holding a spinlock while calling a sleeping function is illegal in kernel context.

### Scheduling Experiments

**Choice:** `nice` values applied via `setpriority(PRIO_PROCESS, 0, nice_value)` inside the container child before `execv`.

**Tradeoff:** `nice` values affect CFS weight but not real-time scheduling classes. For more deterministic results, CPU affinity via `taskset` or `sched_setaffinity` combined with pinning both containers to a single core would give cleaner measurements. The `nice` approach relies on the system being otherwise idle during the experiment.

**Justification:** `nice` is the standard POSIX mechanism for priority adjustment and directly demonstrates CFS weight-based scheduling without requiring real-time privileges or CPU topology manipulation. It is the most accessible and portable approach for illustrating scheduler behavior.

---

## 6. Scheduler Experiment Results

### Setup

Both containers ran `/cpu_hog 30` (30 seconds of pure CPU computation) launched simultaneously using `engine start`. The system was an Ubuntu 22.04 VM with 2 vCPUs. Nice values were applied inside the container child process before exec.

```bash
sudo ./engine start highrun ./rootfs-high '/cpu_hog 30' --nice -10
sudo ./engine start lowrun  ./rootfs-low  '/cpu_hog 30' --nice 10
```

### Raw Results

Log line counts after 15 seconds of concurrent execution (each line = 1 second of cpu_hog progress):

| Container | Nice Value | Log lines at t=15s | Approx CPU share |
|---|---|---|---|
| highrun | -10 | 12 | ~68% |
| lowrun | +10 | 6 | ~32% |

CFS weight table entries: nice -10 → weight 110, nice +10 → weight 27. Expected CPU ratio: 110 / (110 + 27) ≈ 80%. Observed ratio (~68%/32%) is slightly less skewed due to scheduling overhead, timer resolution, and the VM hypervisor's own scheduling.

### Analysis

When both containers compete for CPU simultaneously, the CFS scheduler allocates time proportional to each process's weight. The `highrun` container (nice -10, higher weight) receives approximately twice the CPU time of `lowrun` (nice +10, lower weight), completing more computation per wall-clock second and producing more log lines in the same period.

This demonstrates the key Linux scheduling principle: **nice values only affect relative CPU allocation when processes compete for the same CPU**. An uncontested process always runs at full speed regardless of its nice value. CFS enforces proportional fairness — it does not give any process an absolute time guarantee, only a relative share proportional to its weight. This design balances throughput (high-priority work completes faster) with fairness (low-priority work still makes progress and is never starved indefinitely).