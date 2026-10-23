# M6 Learning Materials — Package Management

> **Purpose.** A self-contained study guide for the ideas named in [`brief.md`](./brief.md): how software
> actually arrives on a system, the difference between a **system** package manager and a **language-level**
> one, and how to *verify* an install instead of trusting it. It teaches the material; it does **not**
> replace the lab. Install real software on your own system and record what really happened in your own
> `write-up.md` (created from [`write-up-template.md`](./write-up-template.md)).
>
> **Audience.** Beginners who have completed M1–M5 and can navigate the filesystem, compose commands, view
> and edit files, and reason about `sudo` and permission bits. No prior experience with `apt`, `pip`, or
> virtual environments is assumed.
>
> **Scope note.** All command output below is **illustrative teaching output**, not a record of a completed
> lab. Replace every example with your own real state before treating anything here as evidence.
>
> **How to use this guide.** Each section follows **Understand → Try → Verify → Continue**, and each
> answers the **"Research:"** question posed under the matching Scope bullet in the brief. The guided path
> takes roughly **70–100 minutes**. M6 is about answering one question honestly — **"where did this software
> come from, and does it actually work?"** — instead of installing things until the errors stop.

## Contents

1. [System-level package management](#1-system-level-package-management)
2. [Language-level package management](#2-language-level-package-management)
3. [Verifying an install](#3-verifying-an-install)
4. [Lab walkthrough](#4-lab-walkthrough)
5. [Best practices and troubleshooting](#5-best-practices-and-troubleshooting)
6. [References](#6-references)

---

## 1. System-level package management

_Estimated: 20–25 minutes._

### Understand

Almost every piece of software on a Linux system arrives through a **package manager** rather than being
downloaded and copied by hand. On Ubuntu that manager is `apt` (acting on `.deb` packages via `dpkg`). It
brings three things that manual copying does not:

- **Tracking** — the system keeps a database of what is installed and at which version
  (`/var/lib/dpkg/status`, queried with `apt list --installed` or `dpkg -l`).
- **Dependency resolution** — a package declares what it needs; the manager fetches and installs those
  dependencies too, and refuses combinations that would conflict.
- **Clean removal** — `apt remove` reverses an install using that same database, rather than leaving stray
  files behind.

Packages are drawn from **repositories** — configured sources listed in `/etc/apt/sources.list` and
`/etc/apt/sources.list.d/`. A repository is just a location (often a URL) holding packages plus an **index**
of their names, versions, and dependencies. `apt update` downloads the current index (the *package list*);
`apt install` then reads that index. Because installing software writes **system-wide** — into `/usr`,
`/etc`, and the package database — it needs **elevated privilege**. That is exactly M5's permission model in
action: writing outside your home directory requires `root`, reached with `sudo`.

> **Research answer — what is a repository, what does dependency resolution do, and what does installing
> system-wide software require from M5's permission model?**
> A **repository** is a configured source of packages together with an index of them; `apt update` refreshes
> that index and `apt install` installs from it. **Dependency resolution** reads each package's declared
> requirements and installs (or verifies) everything it needs, and blocks an install that would conflict
> with what is already present — so you get a working set, not just one file. Installing **system-wide**
> means writing to locations owned by `root`, so it requires **elevated privilege**: `sudo apt install …`.
> An ordinary user cannot modify `/usr` or the package database, which is precisely the permission model
> from M5.

### Try

```bash
apt update                       # refresh the package index (needs sudo on a normal system)
apt list --installed | grep -i jdk    # what apt already tracks
apt policy default-jdk           # which version is available and from which repository
dpkg -l | grep jdk               # the same tracking, via the lower-level tool
```

> **Container note.** In a minimal image you may be `root` already, in which case `sudo` is optional. On a
> normal system, prefix the mutating commands with `sudo`.

### Verify

- `apt update` printed `Hit`/`Get` lines for each configured repository — proof the **index** was refreshed.
- `apt policy default-jdk` showed a **candidate version** and the **repository** it comes from — proof a
  package is only installable if a repository provides it.
- The package database (`apt list --installed`, `dpkg -l`) is what makes removal clean later.

### Continue

`apt` manages the **system**. Language ecosystems add a second, separate layer of package management on top.

---

## 2. Language-level package management

_Estimated: 20–25 minutes._

### Understand

Languages such as Python bring their **own** package managers — `pip` (and `conda`) for Python — for
**their own libraries**, independent of `apt`. The two layers differ in what they manage and in where they
install:

| | System manager (`apt`) | Language manager (`pip`) |
| --- | --- | --- |
| Manages | OS software, system libraries, services | language libraries for that ecosystem |
| Scope | system-wide (`/usr`, `/etc`) | a project or environment |
| Tracks | the dpkg database | the language's own metadata (`pip show`, `pip list`) |
| Needs `sudo` | usually (writes system paths) | **no**, inside a virtual environment |
| Right for | a JDK, a compiler, a daemon | a project's Python dependencies |

These layers can fight. Installing a library **globally** into the **system** Python mixes it with the
distribution's own Python packages, so a version you install can override (or break) a version something
else relies on. Modern Ubuntu goes further and marks the system Python as **externally managed** (PEP 668):
a bare `pip install` is *refused* with an `externally-managed-environment` error, precisely to stop you from
quietly damaging distro tooling. The correct answer is **isolation**, not overriding the guard.

> **Research answer — how does a language-level manager differ from the system manager, and why can a global
> install into the system's Python cause problems?**
> The language-level manager (`pip`) installs **the language's own libraries** into an *environment* it
> controls, independent of `apt`; it tracks its packages separately (`pip show`/`pip list`) and, inside a
> virtual environment, needs no `sudo`. The system manager (`apt`) installs **OS-level software** system-wide
> and is the one that needs elevated privilege. Installing a Python library **globally into the system
> Python** causes problems because it merges your package with the distribution's — version conflicts can
> appear, and upgrading or removing one side can break the other (including system tooling that depends on a
> specific version). That is why the system Python is marked externally managed, and why a **virtual
> environment** is used instead.

### Try

```bash
apt install -y python3-pip python3-venv          # ensure the tooling exists
python3 -m venv ~/linux-course/m6/venv           # an isolated environment
source ~/linux-course/m6/venv/bin/activate       # prompt shows (venv)
pip install requests                             # installs INTO the venv, no sudo
pip show requests                                # the language manager's own record
deactivate                                       # leave the environment
```

### Verify

- The prompt prefix `(venv)` confirmed the virtual environment was active.
- `pip install` needed **no** `sudo` — proof it wrote into the venv, not the system Python.
- `pip show requests` earlier/later is the language manager's record, **separate** from `apt list`.

### Continue

The package is installed — but "it installed" is not the same as "it works". The next section proves it.

---

## 3. Verifying an install

_Estimated: 15–20 minutes._

### Understand

A command that runs **without error** is not proof that an install succeeded, and the version you got may
not be the one you expected. Two kinds of check together are what actually prove an install:

- **A version check** — ask the tool itself what version it is: `java -version`, `python3 --version`,
  `pip --version`. The tool that runs is the tool that answers.
- **A provenance check** — ask the *manager* to confirm the package is recorded: `apt list --installed`,
  `apt policy <pkg>`, `dpkg -l <pkg>`, `pip show <pkg>`. This tells you which manager installed it and from
  where — the "where did it come from" question.

`which <command>` (or `command -v`) shows **which** binary will run, which matters when two versions are
installed and only one is on `PATH`. Together these answer both halves of verification: *what* is installed,
and *where it came from*.

> **Research answer — how do you ask the command line which version of each component is installed and
> where it came from?**
> Ask the tool for its own version (`java -version`, `pip --version`), and ask the **manager** for its
> record of the package (`apt list --installed`, `dpkg -l`, `apt policy`, `pip show`, `pip list`). Use
> `which <cmd>` / `command -v` to see which binary is actually on `PATH`. The version check proves the
> software runs and at what version; the provenance check proves which package manager installed it and from
> which repository — so you know which layer it belongs to.

### Try

```bash
java -version                    # the JDK's own version
which java                       # which binary is on PATH
apt list --installed | grep jdk  # apt's record of the JDK
pip show requests                # pip's record of the library
pip list | grep requests         # the same, as a list
```

### Verify

- `java -version` printed a real version string — **not** assumed from the install log.
- `apt list --installed` contains the JDK — the system manager tracks it.
- `pip show requests` shows the library and its version — the language manager tracks it, separately.

### Continue

The lab now installs a JDK and a Python package through the correct manager for each, verifies both, and
removes the Python package cleanly.

---

## 4. Lab walkthrough

> **Reminder.** The output above is illustrative. Perform these steps on your own machine and capture your
> own raw output.

_Estimated: 25–30 minutes._

**Goal.** Install a JDK and a Python package through the correct package manager for each, verify both from
the command line, and remove the Python package cleanly.

1. Update the local package list, then install a JDK (**system** manager):
   ```bash
   sudo apt update
   sudo apt install default-jdk      # large download; answer Y when prompted
   ```
2. Verify the install by checking the installed version:
   ```bash
   java -version
   ```
3. Confirm the package is tracked by the package manager:
   ```bash
   apt list --installed | grep jdk
   ```
4. Install a Python package with the **language-level** manager and verify it separately:
   ```bash
   apt install -y python3-pip python3-venv
   python3 -m venv ~/linux-course/m6/venv
   source ~/linux-course/m6/venv/bin/activate
   pip install requests
   pip show requests
   ```
5. Remove the Python package cleanly:
   ```bash
   pip uninstall requests
   deactivate
   ```

> **Why the two layers.** The JDK is *system software*, so it comes from `apt`; `requests` is a *Python
> library*, so it comes from `pip` — and it is installed into a **virtual environment**, not the system
> Python, to avoid conflicts with distribution-managed packages. That split is the whole point of the stage.

**Capture.** The `java -version` output, the `apt list` match, the `pip show requests` output, and the
`pip uninstall requests` output.

**Clean-up.** The `pip uninstall` in step 5 removes the Python package; `deactivate` leaves the virtual
environment. (The JDK stays installed — it is meant to remain.)

**Definition of done (self-check).**

- [ ] DoD-01 — the pasted `java -version` output shows the **actual** JDK version `apt` installed, taken
      from the command line and not assumed from the install log (Lab steps 1–2).
- [ ] DoD-02 — the pasted `apt list --installed | grep jdk` output shows the JDK is **tracked** by the
      system package manager (Lab step 3).
- [ ] DoD-03 — the pasted `pip show requests` output shows the Python package and its version, verified
      **separately** from `apt`, and the notes state which manager installed which piece and why that was
      right for that layer (Lab steps 1 and 4).
- [ ] DoD-04 — the pasted `pip uninstall requests` output shows the package was removed **through the
      package manager**, not by deleting files (Lab step 5).
- [ ] DoD-05 — using the version checks from steps 2 and 4, an explanation is written of what would need to
      be true for an install to **succeed silently but still be broken**.

---

## 5. Best practices and troubleshooting

_Estimated: 5 minutes._

### Best practices

- Run **`sudo apt update`** before **`sudo apt install`**.
- Use the **system package manager** for system-wide software, and **`pip`/`conda` in a virtual
  environment** for project-specific Python libraries.
- Verify an install with a **version check**, not by the absence of an error.
- Remove packages with the **package manager**, never by deleting files by hand.

### Troubleshooting

| Symptom | Likely cause | Fix |
| --- | --- | --- |
| `E: Unable to locate package` | The package list is stale, or the name is wrong. | Run `apt update` first, then check the exact name with `apt search`/`apt policy`. |
| `Permission denied` running `apt install` | The package manager writes system-wide. | Prefix with `sudo` (M5's model: it needs `root`). |
| `pip: command not found` | `pip` is not installed in this image or environment. | `apt install -y python3-pip`, or use a venv (`python3 -m venv`) which provides `pip`. |
| `error: externally-managed-environment` (PEP 668) | A bare `pip install` targets the system Python, which Ubuntu protects. | Create and activate a **virtual environment**, then install there — do not disable the guard. |
| `java: command not found` after install | The JDK was not installed, or its bin directory is not on `PATH`. | Confirm with `apt list --installed \| grep jdk`; check `which java` and `update-alternatives --list java`. |
| Wrong version shows up | Multiple versions installed; a different one is first on `PATH`. | Check `which java` / `command -v`, and `apt policy <pkg>` for what is available. |
| Install "succeeded" but the tool misbehaves | The install step finished but the component is missing or mismatched. | Verify with a version check (`java -version`, `pip show`), not the install log. |
| `apt install` filling the disk | A large package (e.g. a JDK) needs headroom. | Check with `df -h`; install only what is needed. |

---

## 6. References

- [apt(8) — Debian man page](https://manpages.debian.org/bookworm/apt/apt.8.en.html) — `update`, `install`, `remove`, `policy`.
- [dpkg(1) — Debian man page](https://manpages.debian.org/bookworm/dpkg/dpkg.1.en.html) — the lower-level package database and `-l`.
- [pip — official documentation](https://pip.pypa.io/en/stable/) — installing, `pip show`, `pip list`, `pip uninstall`.
- [venv — Creation of virtual environments (Python docs)](https://docs.python.org/3/library/venv.html) — isolating Python packages.
- [PEP 668 — Externally managed environments](https://peps.python.org/pep-0668/) — why a bare `pip install` is refused.
- [java(1) — Java launcher](https://docs.oracle.com/en/java/javase/21/docs/specs/man/java.html) — `-version`.
- [apt sources.list(5) — Debian man page](https://manpages.debian.org/bookworm/apt/sources.list.5.en.html) — how repositories are configured.
- [MIT Missing Semester — Package Management](https://missing.csail.mit.edu/2020/command-line/) — software, dependencies, and environments from the shell.

For the module's deliverable, checkpoint set, and definition of done, see [`brief.md`](./brief.md); for an
ordered checklist, see [`tasks.md`](./tasks.md).
