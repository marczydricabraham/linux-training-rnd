# Linux Lab: Module 6 - Package Management & JDK/Python

This document records the steps taken to install, verify, and manage system and Python packages.

## Step 1: Install the JDK
First, we updated the system's package list and installed the default Java Development Kit.

```bash
root@2c19b7676ba4:~/linux-course/m5# apt update
root@2c19b7676ba4:~/linux-course/m5# apt install -y default-jdk
```

## Step 2: Verify the JDK Installation
We checked the installed Java version to confirm it was ready to use.

```bash
root@2c19b7676ba4:~/linux-course/m5# java -version
openjdk version "25.0.4.1" 2026-08-18
OpenJDK Runtime Environment (build 25.0.4.1+1-1-26.04.4-Ubuntu)
OpenJDK 64-Bit Server VM (build 25.0.4.1+1-1-26.04.4-Ubuntu, mixed mode, sharing)
```

## Step 3: Confirm Package Tracking
We verified that the system's package manager (`apt`) is actively tracking the installed JDK packages.

```bash
root@2c19b7676ba4:~/linux-course/m5# apt list --installed | grep jdk

WARNING: apt does not have a stable CLI interface. Use with caution in scripts.

default-jdk-headless/resolute,now 2:1.25-77 arm64 [installed,automatic]
default-jdk/resolute,now 2:1.25-77 arm64 [installed]
openjdk-25-jdk-headless/resolute-updates,resolute-security,now 25.0.4.1+1-1~26.04.4 arm64 [installed,automatic]
openjdk-25-jdk/resolute-updates,resolute-security,now 25.0.4.1+1-1~26.04.4 arm64 [installed,automatic]
openjdk-25-jre-headless/resolute-updates,resolute-security,now 25.0.4.1+1-1~26.04.4 arm64 [installed,automatic]
openjdk-25-jre/resolute-updates,resolute-security,now 25.0.4.1+1-1~26.04.4 arm64 [installed,automatic]
```

## Step 4: Install and Verify a Python Package
After ensuring `python3-pip` was installed, we installed the `requests` library. We used the `--break-system-packages` flag to bypass the externally-managed-environment restriction in this container, then verified the installation.

```bash
root@2c19b7676ba4:~/linux-course/m5# pip install requests --break-system-packages
root@2c19b7676ba4:~/linux-course/m5# pip show requests
Name: requests
Version: 2.34.2
Summary: Python HTTP for Humans.
Home-page: 
Author: 
Author-email: Kenneth Reitz <me@kennethreitz.org>
License: Apache-2.0
Location: /usr/local/lib/python3.14/dist-packages
Requires: certifi, charset_normalizer, idna, urllib3
Required-by: 
```

## Step 5: Cleanly Remove the Python Package
Finally, we uninstalled the package to clean up the environment.

```bash
root@2c19b7676ba4:~/linux-course/m5# pip uninstall requests --break-system-packages
Found existing installation: requests 2.34.2
Uninstalling requests-2.34.2:
  Would remove:
    /usr/local/lib/python3.14/dist-packages/requests-2.34.2.dist-info/*
    /usr/local/lib/python3.14/dist-packages/requests/*
Proceed (Y/n)? y
  Successfully uninstalled requests-2.34.2
WARNING: Running pip as the 'root' user can result in broken permissions and conflicting behaviour with the system package manager, possibly rendering your system unusable. It is recommended to use a virtual environment instead: https://pip.pypa.io/warnings/venv. Use the --root-user-action option if you know what you are doing and want to suppress this warning.
```