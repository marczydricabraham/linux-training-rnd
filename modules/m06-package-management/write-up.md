# Package Management Write-up

## What was built

A verified JDK and Python package install on the Linux guest introduced in M1 — the Docker Desktop LinuxKit
virtual machine running an Ubuntu 26.04.1 LTS container with the **bash** shell (the native macOS Terminal
reports Darwin and is not a Linux environment). Two pieces of software were installed through the two
different categories of package manager, and each was verified against its own manager's records:

- **The JDK** — owned by the **system** package manager, `apt`, which records `default-jdk` `2:1.25-77`
  and the dependency tree it pulled in (`[installed,automatic]`). The runtime reports itself as OpenJDK
  `25.0.4.1`, and the launcher that answers is `/usr/bin/java`.
- **`requests` 2.34.2** — installed through the **language-level** manager, `pip`, into Python 3.14's
  `dist-packages`, bringing `certifi`, `charset_normalizer`, `idna` and `urllib3` with it.

Both installs were then checked with a version check and a provenance check, and the Python package was
removed cleanly through `pip` rather than by deleting files. The JDK remains installed, as intended.

The container runs everything as `root`, so the `sudo` prefix shown in the brief was omitted; on a normal
desktop system it is required for the system-layer commands. The full command record for the lab is kept in
`linux-training-rnd/knowledge/resources/module_6_lab_steps.md`.

## Why it's built this way (key decisions)

- **Why `requests`:** it is a real HTTP client rather than a toy, and it declares four dependencies, so
  dependency resolution is observable in its own record (`Requires: certifi, charset_normalizer, idna,
  urllib3`). It is also a library Python code genuinely needs, which means the language layer is exercised
  for the reason it exists rather than merely to produce output.
- **Why the JDK came from `apt` and the library from `pip`:** the JDK is OS-level software. It installs
  system-wide, is recorded in the dpkg database, and is shared by every user and service on the machine —
  exactly what `apt` exists to manage, and why installing it needs elevated privilege. `requests` is not
  system software: it is a library used by Python code in a particular environment, so `pip`, which tracks
  its own packages in the language's own metadata, owns it. Swapping the two would put each in the wrong
  layer: the JDK is not distributed through PyPI, and a distribution does not version a project's Python
  dependency at the version the project chooses.
- **What made each install confident, rather than "ran without error":** two checks, not one. A **version
  check** asks the tool what it is (`java -version` → `25.0.4.1`); a **provenance check** asks the manager
  whether it recorded the package (`apt list --installed | grep jdk`, `apt policy default-jdk`, `pip show
  requests` → Version and Location). The install commands printing nothing alarming would have proved very
  little here — the JDK install reported that it was *already* the newest version, so the log said nothing
  about which JDK would actually run.
- **Why the system Python was used, and why that is a compromise:** the container's Python is marked
  externally managed (PEP 668), so the lab installed with `--break-system-packages`. That produced a
  working package and a clean `pip show` record, but it is the one place where the lab departed from best
  practice: the guard exists to stop a project library from being merged into the distribution's Python. A
  virtual environment would have isolated `requests` without the flag. The choice is recorded here rather
  than hidden, and the venv route is the one taught in the section below.
- **Method:** the EPAV cycle was followed — **Evaluate** (the brief's research questions reviewed),
  **Plan** (the two layers, the package for each, and the expected outputs chosen), **Apply** (every
  command run, since the definition of done needs real output), and **Validate** (results checked against
  the module's definition of done).

## How to build it (teach it to the next trainee)

A guide to installing and verifying software at both layers, for a Mac user working in the Ubuntu
container.

**Phase 1 — the system layer (as `root` in this container; prefix with `sudo` on a normal system):**

1. Refresh the package index, then install the JDK:
   ```
   apt update
   apt install -y default-jdk
   ```
2. Ask the tool which version is actually installed, and which binary answers:
   ```
   java -version
   which java
   ```
3. Ask the manager to confirm it recorded the package, and where it came from:
   ```
   apt list --installed | grep jdk
   apt policy default-jdk
   ```

**Phase 2 — the language layer:**

4. Make sure the tooling exists, then install inside an isolated environment — the recommended route:
   ```
   apt install -y python3-pip python3-venv
   python3 -m venv ~/linux-course/m6/venv
   source ~/linux-course/m6/venv/bin/activate
   pip install requests
   pip show requests
   ```
   No `sudo` is used here: the install targets the virtual environment, not the system.
   If a bare `pip install` is run against the system Python instead, Ubuntu refuses it with
   `externally-managed-environment`. The container used for this lab bypassed that refusal with
   `--break-system-packages`; the flag is a workaround, not the recommended path.

**Phase 3 — clean removal:**

5. Remove the package through the manager, then prove it is gone:
   ```
   pip uninstall requests
   pip show requests
   deactivate
   ```

## Concepts worth explaining

**System vs. language-level package management**

Two managers, two scopes, two records. The system manager installs operating-system software system-wide,
needs elevated privilege, and tracks everything in the distribution's database. The language manager
installs that language's libraries into an environment it controls, needs no privilege there, and keeps its
own metadata. Neither replaces the other: a normal machine has both, managing different things.

| | System manager (`apt`) | Language manager (`pip`) |
| --- | --- | --- |
| Manages | OS software, system libraries, services | language libraries for that ecosystem |
| Scope | system-wide (`/usr`, `/etc`) | a project or virtual environment |
| Tracks | the dpkg database | pip's own metadata (`pip show`, `pip list`) |
| Needs `sudo` | usually — it writes system paths | no, inside a virtual environment |
| Right for | a JDK, a compiler, a daemon | a project's Python dependencies |

**Verifying an install**

"Ran without error" answers a different question from "is it installed, at the version expected, from the
place expected". Two checks together answer the real one: a **version check** asks the tool itself what it
is (`java -version`, `pip --version`), and a **provenance check** asks the manager whether it recorded the
package and where from (`apt list --installed`, `apt policy`, `pip show`). `which java` supplies the third
fact — which binary is first on `PATH` — and that is what separates "installed" from "installed and actually
the one that runs".

## What tripped up the process

- **The system Python refused a bare install.** `pip install requests` against Ubuntu 26.04's Python
  returned `error: externally-managed-environment`. The lab worked around it with `--break-system-packages`,
  and `pip` replied with its own warning about running as `root` and conflicting with the system package
  manager. The guard was right; the flag silenced it.
- **The JDK install was a no-op.** `apt install default-jdk` reported `default-jdk is already the newest
  version (2:1.25-77)` — the package had arrived in an earlier session. An install log that says nothing
  happened is not evidence of anything, which is why the version had to come from `java -version`.
- **One JDK, two version numbers.** `apt` calls it `2:1.25-77`; `java -version` calls it `25.0.4.1`. Both
  are correct — package version and runtime build — but comparing them directly looks like a mismatch until
  that is understood.
- **`pip uninstall` needs a terminal.** The command asks `Proceed (Y/n)?`; run from a script or with no
  terminal attached, it raises `EOFError` instead of removing anything. Non-interactive use needs `-y`.
- **The prompts still pointed at the previous module.** The captured shell prompts read
  `~/linux-course/m5`, not `m6` — a leftover practice directory, harmless but confusing when the record is
  read later.
- **`apt list` warns about its own interface.** `WARNING: apt does not have a stable CLI interface` appears
  on every `apt list` call; it is noise, not a failure.

## Checkpoint evidence

**The JDK's own version (Lab step 2, DoD-01):**
```text
root@2c19b7676ba4:~/linux-course/m5# java -version
openjdk version "25.0.4.1" 2026-08-18
OpenJDK Runtime Environment (build 25.0.4.1+1-1-26.04.4-Ubuntu)
OpenJDK 64-Bit Server VM (build 25.0.4.1+1-1-26.04.4-Ubuntu, mixed mode, sharing)
```
> Reported by the JDK itself, not inferred from the install log. `apt` calls the same software `2:1.25-77`
> (DoD-01).

**The JDK tracked by the system manager (Lab step 3, DoD-02):**
```text
root@2c19b7676ba4:~/linux-course/m5# apt list --installed | grep jdk

WARNING: apt does not have a stable CLI interface. Use with caution in scripts.

default-jdk-headless/resolute,now 2:1.25-77 arm64 [installed,automatic]
default-jdk/resolute,now 2:1.25-77 arm64 [installed]
openjdk-25-jdk-headless/resolute-updates,resolute-security,now 25.0.4.1+1-1~26.04.4 arm64 [installed,automatic]
openjdk-25-jdk/resolute-updates,resolute-security,now 25.0.4.1+1-1~26.04.4 arm64 [installed,automatic]
openjdk-25-jre-headless/resolute-updates,resolute-security,now 25.0.4.1+1-1~26.04.4 arm64 [installed,automatic]
openjdk-25-jre/resolute-updates,resolute-security,now 25.0.4.1+1-1~26.04.4 arm64 [installed,automatic]
```
> `default-jdk` is listed as explicitly `[installed]`; the others are `[installed,automatic]`, pulled in as
> dependencies. The JDK is in the dpkg database, so `apt` owns its lifecycle (DoD-02).

**The Python package, verified separately from `apt` (Lab step 4, DoD-03):**
```text
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
> **Which manager installed what, and why.** `apt` installed the JDK — system software, shared by every
> user, recorded in dpkg. `pip` installed `requests` — a Python library, recorded in pip's own metadata at
> `/usr/local/lib/python3.14/dist-packages`, and invisible to `apt`. That split is correct for each layer:
> the JDK is an OS-level component with a system-wide lifecycle, while `requests` is a dependency of Python
> code and belongs to the Python environment. `Requires:` also shows the dependency resolution the language
> manager performed — four libraries arrived alongside the one that was asked for (DoD-03).

**Removed through the manager, not by hand (Lab step 5, DoD-04):**
```text
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
> `pip` found the package in its own record, listed exactly which files it would remove, and removed them
> itself (DoD-04). The warning underneath is pip's own verdict on the `--break-system-packages` route taken
> here.

**Supplementary verification run (same container, as `trainee` — not part of the trainee's capture):**

> These outputs come from a second run in container `2c19b7676ba4` under the standard `trainee` account,
> made to confirm the behaviour the lab record shows and to supply the provenance check the lab did not
> capture. Prompts are abbreviated for readability. They are kept separate from the trainee's own capture
> above and are not presented as it.

```text
trainee@2c19b7676ba4:~$ apt install -y default-jdk
Reading package lists...
Building dependency tree...
Reading state information...
default-jdk is already the newest version (2:1.25-77).
Solving dependencies...
Summary:
  Upgrading: 0, Installing: 0, Removing: 0, Not Upgrading: 3
```
> The no-op install, reproduced: the package was already present, so the command succeeded without
> installing anything. This is precisely the case where an install log cannot be trusted as evidence.

```text
trainee@2c19b7676ba4:~$ apt policy default-jdk
default-jdk:
  Installed: 2:1.25-77
  Candidate: 2:1.25-77
  Version table:
 *** 2:1.25-77 500
        500 http://ports.ubuntu.com/ubuntu-ports resolute/main arm64 Packages
        100 /var/lib/dpkg/status
```
> The provenance check: installed equals candidate, and the repository is named —
> `ports.ubuntu.com/ubuntu-ports … resolute/main arm64`. This answers *where it came from*, which
> `apt list --installed` alone does not.

```text
trainee@2c19b7676ba4:~$ pip install requests
error: externally-managed-environment

× This environment is externally managed
╰─> To install Python packages system-wide, try apt install
    python3-xyz, where xyz is the package you are trying to
    install.

    If you wish to install a non-Debian-packaged Python package,
    create a virtual environment using python3 -m venv path/to/venv.
    ...
hint: See PEP 668 for the detailed specification.
```
> The guard the lab's `--break-system-packages` flag bypassed, reproduced under a standard user — it is real
> and not a one-off. The error text itself names the correct fix: a virtual environment.

**A successful install that is still broken (Lab steps 2 and 4, DoD-05):**

The version checks make one thing clear: an install "succeeding" only means the manager wrote something
down. An install can complete silently and still be broken if any of the following is true:

- **A different copy answers.** `java -version` reports whichever `java` is first on `PATH`. An older JDK
  earlier in `PATH` would answer with its own version while `apt` still records `default-jdk` as installed —
  the package is fine, the command is wrong. `which java` and `command -v` expose this.
- **The environment the code runs in is not the environment that was verified.** `pip show requests` proves
  the package exists in the Python that ran `pip`. If the program runs under a different interpreter —
  another Python version, or an inactive virtual environment — the import fails even though `pip show`
  succeeds. Verification has to use the same interpreter the consumer uses.
- **A runtime dependency is missing or mismatched.** `requests` declares `certifi`, `charset_normalizer`,
  `idna` and `urllib3`. If one is absent or an incompatible version is present, installation succeeds and
  the failure appears later, as an `ImportError` or an SSL error.
- **The manager was bypassed.** Here, `--break-system-packages` let a project library into the
  distribution's Python. Both the install and `pip show` succeed while the package sits in a place the
  distribution did not intend — the conflict pip's warning describes, and one a later system upgrade can
  expose.
- **The installed version is not the version required.** The install can be clean while the software still
  fails, because the feature or API the code needs does not exist in that build.

The check that separates these cases from genuine success is the same in every row: ask the tool, ask the
manager, and do it in the environment that will actually run the code.

## What to do differently

If the module were started over, `requests` would go into a virtual environment from the start. The system
Python's PEP 668 guard is not an obstacle to route around — it is the correct answer to the question the
module is asking, and `python3 -m venv ~/linux-course/m6/venv` removes the need for
`--break-system-packages` entirely. The install logs would also be captured rather than only the
verification commands, so the record shows what the manager actually did, not only the state afterwards.
`apt policy default-jdk` would be run at install time to record the repository next to the version, the
verification would import `requests` from the same interpreter that will use it, and the practice directory
would be `~/linux-course/m6` — not the `m5` path the prompts in this capture still show.
