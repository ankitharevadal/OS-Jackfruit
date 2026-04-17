# Multi-Container Runtime with Kernel-Level Monitoring

## 👥 Team Members

* **Ankitha Revadal** — PES1UG24AM044
* **Chidvilas Adi** — PES1UG25AM802

---

## 📌 Project Overview

This project implements a lightweight Linux container runtime written in C. It supports multiple containers running concurrently under a single supervisor process and integrates a kernel module for monitoring memory usage.

The system combines:

* **User-space container runtime**
* **Kernel-space memory enforcement**

---

## 🧩 Core Components

### 🔹 engine.c (User Space)

* Long-running supervisor process
* Handles CLI commands (`start`, `run`, `ps`, `logs`, `stop`)
* Maintains container metadata
* Implements bounded-buffer logging

### 🔹 monitor.c (Kernel Module)

* Tracks container processes using host PID
* Monitors RSS memory usage
* Enforces:

  * Soft limit → warning
  * Hard limit → SIGKILL

---

## ⚙️ Environment Setup

```bash
sudo apt update
sudo apt install -y build-essential linux-headers-$(uname -r)
```

---

## 🛠️ Build

```bash
cd boilerplate
make
```

---

## 📦 Root Filesystem Setup

```bash
mkdir rootfs-base
wget https://dl-cdn.alpinelinux.org/alpine/v3.20/releases/x86_64/alpine-minirootfs-3.20.3-x86_64.tar.gz
tar -xzf alpine-minirootfs-3.20.3-x86_64.tar.gz -C rootfs-base

cp -a rootfs-base rootfs-alpha
cp -a rootfs-base rootfs-beta
```

---

## 🚀 Running the System

### Start Supervisor

```bash
sudo ./engine supervisor ./rootfs-base
```

### Start Containers

```bash
sudo ./engine start alpha ./rootfs-alpha /bin/sh
sudo ./engine start beta ./rootfs-beta /bin/sh
```

### Monitor

```bash
sudo ./engine ps
sudo ./engine logs alpha
```

### Stop

```bash
sudo ./engine stop alpha
```

---

## 🏗️ Architecture Overview

```
┌────────────────────────────────────────────┐
│              CLI PROCESS                   │
│   engine start / run / ps / logs / stop   │
└───────────────┬────────────────────────────┘
                │ UNIX Socket (Control IPC)
                ▼
┌────────────────────────────────────────────┐
│            SUPERVISOR (engine)             │
│                                            │
│  - Handles commands                        │
│  - Tracks containers                       │
│  - Spawns containers (clone)               │
│                                            │
│  ┌──────────────┐   ┌──────────────────┐  │
│  │ Reader Thread│   │ Logger Thread    │  │
│  │ (Producer)   │   │ (Consumer)       │  │
│  └──────┬───────┘   └──────┬──────────┘  │
│         │                  │             │
│     Pipe (stdout/stderr)   │             │
│         ▼                  ▼             │
│     Container        Bounded Buffer      │
│                                            │
└───────────────┬────────────────────────────┘
                │ ioctl
                ▼
┌────────────────────────────────────────────┐
│           Kernel Module (monitor)          │
│                                            │
│  - Tracks PID list                         │
│  - Checks RSS every 1 sec                  │
│  - Soft limit → log                        │
│  - Hard limit → kill                       │
└────────────────────────────────────────────┘
```

---

## 🔄 IPC Mechanisms

| Communication          | Method             |
| ---------------------- | ------------------ |
| CLI → Supervisor       | UNIX domain socket |
| Container → Supervisor | Pipe               |
| Supervisor → Kernel    | ioctl              |

---

## 🧵 Logging System

* Producers → container output threads
* Consumer → logger thread
* Uses:

  * Mutex locks
  * Condition variables

Ensures:

* No data loss
* No deadlocks
* Thread-safe logging

---

## 🧪 Scheduling Experiment

```bash
sudo ./engine start na ./rootfs "/cpu_hog 15" --nice 0
sudo ./engine start nb ./rootfs "/cpu_hog 15" --nice 19
```

### Result:

* Both run 15 seconds
* Lower nice → more CPU share
* Higher nice → slower progress

### Conclusion:

Linux CFS scheduler distributes CPU based on priority.

---

## 🧠 Memory Monitoring

```bash
sudo ./engine start memtest ./rootfs "/memory_hog 100 1" --soft-mib 4 --hard-mib 8
```

### Behavior:

* Soft limit → warning
* Hard limit → process terminated

---

## 🖼️ Screenshots & Results

### 🔹 Multi-container + Metadata

"C:\Users\Ankitha\Desktop\WhatsApp Image 2026-04-17 at 11.48.02.jpeg"

---

### 🔹 Logging + Scheduling Output

![Screenshot](ADD_SCREENSHOT_1_PATH)

---

### 🔹 Kernel Memory Enforcement (dmesg)

![Screenshot](ADD_SCREENSHOT_2_PATH)

---

## 🧠 Engineering Insights

### Isolation

Uses PID, UTS, and mount namespaces with `chroot`.

### Process Lifecycle

Supervisor ensures proper creation, tracking, and cleanup.

### Synchronization

Mutex + condition variables prevent race conditions.

### Memory Management

RSS used for accurate tracking; kernel enforces limits.

### Scheduling

Nice values influence CPU allocation.

---

## 🧹 Cleanup

```bash
sudo rmmod monitor
```

---

## ✅ Conclusion

This project demonstrates:

* Multi-container management
* IPC mechanisms
* Kernel-user interaction
* Scheduling behavior
* Memory enforcement

It acts as a simplified version of real-world container systems like Docker.
