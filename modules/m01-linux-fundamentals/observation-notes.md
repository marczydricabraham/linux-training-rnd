# M1 Linux Fundamentals — Observation Notes

*These notes were prepared to satisfy the M1 deliverable under the "Technical Writer Helper" approach. The evidence below is intended to be populated with the real terminal output from the trainee's own Linux environment (WSL2, VM, or container) as required by the brief. Each claim is backed by a verifiable source with beginner-friendly references at the end.*

## 1) Kernel, Shell, Distro — What each does, and how a command travels to output

### Kernel
Think of the kernel as the "boss" (or conductor) of your computer. It has direct access to the CPU, memory, disk, and hardware. Applications and user programs don't talk to hardware directly — they ask the kernel to do it for them.

### Shell
The shell is the "interpreter" you talk to. It's a program that reads what you type (commands), figures out what you mean, and asks the kernel to carry it out. Common shells: `bash`, `zsh`, etc. The shell is *software*, not the keyboard/monitor.

### Distro (Distribution)
A Linux distribution is a complete package: the Linux kernel plus system utilities, tools, package manager, default shell/configs, and often a desktop. Examples: Ubuntu, Debian, Fedora, Alpine. Different distros share the same kernel family but differ in defaults (package manager, service manager, etc.).

### How a typed command travels from shell → kernel → output
When you type `uname -a` in your terminal and press Enter:

1. **Input**: Terminal emulator sends the keystrokes to the shell (e.g. bash/zsh).
2. **Parsing**: Shell tokenizes/expands the command line.
3. **Execution**: If it's a built-in, shell runs it; if it's an external program, shell looks up the binary in `$PATH` and starts a new process.
4. **System calls**: The running program needs kernel services (get info about system) — it makes **system calls** (syscalls). A system call is a controlled way for user-space programs to request kernel-level operations (like reading files, getting system info, allocating memory). The kernel checks permissions and performs the operation on hardware.
5. **Return**: Results come back from kernel to the process (via syscalls), process writes to stdout/stderr; the shell displays that in the terminal emulator.
6. **You see output**: The terminal draws characters on screen.

### Shell vs Terminal Emulator (important distinction)
- **Terminal emulator** (terminal app): the window/gui that draws text, handles keystrokes, and connects to a shell. Examples: Terminal.app, GNOME Terminal, Windows Terminal, Alacritty. It's like the "phone screen + keyboard" (display+input).
- **Shell**: the program running inside that window that interprets commands. Examples: `bash`, `zsh`, `fish`. It's like the "operator" you talk to on the other end. They are different things; you can change shells without changing terminals, and vice versa.

**Research answers required by Scope:**
- System call: yes, defined above (controlled interface between user-space and kernel).
- Shell vs terminal emulator: yes, distinguished above.

## 2) System identity — kernel and distro from your own system

You must inspect your own system. Record raw output first (DoD-01), then extract facts with pointers to where they appear (DoD-02).

### Evidence (raw output)

#### `uname -a`
```text
# PASTE REAL OUTPUT FROM: uname -a
# Example placeholder (do not copy from docs): Linux hostname 5.15.133.1-microsoft-standard-WSL2 #1 SMP ... x86_64 GNU/Linux
```

#### `cat /etc/os-release`
```text
# PASTE REAL OUTPUT FROM: cat /etc/os-release
# Example placeholder: NAME="Ubuntu", VERSION="22.04.4 LTS (Jammy Jellyfish)", etc.
```

### Extracted facts (from the pasted output above)
- **Kernel version**: `<paste exact kernel version string from uname -a>` — *where in uname -a*: look for the version after kernel release (e.g. `5.15.133.1...` shows the kernel release version). Note the exact token(s).
- **Kernel architecture**: `<e.g. x86_64 or aarch64>` — *where in uname -a*: usually the machine/arch field near the end (e.g. `x86_64`).
- **Distro name**: `<from /etc/os-release, e.g. Ubuntu>` — *where in /etc/os-release*: `NAME="..."` field.
- **Distro version**: `<from /etc/os-release, e.g. 22.04.4 LTS>` — *where in /etc/os-release*: `VERSION="..."` or `VERSION_ID="..."` (explain which one you chose and why).

### Why `uname -a` and `/etc/os-release` are not the same fact
- `uname -a` reports **kernel** identity: kernel name, hostname, kernel release/version, build date, machine hardware name, OS. It tells you *which Linux kernel* is running.
- `/etc/os-release` is a distro-provided file reporting **distribution** identity: distro name, version, ID, pretty name, sometimes URLs. It tells you *which Linux distribution* is installed (Ubuntu vs Fedora vs Debian) and its version — this is metadata from the distro, not the kernel itself.

Same host can run same kernel but different distros? No. More directly: upgrading the distro packages doesn't necessarily change kernel reported by `uname -a` immediately; switching kernel or running in WSL2/VM changes kernel. Also, some facts differ — e.g. `uname` shows kernel arch; `os-release` shows distro branding.

### Step 3 (Lab): What changes if switching WSL2 ↔ VM (or reverse)?
Kernel and distro are independent facts. Switching between WSL2 and a Virtual Machine will typically change the **kernel** reported by `uname -a` (different kernel build: Microsoft WSL2 kernel vs the VM's guest kernel, possibly different versions/arch in practice), while the **distro** reported by `/etc/os-release` may stay the same if you install the same distro image in both, or may differ if you use different images. Conversely, you could change distros without changing kernel. So in step 1’s output:
- `uname -a` line changes (hostname/kernel release/build/machine may differ).
- `/etc/os-release` fields may remain identical (if same distro+version) or change (if different distro). The key point: they are independent; kernel identity is not determined by distro identity.

*(Populate with specifics once you have actual output from both environments if you switch.)*

## 3) The filesystem as an abstraction — "everything is a file"

Linux treats many resources as if they were files in the filesystem hierarchy, even if they aren't regular documents on disk. This is a powerful abstraction: you can use the same tools (`cat`, `ls`, `read`, `write`) to interact with configuration, devices, kernel info, and processes.

### What does "everything is a file" mean in practice?
- Regular files (`/etc/passwd`) are files.
- Directories are special files.
- Device nodes in `/dev` let you read/write hardware as files.
- Kernel interfaces in `/proc` (and `/sys`) expose runtime state as files.
- Sockets, pipes (FIFOs) are also represented as file descriptors.

It's not literally true for everything in a pedantic sense, but it's a useful Unix design principle: treat I/O uniformly.

### What lives in `/dev`?
`/dev` contains **device files** (nodes) that represent hardware devices and pseudo-devices. Examples: `/dev/sda` (disk), `/dev/null` (discard), `/dev/zero` (zeros), `/dev/random`/`urandom` (random data), `/dev/tty*` (terminals), `/dev/stdin/stdout/stderr`. Reading/writing these files talks to the kernel drivers for those devices.

**Evidence to capture (optional but recommended):**
```text
# ls /dev | head -15  (paste a small sample)
```

### What lives in `/proc`?
`/proc` is a **virtual (pseudo) filesystem** created by the kernel at runtime. It exposes process and kernel information as files. For example: `/proc/cpuinfo` (CPU details), `/proc/meminfo` (memory), `/proc/version` (kernel version), `/proc/<pid>/` directories for each running process (cmdline, status, fd), `/proc/uptime`. These files don't exist on disk; the kernel generates their contents on-the-fly when read.

**Evidence to capture:**
```text
# cat /proc/version
# cat /proc/cpuinfo | head -5
# ls /proc | head -10
```

### Filesystem abstraction vs physical storage
The filesystem hierarchy (FHS-like layout: `/`, `/etc`, `/home`, `/usr`, `/dev`, `/proc`, `/var`) is an **abstraction** layered over physical storage. Physical storage is blocks on disks/SSDs/partitions (managed by filesystems like ext4, XFS). The kernel's VFS (Virtual File System) presents a unified tree view; virtual filesystems like `/proc`, `/sys`, `/dev` have no backing disk blocks — they reflect kernel state. So "what you see in the tree" is not always "files on disk".

## 4) A live system — what's running right now

Investigate what’s actually running on your machine now and note differences from expectations.

### How to list running processes
Common commands (all available in typical shells):
- `ps aux` — snapshot of all processes (BSD-style)
- `ps -ef` — snapshot (POSIX-style)
- `top` — interactive real-time view (q to quit)
- `htop` — nicer interactive view (may not be installed everywhere)
- `pgrep`, `pidof` for specific lookups

The kernel is responsible for process creation, scheduling, memory allocation, and termination. Each process has a PID, parent, user, command line, etc. The shell itself is a process; every command you run (external) becomes a child process.

### Evidence (capture real output)
```text
# ps aux | head -15
# top -b -n 1 | head -20  (batch mode, one sample)
```

### Observations (in your own words)
- What’s running now? Note a few interesting processes (e.g. shell, terminal, system services, browser if any). Avoid sharing sensitive paths if not desired.
- Kernel’s role with them: scheduler decides CPU time, kernel manages syscalls/memory/signals, tracks PIDs.
- Differences vs expectations: What surprised you? (More/fewer processes than expected? Something you didn’t realize runs? etc.)

## 5) Evidence summary (Lab requirements)

- **DoD-01**: Raw pasted output of `uname -a` and `cat /etc/os-release` from your own terminal (above placeholders replaced with real output). No paraphrase.
- **DoD-02**: Kernel version, architecture, distro name, distro version named with pointers to where each appears in the output (see section 2).
- **DoD-03**: Explanation for switching WSL2↔VM (section 2) showing kernel and distro independent.
- **DoD-04**: Self-check answer below.

### Self-check: "What happens between typing a command and seeing output?"
In plain language (use your evidence): type in terminal → terminal sends to shell → shell parses/looks up binary → program makes system calls to kernel → kernel performs operation (talks to hardware/resources) → results returned to program → program writes to stdout → shell+terminal display it. Distinguish shell vs terminal; kernel is the one with hardware access; syscalls are the interface.

*(Write your own 1–2 sentences using the concepts above.)*

## References for Further Study (beginner-friendly)

- [Linux man pages](https://man7.org/linux/man-pages/) — authoritative, includes `uname(1)`, `uname(2)`, `ps(1)`, `proc(5)`. Great to learn how to read man pages.
- [Kernel.org — What is Linux?](https://www.kernel.org/category/about.html) — plain overview of the Linux kernel.
- [The Linux Documentation Project (TLDP)](https://tldp.org/) — beginner-friendly guides (e.g. "Introduction to Linux", "Bash Guide for Beginners").
- [Filesystem Hierarchy Standard (FHS)](https://refspecs.linuxfoundation.org/FHS_3.0/fhs/index.html) — explains `/dev`, `/proc` roles conceptually (can skim for beginners).
- [Ubuntu docs — os-release](https://manpages.ubuntu.com/manpages/jammy/en/man5/os-release.5.html) — explains fields in `/etc/os-release`.
- [Bash Guide for Beginners (TLDP)](https://tldp.org/LDP/Bash-Beginners-Guide/html/) — helps clarify shell vs terminal, parsing, PATH.
- [ExplainShell](https://explainshell.com/) — paste a command and see what each part does (useful for learning).