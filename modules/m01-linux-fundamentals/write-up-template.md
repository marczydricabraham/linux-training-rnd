# Linux Fundamentals Write-up

## What was built
Observation notes detailing the foundational system identity of a local Linux environment. Because the host machine is a Mac, a containerized Linux environment is set up using Docker to inspect the kernel, distribution, and system architecture. This setup successfully identifies the specific Linux kernel and Ubuntu distribution running inside the container, independent of the macOS host.

## Why it's built this way (key decisions)

* The EPAV framework was followed: Evaluate (OpenCode explained the brief's Research questions and provided quizzes), Plan (expectations were written and the order of checks was decided), Apply (every command was run manually, as the DoD requires output from the local terminal), and Validate (`/validate` passed all four DoD items).
* The brief's four "Research:" questions were explored using OpenCode as a tutor, utilizing explanations and quizzes rather than having notes generated.
* The commands `uname -a` and `cat /etc/os-release` were run first, because the kernel and distro version are the first two facts to check when something behaves differently across machines.
* **What was surprising:** It was expected that `/etc/os-release` would show when the OS was released, but it actually shows the distro's name, version, and ID. It was also unexpected for the kernel name to include "linuxkit". In a Docker environment on a Mac, the kernel is provided by Docker Desktop's lightweight VM (LinuxKit), while Ubuntu only provides the distribution layered on top of it.
* **What was surprising about running processes:** The shell's parent isn't the macOS terminal app itself, but a Docker container runtime process, because the actual terminal window runs on macOS. The Docker container also runs entirely isolated, meaning native Mac background processes are completely hidden from the Linux environment.

## How to build it (teach it to the next trainee)
For someone recreating this training locally who has never looked under the hood of their machine, here is how to find real evidence of the system's identity:

1. **Install and Configure Docker (Mac Users):** Native macOS terminals use Darwin, not Linux. Download and install Docker Desktop. 
2. **Launch the Container:** Open the terminal, ensure you are in the correct working directory, and run the following command to start an interactive Linux shell:
   `docker run -it ubuntu /bin/bash`
3. **Check the distribution:** Once inside the container (the prompt will change to root), run `cat /etc/os-release`. This reads a static file in the filesystem that the OS uses to identify its name and version. 
4. **Check the kernel:** Next, run `uname -a`. This asks the running system core for its live details. The third word is usually the kernel version, and the architecture (like `aarch64` or `x86_64`) appears near the end.
5. **Compare the two:** Recognize that these are independent facts. The distribution provides the tools and package managers, while the kernel actually talks to the hardware.

## Concepts worth explaining
**Kernel vs. Distro Independence**
A key takeaway from this exercise is that the distribution and the kernel are decoupled. In this lab, the distribution is Ubuntu, but the kernel is `linuxkit`, provided by the Docker Desktop host. If switching from a container to an isolated Virtual Machine (like Multipass), the Ubuntu distro files (`/etc/os-release`) could look exactly the same, but the kernel (`uname -a`) would change to reflect the VM's isolated kernel.

## What tripped up the process
Initially, attempting to use a command (`orb -m ubuntu`) for a tool that was not installed caused an error. It was also necessary to navigate the realization that standard Linux commands behave differently or output different results (like `Darwin`) if run directly in the macOS terminal. Utilizing Docker Desktop provided the necessary isolated Linux environment to properly execute the lab.

## Checkpoint evidence
Here is the real, raw terminal output used as evidence for the notes:

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
HOME_URL="[https://www.ubuntu.com/](https://www.ubuntu.com/)"
SUPPORT_URL="[https://help.ubuntu.com/](https://help.ubuntu.com/)"
BUG_REPORT_URL="[https://bugs.launchpad.net/ubuntu/](https://bugs.launchpad.net/ubuntu/)"
PRIVACY_POLICY_URL="[https://www.ubuntu.com/legal/terms-and-policies/privacy-policy](https://www.ubuntu.com/legal/terms-and-policies/privacy-policy)"
UBUNTU_CODENAME=resolute
LOGO=ubuntu-logo
```

## What to do differently
If starting this module over, it is best to immediately verify the active shell environment (confirming a root shell is active inside a container) before trying to run diagnostic commands. Understanding the boundary between the host OS (macOS) and the guest OS (Linux container) earlier streamlines the initial setup process.