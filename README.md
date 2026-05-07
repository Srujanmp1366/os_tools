# KernelFS — OS File System Simulator

A browser-based interactive simulator that demonstrates the inner workings of an operating system's file system. Built as a single-file JavaScript application with no dependencies, KernelFS lets you explore OS concepts hands-on — watching kernel tables update in real time as you perform file operations.

![Light theme dashboard with sidebar filesystem, inode inspector, file pointer tracker, operations bar, per-process and system-wide tables, resource allocation graph, and kernel log](screenshot.png)

---

## 🎯 Purpose

This project was built to visually explain the OS file system concepts from *Operating System Concepts* (Silberschatz, Galvin, Gagne) — Chapter 10: File-System Interface. Instead of reading static diagrams, you interact with a live simulation where every syscall has visible consequences across hardware, memory, and kernel data structures.

---

## 🚀 Getting Started

No installation, no build step, no dependencies.

```bash
git clone https://github.com/your-username/kernelfs.git
cd kernelfs
open fs_simulator.html   # or just double-click the file
```

Works in any modern browser (Chrome, Firefox, Edge, Safari).

---

## 📐 Layout

```
┌─────────────────────────────────────────────────────────────────┐
│  TOPBAR — KernelFS logo · kernel status · demo buttons · clock  │
├──────────────┬──────────────────────────────┬───────────────────┤
│              │  INODE INSPECTOR             │  MEMORY PAGES     │
│  FILESYSTEM  │  (13 file attributes)        │                   │
│  SIDEBAR     ├──────────────────────────────┤  RESOURCE         │
│              │  FILE POINTER TRACKER        │  ALLOCATION       │
│  proc_A      │  (per-process position bars) │  GRAPH            │
│  proc_B      ├──────────────────────────────┤                   │
│  proc_C      │  OPERATIONS BAR              │  KERNEL LOG       │
│              ├──────────────────────────────┤                   │
│  DISK BLOCKS │  PER-PROCESS TABLE │ SYS-WIDE│                   │
└──────────────┴──────────────────────────────┴───────────────────┘
```

---

## 🧠 Concepts Demonstrated

### File Attributes (Inode Inspector)
Every file exposes its full inode data when selected:

| Attribute | Description |
|---|---|
| **Name** | Symbolic name — the only human-readable identifier |
| **Inode #** | Unique numeric identifier used internally by the kernel |
| **Type** | File type: text, binary, audio, video, log, exec, etc. |
| **Size** | Current size in bytes/KB/MB/GB |
| **Disk Blocks** | Physical block addresses on `/dev/sda1` |
| **Protection** | `rwxrwxrwx` permission bits displayed visually + octal |
| **Created / Modified** | Timestamps |
| **Open Count** | Number of processes currently holding this file open |
| **Lock** | Which process holds an exclusive lock, if any |
| **Location** | Physical disk address |
| **Link Count** | Hard link count |
| **Dirty** | Whether in-memory data has not yet been flushed to disk |

---

### File Operations
All six POSIX basic file operations are implemented, plus extras:

| Syscall | What it demonstrates |
|---|---|
| `open()` | Allocates a file descriptor, adds entry to both tables, checks permissions |
| `read()` | Advances the per-process file pointer, checks read permission and lock status |
| `write()` | Advances pointer, grows file size, marks pages dirty, detects race conditions |
| `seek()` | Repositions the file pointer to a random byte offset (lseek) |
| `truncate()` | Resets file size to zero while preserving all inode attributes |
| `lock()` | Acquires / releases an exclusive advisory lock (`flock LOCK_EX`) |
| `close()` | Frees fd, decrements open count, flushes dirty pages, removes table entry |
| `chmod()` | Interactive permission bit editor with real-time octal display |
| `create()` | Allocates a new inode, assigns disk blocks, sets default permissions |
| `unlink()` | Removes directory entry and frees inode (blocked if file is open) |

---

### Two-Level Open-File Table
The most important structural concept in the simulator:

**Per-Process Open-File Table** — one row per open file per process, containing:
- File descriptor number (`fd`)
- Access mode (`ro` / `rw` / `wo`)
- **File pointer** — the current read/write position, unique to each process
- Flags

**System-Wide Open-File Table** — one row per unique open file across all processes, containing:
- Inode number
- Current size
- Open count
- Lock status
- Dirty flag

This separation is why three processes can open the same file and each maintain an independent position — the file pointer lives in the per-process table, not on disk.

---

### File Pointer Tracker
A dedicated visual panel shows each process's current position inside the selected file as an animated progress bar. As you call `read()`, `write()`, and `seek()`, you can watch the needles move independently — making it clear why the per-process table exists.

---

### Permission System & Access Denial

Files carry `rwxrwxrwx` permission bits. The simulator enforces them:

- **proc_A** is treated as the **owner** (uses owner bits)
- **proc_B / proc_C** are treated as **group/other** (uses group bits)

Attempting `open()`, `read()`, or `write()` without the appropriate bit set triggers an `EACCES: Permission Denied` modal explaining exactly which bit failed and how the kernel checks it. The `chmod()` operation opens an interactive bit-toggle panel where you can flip individual `r`, `w`, `x` bits for owner, group, and others, with the octal representation updating live.

---

### File Locking

`lock()` acquires or releases an exclusive advisory lock (`LOCK_EX`). Once locked:
- Other processes attempting `read()` or `write()` receive `EAGAIN`
- The resource allocation graph immediately shows the hold edge in red
- Disk blocks for the locked file pulse red

Calling `lock()` again from the same process releases it (`LOCK_UN`).

---

### Deadlock Detection

When two processes each hold a lock and attempt to acquire the other's resource, the simulator detects the **circular wait** and shows a detailed modal explaining all four Coffman conditions:

1. Mutual exclusion
2. Hold-and-wait
3. No preemption
4. Circular wait

The **Resource Allocation Graph** renders the cycle visually — solid red arrows for "holds", dashed blue arrows for "requests". A deadlock status indicator appears below the graph.

Click **▸ Deadlock** in the topbar to trigger an automatic demonstration with `proc_A ↔ proc_B`.

---

### Race Condition Detection

When two or more processes have the same file open for writing simultaneously **without a lock**, any `write()` call triggers a race condition warning explaining:

- Why writes interleave non-atomically
- How independent file pointers cause overwrites
- The TOCTOU (Time-of-Check to Time-of-Use) problem
- How `flock(LOCK_EX)` or `O_APPEND` prevents it

Click **▸ Race Cond.** in the topbar to open all three processes on the same file simultaneously.

---

### Hardware Visualization

**Disk Block Grid** — 60 blocks representing `/dev/sda1`. Each file occupies specific blocks which light up by color:
- 🔵 Blue — in use
- 🔴 Red (pulsing) — locked
- 🟡 Amber — dirty (modified, not yet flushed)
- Grey — free

**Virtual Memory Pages** — 32 pages showing kernel space, user space, page cache, and swap usage. Updates dynamically as files are opened and closed.

---

### Dirty Pages & Writeback

When `write()` is called, data goes to the **page cache** (RAM) first. The file is marked **dirty**. On `close()` when the open count hits zero, the kernel flushes dirty pages to disk (simulated with a short delay and a "flushing…" log message). This demonstrates why:
- Writes appear fast (they go to RAM)
- Crashes before flush can lose data
- `fsync()` exists for critical applications

---

## 📁 File Types Included

The simulator ships with a realistic filesystem including:

| File | Type | Notes |
|---|---|---|
| `kernel.log` | Log | World-readable |
| `passwd` | Config | Owner-only (`600`) |
| `image.raw` | Binary | Large file, many blocks |
| `script.sh` | Executable | Execute bit set |
| `config.json` | JSON | Used in deadlock demo |
| `report.pdf` | PDF | Read-only (`444`) |
| `notes.txt` | Text | Used in deadlock + race demos |
| `database.db` | Database | Owner-only (`600`) |
| `lecture.mp3` | **Audio** | 5MB, 10 disk blocks |
| `ambient.wav` | **Audio** | 18MB, 7 disk blocks |
| `demo_os.mp4` | **Video** | 100MB, 5 disk blocks |
| `lecture2.mkv` | **Video** | 200MB, 5 disk blocks, read-only |

You can also create any of these types dynamically using `create()`.

---

## 🎮 Usage Guide

### Basic Flow
1. **Select a file** from the sidebar — inode attributes populate immediately
2. **Select a process** (`proc_A`, `proc_B`, or `proc_C`) using the chips
3. **Run operations** from the ops bar — `open()` first, then `read()` / `write()` / etc.
4. Watch the **two tables**, **file pointer tracker**, **disk blocks**, and **kernel log** update

### Try These Scenarios

**Permission denial:**
> Select `passwd` → switch to `proc_B` → try `open()` → observe `EACCES`

**File locking:**
> Select `notes.txt` → open as `proc_A` → `lock()` → switch to `proc_B` → open → try `write()` → observe `EAGAIN`

**Independent file pointers:**
> Open `lecture.mp3` as all three processes → `read()` multiple times from each → watch the pointer tracker bars diverge

**Dirty writeback:**
> Open any writable file → `write()` → observe dirty flag + amber disk blocks → `close()` → watch flush sequence

**Deadlock:**
> Click **▸ Deadlock** in topbar → observe RAG cycle → dismiss → manually `close()` files to recover

**Race condition:**
> Click **▸ Race Cond.** in topbar → try `write()` from any process → observe warning

---

## 🗂️ Project Structure

```
kernelfs/
├── fs_simulator.html    # Entire application — HTML + CSS + JS in one file
└── README.md
```

---

## 🛠️ Technical Details

- **Zero dependencies** — pure HTML, CSS, and vanilla JavaScript
- **Single file** — the entire simulator is `fs_simulator.html`
- **Canvas-based RAG** — the resource allocation graph is drawn on an HTML5 `<canvas>` element
- **No bundler / no framework** — open in any browser, works offline
- **~1,400 lines** of hand-written code

---

## 📚 Reference

Based on concepts from:

> Silberschatz, A., Galvin, P. B., & Gagne, G. (2018). *Operating System Concepts* (10th ed.). Wiley. — Chapter 10: File-System Interface

Key sections covered:
- §10.1 File Concept (attributes, operations, types)
- §10.1.2 File Operations (the six basic syscalls)
- §10.1.3 Open-File Tables (per-process vs system-wide)
- §10.1.4 File Locking
- Deadlock from Chapter 8

---

## 📄 License

MIT License — free to use, modify, and distribute.
