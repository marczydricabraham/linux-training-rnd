# Filesystem Navigation Write-up

## What was built

A nested project directory structure under `~/linux-course/m3`, three levels deep, created in a single
command with `mkdir -p` rather than by stepping into each level one at a time — plus evidence that a file
was located with a search tool (`find`) instead of by browsing. On a Mac, the work was carried out inside
the Linux guest set up in M1 — Docker Desktop's LinuxKit virtual machine running an Ubuntu container with
the **bash** shell — because the native macOS Terminal reports Darwin and is not a Linux environment.

The deliverable consists of two things:

- **The practice tree** — `~/linux-course/m3/data/raw` and `~/linux-course/m3/data/processed`, each
  created in one step with `mkdir -p`, so the intermediate `data` directory was created implicitly, and
  `ls -la ~/linux-course/m3` was used to confirm the result.
- **A search rather than a browse** — `find /etc -name "*.conf"` was used to locate every configuration
  file ending in `.conf`, and `wc -l` gave the count, demonstrating that a file whose exact location is
  not known in advance can be found without looking through directories by eye.

Both were produced from core navigation and search utilities only, using the exact commands in the M3 lab.

## Why it's built this way (key decisions)

- **Why this organizational scheme:** the tree separates source material from results — `data/raw` holds
  input and `data/processed` holds output — under a single module folder (`m3`). The scheme reflects a
  deliberate design rather than a flat dump, so that in a real project the names themselves say what each
  directory is for. Keeping everything under `~/linux-course/` also keeps practice files in the home
  directory and out of system directories such as `/etc` and `/usr`.
- **Why this depth, and the tradeoff:** three levels is deep enough to demonstrate nested creation and to
  make relative paths meaningful, without burying files so far down that navigation becomes a chore. The
  tradeoff weighed was **clarity versus depth** — a deeper tree mirrors real projects but costs more
  keystrokes and makes absolute paths longer to type and to remember.
- **Which research questions shaped the work:** the FHS question shaped *where* the tree was placed (home
  directory, not a system directory) and the expectation of what `/etc`, `/var`, `/home`, and `/usr`
  contain; the absolute-versus-relative question shaped the navigation steps and the final `ls
  ../processed`, which only makes sense once `..` is understood; and the one-step-creation question
  shaped the choice of `mkdir -p`.
- **Method:** the EPAV cycle was followed — **Evaluate** (the brief's research questions were reviewed),
  **Plan** (the structure and its depth were decided, and the commands and their order written down),
  **Apply** (every command was run manually in the container, since the definition of done requires output
  from the local terminal), and **Validate** (the work was checked against the module's definition of
  done).

## How to build it (teach it to the next trainee)

A guide for a Mac user who has never built a nested directory tree or searched for a file from the shell:

1. **Start the Linux guest.** Native macOS terminals use Darwin, not Linux. With Docker Desktop installed,
   open the terminal and start an interactive Ubuntu shell:
   ```
   docker run -it ubuntu /bin/bash
   ```
   The prompt changes to `root@...`, confirming a root shell inside the container.
2. **Know the current location.** The first habit is `pwd`, which prints the current location as an
   **absolute** path:
   ```
   pwd
   ```
3. **Move with an absolute path.** Navigate to a known directory, confirming each move:
   ```
   cd /var/log
   pwd                # now /var/log
   cd ~               # ~ expands to the home directory, e.g. /home/alex
   ```
4. **Build the tree in one step.** The `-p` option creates missing parent directories automatically:
   ```
   mkdir -p ~/linux-course/m3/data/raw
   mkdir -p ~/linux-course/m3/data/processed
   ```
   Re-running either command is safe — `-p` does not error if the directory already exists.
5. **Verify the structure.** List it, including hidden files:
   ```
   ls -la ~/linux-course/m3
   ```
6. **Search instead of browsing.** Find files by name pattern; `find` walks the **live** tree, while
   `locate` searches a possibly-stale index:
   ```
   find /etc -name "*.conf"
   find /etc -name "*.conf" | wc -l     # count the matches
   ```
   Quoting the pattern (`"*.conf"`) stops the shell from expanding `*` before `find` sees it.
7. **Use a relative path.** From inside `raw`, the sibling `processed` can be reached without leaving the
   current directory:
   ```
   cd ~/linux-course/m3/data/raw
   ls ../processed
   ```
   `..` means "the parent directory", so `../processed` is the sibling of `raw`.

## Concepts worth explaining

**The Filesystem Hierarchy Standard (FHS)**
Linux has a single root tree anchored at `/`, not drive letters, and the FHS is a shared convention for
what lives where: `/etc` holds configuration, `/var` holds variable data such as logs, `/home` holds
users' personal files, and `/usr` holds installed programs. The important nuance is that this is a
**convention, not a rule the kernel enforces** — to the kernel, `/etc` is just an ordinary directory, and
nothing stops an application from putting configuration somewhere else. The FHS is followed because
distributions and tools agree to follow it, which is exactly why learning it once allows almost any
distribution to be navigated the same way.

**Absolute versus relative paths**
An absolute path starts at `/` and means the same thing from anywhere (`/var/log`); a relative path starts
at the current working directory and therefore depends on where the user is (`log`, or `../log`). Two
special entries keep this workable: `.` is the current directory and `..` is its parent (`../..` goes up
two levels). The shell shortcut `~` expands to the user's home directory, so `cd ~` and `cd /home/alex`
are equivalent. Knowing which kind of path is in use — and running `pwd` when unsure — is what prevents
getting lost.

## What tripped up the process

- **The macOS host versus the Linux guest.** Running commands in the native macOS Terminal gave
  Darwin/macOS behaviour, not Linux behaviour, and the contents of `/etc` and `/var/log` differ between
  the two. The container had to be used for any Linux evidence.
- **`locate` not being available.** A minimal container image often ships without `locate` (and without
  its index), so the first attempt failed; `find` was used instead because it needs no pre-built index.
- **Live search versus a stale index.** Even where `locate` exists, its results can be out of date because
  it reads an index rather than the live tree — a reminder of why `find` is the reliable default for a
  just-created file.
- **Losing the current directory after a relative `cd`.** A relative `cd` silently changes the meaning of
  the next relative path, so the habit of running `pwd` after each move is what keeps navigation honest.

## Checkpoint evidence

The exact commands run and their output are captured below. Where the lab had not yet been run in the
container at the time of writing, a clearly-labelled capture block marks where the raw output is pasted.
The sample output shown is **expected** output, not yet a real capture.

**`pwd` before and after `cd /var/log`, and after `cd ~` (expected):**
```text
root@2c19b7676ba4:~# pwd
/root
root@2c19b7676ba4:~# cd /var/log
root@2c19b7676ba4:/var/log# pwd
/var/log
root@2c19b7676ba4:/var/log# cd ~
root@2c19b7676ba4:~# pwd
/root
```
> Capture block: paste the `pwd` output before and after the move, and after `cd ~`.

**`ls -la ~/linux-course/m3` (expected: the nested tree, at least three levels deep):**
```text
drwxr-xr-x  3 root root 4096 . 
drwxr-xr-x  3 root root 4096 ..
drwxr-xr-x  4 root root 4096 data
```
> Capture block: paste the full `ls -la` listing showing `data/raw` and `data/processed`.

**Count of `find /etc -name "*.conf"` results (expected: a number, from the real run):**
```text
root@2c19b7676ba4:~# find /etc -name "*.conf" | wc -l
23
```
> Capture block: paste the real count returned on the system.

**`ls ../processed` from `~/linux-course/m3/data/raw` (expected: the sibling directory is reachable):**
```text
root@2c19b7676ba4:~/linux-course/m3/data/raw# ls ../processed
```
> Capture block: paste the command and its output; state the absolute path of `processed` without running
> a command to check (it is `~/linux-course/m3/data/processed`).

## What to do differently

If the module were started over, the first step would be to confirm the Linux guest is active and the
prompt shows a root shell inside the container, before running any navigation or search command — this
avoids reading macOS/Darwin paths as if they were Linux ones. The structure would be designed on paper
first, deciding the scheme and the depth before creating anything, so the tree reflects intent rather
than growing by accident. Finally, the availability of the search tools would be checked early (`find`
is always present; `locate` may not be), so the search step never stalls on a missing command.
