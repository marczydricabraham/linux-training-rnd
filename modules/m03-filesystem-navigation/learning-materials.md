# M3 Learning Materials — Filesystem Navigation

> **Purpose.** A self-contained study guide for the concepts and commands named in
> [`brief.md`](./brief.md): the Filesystem Hierarchy Standard, absolute and relative paths, building
> nested structures in one step, and searching instead of browsing. It teaches the material; it does
> **not** replace the lab. Build the structure in your own Linux environment and record what really
> happened in your own `write-up.md` (created from [`write-up-template.md`](./write-up-template.md)).
>
> **Audience.** Beginners who have completed M1 and M2 and can open a shell, run commands, and use pipes
> and redirection. No prior experience with directory trees, paths, or search tools is assumed.
>
> **Scope note.** All command output below is **illustrative teaching output**, not a record of a
> completed lab. Replace every example with your own real state before treating anything here as
> evidence.
>
> **How to use this guide.** Each section follows **Understand → Try → Verify → Continue**, and each
> answers the **"Research:"** question posed under the matching Scope bullet in the brief. The guided path
> takes roughly **65–100 minutes**. M3 removes the course's second failure mode — not being able to
> find the log, config, or data file.

## Contents

1. [The Filesystem Hierarchy Standard (FHS)](#1-the-filesystem-hierarchy-standard-fhs)
2. [Absolute vs. relative paths](#2-absolute-vs-relative-paths)
3. [Building nested structures in one step](#3-building-nested-structures-in-one-step)
4. [Searching instead of browsing](#4-searching-instead-of-browsing)
5. [Lab walkthrough](#5-lab-walkthrough)
6. [Best practices and troubleshooting](#6-best-practices-and-troubleshooting)
7. [References](#7-references)

---

## 1. The Filesystem Hierarchy Standard (FHS)

_Estimated: 15–20 minutes._

### Understand

Linux has **one root tree**, anchored at `/`, rather than Windows-style drive letters (`C:\`, `D:\`).
Every disk, partition, or network share is *mounted* somewhere inside that single tree.

The **Filesystem Hierarchy Standard (FHS)** is a shared **convention** for what lives where, so that
almost any distribution can be navigated the same way:

- **`/etc`** — system-wide **configuration** files (text you edit), e.g. `/etc/os-release`.
- **`/var`** — **variable data** that changes as the system runs: logs (`/var/log`), spool, caches.
- **`/home`** — **users' personal files**, one directory per user, e.g. `/home/alex` (reached as `~`).
- **`/usr`** — **installed programs** and their supporting files: binaries (`/usr/bin`), libraries,
  documentation.

> **Research answer — why is the FHS a convention rather than a kernel-enforced rule?**
> The kernel makes no judgments about names. To the kernel, `/etc` is just an ordinary directory under
> `/`; it could contain anything. The FHS is respected because **distributions and applications agree**
> to follow it. That agreement is enormously useful — tools and humans expect `/etc` to hold
> configuration — but nothing stops a program from putting configuration elsewhere. Compare this with a
> rule the kernel *does* enforce: a file's permissions, or the fact that only the kernel can talk to
> hardware.

### Try

```bash
ls /                 # the top of the single tree
ls /etc | head       # configuration files
ls /var/log | head   # log files
ls /home             # user home directories
ls /usr/bin | head   # installed programs
```

### Verify

- Every path begins at `/` and descends — there is no `C:`.
- `/etc`, `/var`, `/home`, and `/usr` exist as siblings under `/` and hold the kinds of files described
  above.

### Continue

Knowing *where* things live is the map. The next section is the compass: how a path names a location.

---

## 2. Absolute vs. relative paths

_Estimated: 15 minutes._

### Understand

A **path** names a location in the tree, in one of two ways:

- An **absolute path** starts at the root `/` and is unambiguous anywhere: `/var/log`.
- A **relative path** starts at the **current working directory** and depends on where you are now:
  `log` (inside `/var`) or `../log`.

Which one is in use is the difference between staying oriented and getting lost. Two special names help:

- **`.`** — the **current** directory.
- **`..`** — the **parent** directory (one level up). Chains work: `../..` is two levels up.
- **`~`** — expands to the **current user's home directory** (e.g. `/home/alex`); `~/x` is the same as
  `/home/alex/x`.

> **Research answer — what do `.` and `..` mean, and what does `~` expand to?**
> `.` is the directory you are currently in; `..` is its parent. They are real directory entries present
> in every directory. `~` is a **shell** shortcut (not a filesystem entry): the shell replaces it with the
> value of `$HOME` before running the command, so `cd ~` and `cd /home/alex` are equivalent.

### Try

```bash
pwd                 # the current location (absolute)
cd /var/log         # absolute: anchored at /
pwd                 # now /var/log
cd ..               # relative: up one level -> /var
pwd
cd ~                # back to the home directory
pwd
```

### Verify

- `pwd` always prints an **absolute** path — it is your anchor.
- `cd ..` moved up exactly one level from wherever you were, proving `..` is context-dependent.
- `cd ~` returned to `/home/<you>`.

### Continue

With paths understood, the next step is to create a whole tree at once instead of one level at a time.

---

## 3. Building nested structures in one step

_Estimated: 10–15 minutes._

### Understand

`mkdir` makes a directory, but it fails if the parent does not exist. Stepping through each level
(`mkdir a`, `cd a`, `mkdir b`, …) is slow and error-prone. The **`-p`** option changes that: it creates
any **missing parent directories** in the path automatically, then the target — in a single command.

> **Research answer — which option creates missing parent directories?**
> `-p` (from "parents"). `mkdir -p ~/linux-course/m3/data/raw` creates `linux-course`, `m3`, `data`, and
> `raw` as needed, in one step. `-p` is also **idempotent**: if the directory already exists, it does
> **not** error — handy in scripts.

### Try

```bash
mkdir -p ~/linux-course/m3/data/raw
mkdir -p ~/linux-course/m3/data/processed
ls -la ~/linux-course/m3
```

Illustrative output:

```text
drwxr-xr-x  3 alex alex 4096 . 
drwxr-xr-x  4 alex alex 4096 data
```

### Verify

- Both leaf directories exist, and the intermediate `data` directory was created even though it did not
  exist before.
- Running the same `mkdir -p` again produces no error (idempotence).

### Continue

A well-built tree is useful only if a file inside it can be found later — often without knowing exactly
where it is.

---

## 4. Searching instead of browsing

_Estimated: 15 minutes._

### Understand

Browsing means `cd`-ing through directories looking for a file by eye — fine for a small tree, hopeless
for a large one or for a file whose location is unknown. **Search tools** find files instead.

Two tools:

- **`find`** — walks the filesystem **live**, descending the tree and testing each entry against the
  criteria given. Always current, but slower on huge trees.
- **`locate`** — looks up a **pre-built index** of filenames, so it is fast, but it can be **stale**
  (it does not see files created since the index was last rebuilt) and may not be installed in a slim
  container.

> **Research answer — how does `find` differ from `locate`, and how do you match a name pattern?**
> `find` searches the **live** tree (current, thorough, slower); `locate` searches an **index** (fast, can
> be out of date and may be absent). Name matching in `find` uses **`-name`** with a **glob pattern**:
> `*` matches any run of characters, `?` matches one, and quoting the pattern (`-name "*.conf"`) stops the
> shell from expanding it first.

**Names vs. contents:** `find` locates files **by name or attribute** (size, time, type); `grep`
searches **inside** files. Use `find … | grep …` to search *names* by a richer pattern, and `grep pattern
file` to search *contents*.

### Try

```bash
find /etc -name "*.conf"
find /etc -name "*.conf" | wc -l          # count the matches
ls -l /etc | grep ".conf"                 # names via a pipe (M2 pattern)
```

### Verify

- `find /etc -name "*.conf"` lists only paths ending in `.conf`, each printed from `/`.
- `wc -l` gives a count — the number Lab step 4 asks you to record.
- The result came from **searching**, not from browsing directory by directory.

### Continue

The lab puts navigation, one-step creation, and search together. Capture the real output as you go.

---

## 5. Lab walkthrough

> **Reminder.** The output above is illustrative. Perform these steps on your own machine and capture
> your own raw output.

_Estimated: 15–20 minutes._

**Goal.** Move around the filesystem, build a nested directory tree in one step, and find files by search
instead of browsing.

1. From the home directory, confirm your location, move to `/var/log`, confirm again, and return:
   ```bash
   pwd
   cd /var/log
   pwd
   cd ~
   pwd
   ```
2. Create a nested practice tree in one step:
   ```bash
   mkdir -p ~/linux-course/m3/data/raw
   mkdir -p ~/linux-course/m3/data/processed
   ```
3. List the tree, including hidden files:
   ```bash
   ls -la ~/linux-course/m3
   ```
4. Search `/etc` for every `.conf` entry and record how many results are returned:
   ```bash
   find /etc -name "*.conf"
   find /etc -name "*.conf" | wc -l
   ```
5. Reference a sibling directory with a **relative** path:
   ```bash
   cd ~/linux-course/m3/data/raw
   ls ../processed
   ```

**Capture.** The `pwd` output before and after the move, the `ls -la` listing of the tree, the count of
`.conf` results, and the relative-path command with its output.

**Clean-up.** Nothing to clean up. The practice tree stays under `~/linux-course/`.

**Definition of done (self-check).**

- [ ] DoD-01 — pasted `pwd` output showing the location before and after `cd /var/log`, and that `cd ~`
      returned to home.
- [ ] DoD-02 — the tree under `~/linux-course/m3` is **at least three levels deep**, created with
      `mkdir -p` (not one level at a time); the `ls -la` output shows it.
- [ ] DoD-03 — the notes record how many results `find /etc -name "*.conf"` returned, from your own
      output, showing files were found by **search**, not by browsing.
- [ ] DoD-04 — `ls ../processed` ran from `~/linux-course/m3/data/raw` using a **relative** path, and you
      can state the **absolute** path of `processed` without running a command to check.

---

## 6. Best practices and troubleshooting

_Estimated: 5 minutes._

### Best practices

- Use **`pwd` liberally** while learning to navigate — it is the anchor that prevents getting lost.
- Prefer **`mkdir -p`** when creating nested paths; it is one step and safe to re-run.
- Keep practice files in the **home directory** (`~/linux-course/`), not in `/etc`, `/usr`, or other
  system directories.
- Use **`find`** to locate files by name or attribute, and **`grep`** to search file contents.

### Troubleshooting

| Symptom | Likely cause | Fix |
| --- | --- | --- |
| `pwd` output is not what you expected | You changed directory earlier, or followed a relative path. | Run `pwd` after every `cd` while practising. |
| `mkdir: cannot create directory …: No such file or directory` | The parent path was missing and `-p` was omitted. | Re-run with `mkdir -p`. |
| `find` returns far more than expected | The pattern matched system-wide; `*` is broad. | Narrow the pattern (e.g. `-name "*.conf"`) or scope the start path. |
| `locate: command not found` | Minimal container lacks `mlocate`/`plocate`, or the index is empty. | Use `find` instead; it needs no index. |
| `find` output shows stale results | Index-based tooling (not `find`). | Rebuild the index, or switch to `find` for live results. |
| `ls ../processed` lists the wrong thing | You were not starting from `data/raw`. | `cd ~/linux-course/m3/data/raw` first, then re-run. |
| Pattern behaves oddly | The shell expanded `*` before `find` saw it. | Quote the pattern: `-name "*.conf"`. |

---

## 7. References

- [Filesystem Hierarchy Standard (FHS 3.0)](https://refspecs.linuxfoundation.org/FHS_3.0/fhs/index.html) — the canonical directory-purpose document.
- [The Linux Filesystem Explained (Linux Foundation)](https://www.linuxfoundation.org/blog/blog/classic-sysadmin-the-linux-filesystem-explained) — a readable tour of `/etc`, `/var`, `/home`, `/usr`.
- [find(1) — Linux man page](https://man7.org/linux/man-pages/man1/find.1.html) — `-name`, patterns, and attributes.
- [locate(1) — Linux man page](https://man7.org/linux/man-pages/man1/locate.1.html) — the indexed search tool.
- [Bash Reference Manual — Tilde Expansion](https://www.gnu.org/software/bash/manual/html_node/Tilde-Expansion.html) — what `~` expands to.
- [Bash Reference Manual — Filename Expansion](https://www.gnu.org/software/bash/manual/html_node/Filename-Expansion.html) — glob patterns and quoting.
- [ExplainShell](https://explainshell.com/) — paste a command and see what each part does.
- [MIT Missing Semester — The Shell](https://missing.csail.mit.edu/2020/course-shell/) — paths, navigation, and pipelines for beginners.

For the module's deliverable, checkpoint set, and definition of done, see [`brief.md`](./brief.md);
for an ordered checklist, see [`tasks.md`](./tasks.md).
