# Linux Fundamentals Write-up

## What I built
I compiled observation notes detailing the foundational system identity of my local Linux environment. Because my host machine is a Mac, I set up a containerized Linux environment using Docker to inspect the kernel, distribution, and system architecture. I successfully identified the specific Linux kernel and Ubuntu distribution running inside my container, independent of my macOS host.

## Why it's built this way (key decisions)
- **Research focus:** I answered the core questions regarding system identity by investigating my kernel version, architecture, and distribution facts. I inspected the distribution first (`/etc/os-release`) to understand the user space I was in, followed by the kernel (`uname -a`) to see what was powering it.
- **Surprises:** I expected a standard `x86_64` architecture and a generic Linux kernel, but my output revealed `aarch64` and a `linuxkit` kernel. This was a clear indicator that my environment is running on an Apple Silicon Mac, and Docker is using a specialized lightweight virtual machine (LinuxKit) to provide the shared Linux kernel to the container. 

## How to build it (teach it to the next trainee)
For someone who has never looked under the hood of their machine, here is how to find real evidence of your system's identity:
1. **Ensure you are actually in Linux:** If you are on a Mac, your native terminal uses Darwin (macOS), not Linux. You must start a container first (e.g., `docker run -it ubuntu /bin/bash`). 
2. **Check the distribution:** Run `cat /etc/os-release`. This reads a static file in the filesystem that the OS uses to identify its name and version. Look for `PRETTY_NAME` or `VERSION_ID`.
3. **Check the kernel:** Run `uname -a`. This asks the running system core for its live details. The third word is usually your kernel version, and near the end is your architecture (like `aarch64` or `x86_64`).
4. **Compare the two:** Realize that these are independent facts. The distribution provides your tools and package managers, while the kernel actually talks to the hardware.

## Concepts worth explaining
**Kernel vs. Distro Independence**
A key takeaway from this exercise is that the distribution and the kernel are decoupled. In my lab, the distribution is Ubuntu (which provides the bash shell, apt package manager, and utilities), but the kernel is `linuxkit`, provided by the Docker Desktop host. If I were to switch from a container to an isolated Virtual Machine (like Multipass), the Ubuntu distro files (`/etc/os-release`) could look exactly the same, but the kernel (`uname -a`) would change to reflect the VM's isolated kernel. 

## What tripped me up
Initially, I attempted to use a command (`orb -m ubuntu`) for a tool that was not installed on my system. I also had to navigate the realization that standard Linux commands behave differently or output different results (like `Darwin`) if run directly in the macOS terminal. Utilizing Docker Desktop (Path C) provided the necessary isolated Linux environment to properly execute the lab.

## Checkpoint evidence
Here is the real, raw terminal output used as evidence for my notes:

**`uname -a` Output:**
`Linux 9ae51219e88a 6.12.76-linuxkit #1 SMP Mon Jul 27 16:56:11 UTC 2026 aarch64 GNU/Linux`

**`cat /etc/os-release` Output:**
```text
PRETTY_NAME="Ubuntu 26.04.1 LTS"
NAME="Ubuntu"
VERSION_ID="26.04"
VERSION="26.04.1 LTS (Resolute Raccoon)"
VERSION_CODENAME=resolute
ID=ubuntu
ID_LIKE=debian
HOME_URL="https://www.ubuntu.com/"
SUPPORT_URL="https://help.ubuntu.com/"
BUG_REPORT_URL="https://bugs.launchpad.net/ubuntu/"
PRIVACY_POLICY_URL="https://www.ubuntu.com/legal/terms-and-policies/privacy-policy"
UBUNTU_CODENAME=resolute
LOGO=ubuntu-logo
```

## What I'd do differently
If I started this module over, I would immediately verify my active shell environment (confirming I have a root shell inside a container) before trying to run diagnostic commands. Understanding the boundary between the host OS (macOS) and the guest OS (Linux container) earlier would have streamlined the initial setup process.