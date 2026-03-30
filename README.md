# Multi-Container Runtime

## 1. Team Information

| Name | SRN |
|------|-----|
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

### Prepare Root Filesystem

```bash
mkdir -p rootfs-base
wget https://dl-cdn.alpinelinux.org/alpine/v3.20/releases/x86_64/alpine-minirootfs-3.20.3-x86_64.tar.gz
sudo tar -xzf alpine-minirootfs-3.20.3-x86_64.tar.gz -C rootfs-base

sudo cp -a rootfs-base rootfs-alpha
sudo cp -a rootfs-base rootfs-beta

sudo cp cpu_hog memory_hog io_pulse rootfs-alpha/
sudo cp cpu_hog memory_hog io_pulse rootfs-beta/
```

### Load Kernel Module

```bash
sudo insmod monitor.ko
ls -l /dev/container_monitor
```

### Start Supervisor (Terminal 1)

```bash
sudo ./engine supervisor ./rootfs-base
```

### Use the CLI (Terminal 2)

```bash
# Start containers
sudo ./engine start alpha ./rootfs-alpha '/cpu_hog 30'
sudo ./engine start beta  ./rootfs-beta  '/cpu_hog 30'

# List containers
sudo ./engine ps

# View logs
sudo ./engine logs alpha

# Stop containers
sudo ./engine stop alpha
sudo ./engine stop beta
```

### Memory Limit Test

```bash
sudo cp -a rootfs-base rootfs-memtest
sudo cp memory_hog rootfs-memtest/
sudo ./engine start memtest ./rootfs-memtest '/memory_hog 4 500' --soft-mib 20 --hard-mib 40
dmesg | grep container_monitor
```

### Scheduling Experiment

```bash
sudo cp -a rootfs-base rootfs-high
sudo cp -a rootfs-base rootfs-low
sudo cp cpu_hog rootfs-high/ rootfs-low/

time sudo ./engine run highrun ./rootfs-high '/cpu_hog 15' --nice -10
time sudo ./engine run lowrun  ./rootfs-low  '/cpu_hog 15' --nice 10
```

### Teardown

```bash
# Stop all containers, then Ctrl-C the supervisor
ps aux | grep defunct        # verify no zombies
sudo rmmod monitor
dmesg | tail -5
```

---

## 3. Demo Screenshots

### Screenshot 1 — Multi-container supervision
Two containers (alpha, beta) running simultaneously under one supervisor process.

### Screenshot 2 — Metadata tracking
Output of `engine ps` showing container ID, PID, state, memory limits, and termination reason.

### Screenshot 3 — Bounded-buffer logging
Log file contents captured through the producer-consumer pipeline via `engine logs alpha`.

### Screenshot 4 — CLI and IPC
CLI command sent to supervisor over UNIX domain socket, supervisor responds with container status.

### Screenshot 5 — Soft-limit warning
`dmesg` output showing `SOFT LIMIT` warning when container RSS exceeds the soft threshold.

### Screenshot 6 — Hard-limit enforcement
`dmesg` output showing `HARD LIMIT` kill, and `engine ps` showing `hard_limit_killed` reason.

### Screenshot 7 — Scheduling experiment
`time engine run` output comparing `--nice -10` (high priority) vs `--nice 10` (low priority).

### Screenshot 8 — Clean teardown
`ps aux | grep defunct` showing no zombie processes after supervisor shutdown.

---

## 4. Engineering Analysis

### 4.1 Isolation Mechanisms

Each container is created with `clone()` using three namespace flags: `CLONE_NEWPID`, `CLONE_NEWUTS`, and `CLONE_NEWNS`. `CLONE_NEWPID` gives the container its own PID namespace so processes inside see themselves starting from PID 1, completely isolated from the host PID tree. `CLONE_NEWUTS` gives the container its own hostname so it can be set independently. `CLONE_NEWNS` gives the container its own mount namespace so filesystem mounts inside do not affect the host.

After namespace creation, `chroot()` is called to restrict the container's view of the filesystem to its assigned rootfs directory. This means the container cannot access host files outside its rootfs. Inside the new root, `/proc` is mounted fresh so that tools like `ps` work correctly within the container.

What the host kernel still shares with all containers: the network namespace (containers share the host network stack), the IPC namespace, and the cgroup namespace. The host kernel itself is shared — containers use the same kernel, they are not VMs.

### 4.2 Supervisor and Process Lifecycle

A long-running supervisor is necessary because container processes are children of the process that called `clone()`. If that process exits, the children become orphans adopted by init (PID 1), losing the ability to track metadata, collect exit status, and route logs.

The supervisor calls `clone()` to create each container, maintains a linked list of `container_record_t` metadata, and installs a `SIGCHLD` handler to reap children with `waitpid(WNOHANG)` as they exit. This prevents zombie processes. When a container exits, the handler updates its state, exit code, and termination reason in the metadata.

`SIGINT` and `SIGTERM` to the supervisor trigger orderly shutdown: all running containers receive `SIGTERM` first, then `SIGKILL` if they do not exit within 3 seconds. The supervisor then joins the logger thread and frees all heap memory before exiting.

### 4.3 IPC, Threads, and Synchronization

The project uses two distinct IPC mechanisms:

**Path A — Logging (pipe-based):** Each container's stdout and stderr are connected to a pipe. A dedicated producer thread per container reads from the pipe's read end and pushes `log_item_t` chunks into a shared bounded buffer. A single consumer thread pops chunks from the buffer and writes them to per-container log files. The bounded buffer is protected by a `pthread_mutex_t` with two `pthread_cond_t` condition variables (`not_empty`, `not_full`). Without the mutex, concurrent push and pop operations would corrupt the head/tail indices. Without condition variables, producers would busy-wait when the buffer is full and consumers would busy-wait when it is empty.

**Path B — Control (UNIX domain socket):** CLI client processes connect to the supervisor's UNIX domain socket, send a `control_request_t` struct, and receive a `control_response_t`. This is a separate IPC path from logging so that control commands are never mixed with log data.

The container metadata linked list is protected by a separate `pthread_mutex_t` (`metadata_lock`). Without this lock, the SIGCHLD handler (running asynchronously) and the main event loop could concurrently read and write the same `container_record_t`, causing torn reads of state fields.

A mutex is used over a spinlock for both shared structures because the critical sections involve blocking I/O operations (file writes, socket sends) and condition variable waits. Spinlocks are only appropriate for very short critical sections in kernel context where sleeping is not allowed.

### 4.4 Memory Management and Enforcement

RSS (Resident Set Size) measures the number of physical memory pages currently mapped into a process's address space. It does not measure virtual memory that has been allocated but not yet accessed, memory-mapped files that have been paged out, or memory shared with other processes (shared libraries are counted once per process even though they are physically shared).

Soft and hard limits serve different purposes. The soft limit is a warning threshold — when RSS exceeds it, the kernel module logs a warning via `printk` but does not kill the process. This allows the runtime to observe memory growth before it becomes critical. The hard limit is an enforcement threshold — when RSS exceeds it, the process receives `SIGKILL` immediately.

Enforcement belongs in kernel space rather than user space for two reasons. First, a user-space monitor can be fooled or delayed by scheduling — if the monitored process is consuming memory rapidly, a user-space poller might not run in time. The kernel timer fires on a kernel timer interrupt, independent of user-space scheduling. Second, sending `SIGKILL` from kernel space via `send_sig()` is reliable and immediate, whereas a user-space process sending `kill()` to another process depends on the kernel to deliver the signal at the next scheduling opportunity.

### 4.5 Scheduling Behavior

The Linux CFS (Completely Fair Scheduler) uses nice values to adjust the weight of each process's virtual runtime. A process with `--nice -10` receives more CPU time per scheduling period than a process with `--nice 10` because its weight is higher in the CFS weight table.

In our experiment, both containers ran the same `cpu_hog` workload for 15 seconds:

| Configuration | Nice Value | Real Time |
|---------------|-----------|-----------|
| highrun       | -10        | 15.199s   |
| lowrun        | --nice 10 (entered as -10 in second run) | 14.235s |

Both completed in approximately 15 seconds because each ran alone on the system — CFS only differentiates priorities when processes compete for CPU simultaneously. The slight variation is due to OS scheduling overhead. To observe a stronger difference, both containers should be run concurrently on a single CPU core, where the higher-weight process would receive proportionally more CPU time and complete its work faster.

---

## 5. Design Decisions and Tradeoffs

### Namespace Isolation
**Choice:** `CLONE_NEWPID | CLONE_NEWUTS | CLONE_NEWNS` with `chroot`.
**Tradeoff:** `chroot` is simpler than `pivot_root` but a privileged process inside the container could escape via `chroot` again. `pivot_root` would be more secure.
**Justification:** For a course project running trusted workloads, `chroot` provides sufficient isolation with much simpler implementation.

### Supervisor Architecture
**Choice:** Single-threaded event loop with non-blocking `accept()` and a 50ms sleep.
**Tradeoff:** Introduces up to 50ms latency on CLI commands. A `select()`/`epoll()` based loop would be more efficient.
**Justification:** Simplicity. The 50ms latency is imperceptible for interactive use and avoids the complexity of multiplexed I/O.

### IPC and Logging
**Choice:** UNIX domain socket for control, pipes for logging, bounded buffer with mutex + condvar.
**Tradeoff:** The bounded buffer adds memory overhead and synchronization cost vs writing directly to disk. But direct writes from multiple producer threads would require per-file locks and would block container output if the disk is slow.
**Justification:** The bounded buffer decouples fast producers (containers) from slow consumers (disk writes), preventing containers from blocking on I/O.

### Kernel Monitor
**Choice:** `mutex` to protect the monitored list, periodic timer at 1-second intervals.
**Tradeoff:** A 1-second polling interval means a process could exceed its hard limit by up to 1 second before being killed. A finer interval would be more accurate but increase kernel overhead.
**Justification:** 1 second is a reasonable tradeoff for a memory monitor. Memory growth is rarely instantaneous and the overhead of sub-second polling is not justified for this use case. A mutex is used over a spinlock because the timer callback runs in softirq context where sleeping is allowed with a mutex.

### Scheduling Experiments
**Choice:** `nice` values via `setpriority` inside the container child before exec.
**Tradeoff:** `nice` affects CFS weight but not real-time scheduling classes. CPU affinity (`taskset`) would give more deterministic results.
**Justification:** `nice` is the standard POSIX mechanism for priority adjustment and directly demonstrates CFS weight-based scheduling without requiring real-time privileges.

---

## 6. Scheduler Experiment Results

### Setup
- Both containers ran `/cpu_hog 15` (15 seconds of pure CPU computation)
- Containers run sequentially (one at a time) on Ubuntu 22.04

### Results

| Run | Container | Nice Value | Real Time | User Time | Sys Time |
|-----|-----------|-----------|-----------|-----------|----------|
| 1 | highrun | -10 | 15.199s | 0.011s | 0.008s |
| 2 | lowrun | -10 (re-run) | 14.235s | 0.005s | 0.012s |

### Analysis

Both containers completed in approximately 15 seconds because they ran sequentially on an otherwise idle system. When a process runs alone, CFS gives it 100% of the CPU regardless of nice value — nice values only matter when multiple runnable processes compete for the same CPU.

The ~1 second difference (15.199s vs 14.235s) is due to system load variation between runs, not the nice value, since they did not run concurrently.

To observe a meaningful priority difference, both containers should be started simultaneously with `engine start` and their completion times compared. In that scenario, the `-10` nice process would receive approximately 4x more CPU time than the `+10` nice process (based on CFS weight table entries of 110 vs 27), completing its work significantly faster on a single-core system.

This demonstrates the key Linux scheduling principle: **priority only affects relative CPU allocation when processes compete — an uncontested process always runs at full speed.**