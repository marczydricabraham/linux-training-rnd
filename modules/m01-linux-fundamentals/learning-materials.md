# M1 Learning Materials — Linux Fundamentals

> **Purpose.** A self-contained study guide for the concepts named in
> [`brief.md`](./brief.md). It teaches the material; it does **not** replace the lab. Run the lab against
> your own machine and record what really happened in your own observation notes.
>
> **Audience.** Beginners with no prior Linux experience, working on a **Mac**. Steps assume macOS
> (Apple Silicon or Intel) and a terminal; no server, cloud account, or package installs beyond Docker
> Desktop are required.
>
> **Scope note.** All command output below is **illustrative teaching output**, not a record of a
> completed lab. Replace every example with your own real state before treating anything here as
> evidence.
>
> **How to use this guide.** Each section follows **Understand → Try → Verify → Continue**. The guided
> path takes roughly **60–90 minutes**. M1 is a small, foundational module; the goal is understanding,
> not speed.
>
> **Platform note (macOS).** A Mac runs **macOS**, whose kernel is **Darwin** — it is *not* Linux. To
> work with a real Linux system, a Linux guest is started on the Mac. This guide uses **Docker Desktop**
> (a lightweight LinuxKit virtual machine running an **Ubuntu** container) because it is the smallest
> setup and needs no separate VM manager. WSL2 is a Windows-only feature and does not apply on macOS.
> Where the module brief's Lab step 3 names a **WSL2 ↔ Virtual Machine** switch, this macOS guide teaches
> the equivalent **container ↔ isolated VM** comparison; the underlying claim — that the kernel and the
> distribution are independent facts — is identical either way.

## Contents

1. [Getting a Linux environment on a Mac](#1-getting-a-linux-environment-on-a-mac)
2. [The big picture: kernel, shell, and distro](#2-the-big-picture-kernel-shell-and-distro)
3. [System identity: `uname -a` and `/etc/os-release`](#3-system-identity-uname--a-and-etc-os-release)
4. [The filesystem as an abstraction](#4-the-filesystem-as-an-abstraction)
5. [A live system: processes](#5-a-live-system-processes)
6. [Lab walkthrough](#6-lab-walkthrough)
7. [Best practices and troubleshooting](#7-best-practices-and-troubleshooting)
8. [References](#8-references)

---

## 1. Getting a Linux environment on a Mac

_Estimated: 5–10 minutes._

### Understand

Two words that are easy to mix up, and that matter for the rest of this module:

- The **terminal emulator** is the app window that draws text and captures keystrokes — on a Mac,
  **Terminal.app** (or iTerm). It is only a display and keyboard surface.
- The **shell** is the program that runs *inside* that window, reads the commands typed, and asks the
  kernel to carry them out. On macOS the default shell is `zsh`; inside an Ubuntu container it is
  `bash`.

Because macOS is **Darwin**, not Linux, typing Linux commands directly into Terminal.app gives
macOS/BSD results — not Linux results. A Linux guest is needed for accurate evidence.

### Try

Install **Docker Desktop** for macOS, launch it, and start an Ubuntu container with an interactive
shell:

```bash
docker run -it ubuntu /bin/bash
```

Inside, confirm where the shell is:

```bash
echo "$0"          # the shell program, e.g. /bin/bash
uname -s           # the operating-system (kernel) name: Linux
```

### Verify

- The prompt changes to something like `root@<container-id>:/#`, showing a root shell inside the
  container.
- `uname -s` prints `Linux` (not `Darwin`).

### Continue

Everything from here runs **inside the Ubuntu container**, not in the macOS Terminal.

---

## 2. The big picture: kernel, shell, and distro

_Estimated: 10–15 minutes._

### Understand

- **Kernel** — the core of the operating system and the only part with direct access to hardware (CPU,
  memory, disks, network). It manages memory, decides which program runs next (**scheduling**), and
  talks to devices. Ordinary programs never touch hardware directly; they ask the kernel.
- **Shell** — the interpreter you talk to. It reads a line, splits it into words, expands variables and
  wildcards, then runs the matching program as a new **process**.
- **Distribution (distro)** — a complete bundle built around the kernel: utilities, a package manager
  (e.g. `apt` on Ubuntu), a default shell, and configuration. Distros share the kernel family but differ
  in defaults.

**How a typed command travels to output** (tracing `uname -a`):

1. **Input** — the terminal emulator sends the completed line to the shell.
2. **Parsing** — the shell breaks it into the command name (`uname`) and its argument (`-a`).
3. **Lookup and execution** — the shell finds `uname` on the `$PATH` and starts it as a child process.
4. **System call** — `uname` cannot read system information itself, so it makes a **system call**: a
   controlled request to the kernel, which checks permissions, performs the operation, and returns the
   result.
5. **Output** — the program writes to standard output (`stdout`), connected back to the shell.
6. **Display** — the shell hands the text to the terminal emulator, which draws it on screen.

A **system call** (syscall) is the boundary between *user space* (ordinary programs) and *kernel space*
(the kernel). It is the single controlled doorway that stops a program from damaging the system.

The **shell** and the **terminal emulator** are two different programs: the window can be changed
without changing the shell, and the shell can be changed without changing the window.

### Try

```bash
echo "$SHELL"       # the shell program
echo "$TERM"        # the terminal type the emulator reports
type uname          # is it a built-in or an external program?
uname -a            # the kernel describing itself
```

### Verify

- `type uname` reports a path such as `/usr/bin/uname` (an external program, not a shell built-in).
- Every command follows the same shape: **name + options + arguments**.

### Continue

Move on to reading the two facts `uname -a` and `/etc/os-release` actually report.

---

## 3. System identity: `uname -a` and `/etc/os-release`

_Estimated: 10–15 minutes._

### Understand

- `uname -a` asks the **running kernel** to describe itself: kernel name, hostname, kernel release and
  build date, machine architecture, and OS name. It answers *"which Linux kernel is executing now?"*
- `/etc/os-release` is a **plain text file shipped by the distribution**: distro name, version, ID,
  branding. It answers *"which Linux distribution is installed?"*

They are **independent facts**: `/etc/os-release` is just a file on disk and can be replaced without
changing the kernel, while the kernel can be upgraded without touching the file.

### Try

```bash
uname -a
cat /etc/os-release
```

Illustrative output:

```text
Linux 9ae51219e88a 6.12.76-linuxkit #1 SMP Mon Jul 27 16:56:11 UTC 2026 aarch64 GNU/Linux
```

```text
PRETTY_NAME="Ubuntu 26.04.1 LTS"
NAME="Ubuntu"
VERSION_ID="26.04"
VERSION="26.04.1 LTS (Resolute Raccoon)"
ID=ubuntu
ID_LIKE=debian
```

### Verify

Read the four facts out of the real output:

- **Kernel version** — the third field of `uname -a` (the *kernel release*), e.g. `6.12.76-linuxkit`.
- **Kernel architecture** — near the end of `uname -a`, e.g. `aarch64` (Apple Silicon; `x86_64` on
  Intel).
- **Distro name** — the `NAME=` field of `/etc/os-release`, e.g. `Ubuntu`.
- **Distro version** — the `VERSION=` field, e.g. `26.04.1 LTS`.

The `-linuxkit` suffix shows the kernel came from Docker Desktop's VM, not a bare-metal install.

### Continue

Switching environments changes the **kernel** independently of the **distro**: an isolated VM (e.g.
Multipass) would boot its own guest kernel, so `uname -a` would change, while `/etc/os-release` could
stay identical if the same Ubuntu image is used.

---

## 4. The filesystem as an abstraction

_Estimated: 10–15 minutes._

### Understand

Linux presents far more than documents through the filesystem, so the same tools (`cat`, `ls`) work on
all of it — the principle **"everything is a file"**:

- **Regular files** (`/etc/os-release`) are documents on disk.
- **Device nodes** in `/dev` represent hardware and pseudo-devices.
- **Kernel interfaces** in `/proc` and `/sys` expose live system state.
- **Pipes and sockets** are reached through file descriptors too.

- **`/dev`** holds device nodes: `/dev/sda` (disk), `/dev/null` (discards data), `/dev/zero`, `/dev/random`.
- **`/proc`** is a **virtual filesystem** generated by the kernel at read time: `/proc/cpuinfo`,
  `/proc/meminfo`, `/proc/version`, `/proc/uptime`, and one `/proc/<pid>/` folder per process.

The tree (`/`, `/etc`, `/dev`, `/proc`, …) is an **abstraction over physical storage**. Virtual
filesystems like `/proc`, `/sys`, and `/dev` have **no disk blocks behind them** — they reflect live
kernel state.

### Try

```bash
ls /dev | head -15
cat /proc/version
cat /proc/cpuinfo | head -5
ls /proc | head -10
```

### Verify

- `/proc/version` looks like a kernel description; `/dev` lists names such as `null`, `zero`, `tty`.
- Notice that these files never saved anything to disk — they were produced on demand.

### Continue

The `/proc/<pid>/` folders hint at the final topic: what is running right now.

---

## 5. A live system: processes

_Estimated: 10 minutes._

### Understand

A **process** is a running program; each has a **PID** (process ID), a parent, an owning user, and a
command line. The kernel **creates, schedules, and terminates** processes, gives them memory and CPU
time, and tracks their PIDs. The shell itself is a process; each command run becomes its child.

### Try

```bash
ps aux | head -15
top -b -n 1 | head -20
```

### Verify

- A container is deliberately minimal: only a handful of processes appear (the shell and the current
  command) — far fewer than on a desktop machine.
- Inside a container, PID 1 is typically the shell or the entry point, not a full init system.

### Continue

Record what surprised you (e.g. how few processes there are) — that surprise is exactly the kind of
observation the brief wants.

---

## 6. Lab walkthrough

> **Reminder.** The output above is illustrative. Perform these steps on your own machine and record
> your own raw output.

_Estimated: 15–20 minutes._

**Goal.** Examine your own system and record, from real terminal output, which kernel and which
distribution it is running.

1. Start the Ubuntu container (§1) and run, recording the raw output of each:
   ```bash
   uname -a
   cat /etc/os-release
   ```
2. From that output, identify the **kernel version**, **kernel architecture**, **distro name**, and
   **distro version**, and show where in the output each appears.
3. In one or two sentences, explain what would change in step 1's output if you switched from the
   container path to an isolated VM (or the reverse), showing that kernel and distro are independent.

**Capture.** Use the output of both commands as evidence for your observation notes.

**Definition of done (self-check).**

- [ ] DoD-01 — raw, pasted `uname -a` and `cat /etc/os-release` output from your own terminal.
- [ ] DoD-02 — kernel version, architecture, distro name, and distro version named, with their location
      in the output.
- [ ] DoD-03 — one or two sentences on the WSL2/VM-style switch (container ↔ isolated VM on a Mac),
      showing kernel and distro are independent.
- [ ] DoD-04 — you can answer unprompted, *"what happens between typing a command and seeing output?"*

---

## 7. Best practices and troubleshooting

_Estimated: 5 minutes._

### Best practices

- Treat **kernel and distro version** as the first two facts to check when something behaves
  differently across machines.
- Do not assume **distro-specific defaults** (package manager, shell, service manager) transfer to
  another distro.
- Keep **"the shell"** and **"the terminal emulator"** distinct.
- On a Mac, always gather Linux evidence **inside the guest**, never in the native Terminal.

### Troubleshooting

| Symptom | Likely cause | Fix |
| --- | --- | --- |
| `uname -a` prints `Darwin` | You are in the macOS Terminal, not the Linux guest. | Start the Ubuntu container (§1) and run it there. |
| `docker: command not found` | Docker Desktop is not installed or not running. | Install/launch Docker Desktop, then retry. |
| Kernel name contains `linuxkit` | Expected: Docker Desktop's VM provides the kernel. | This is correct on macOS; explain it in your notes. |
| `/proc` files look empty | Rare; the kernel generates them on read. | Re-run `cat /proc/version`; a minimal image still provides it. |
| Very few processes in `ps aux` | Expected in a container. | Note it as an observation, not a fault. |

---

## 8. References

- [Kernel.org — What is Linux?](https://www.kernel.org/category/about.html) — plain overview of the kernel.
- [Ubuntu manpage — os-release(5)](https://manpages.ubuntu.com/manpages/jammy/en/man5/os-release.5.html) — explains each `/etc/os-release` field.
- [Filesystem Hierarchy Standard (FHS)](https://refspecs.linuxfoundation.org/FHS_3.0/fhs/index.html) — the roles of `/dev`, `/proc`, and the tree.
- [proc(5) — Linux man pages](https://man7.org/linux/man-pages/man5/proc.5.html) — the `/proc` virtual filesystem.
- [Bash Guide for Beginners (TLDP)](https://tldp.org/LDP/Bash-Beginners-Guide/html/) — the shell, parsing, and `$PATH`.
- [The Linux Filesystem Explained (Linux Foundation)](https://www.linuxfoundation.org/blog/blog/classic-sysadmin-the-linux-filesystem-explained) — a readable filesystem tour.
- [ExplainShell](https://explainshell.com/) — paste a command and see what each part does.

For the module's deliverable and checkpoints, see [`brief.md`](./brief.md); for an ordered checklist,
see [`tasks.md`](./tasks.md).
