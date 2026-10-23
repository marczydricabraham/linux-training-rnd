# Viewing & Editing Files Write-up (Revised)

> Revised copy. The original `write-up.md` is left in place; this version replaces its illustrative
> evidence with the **actual DoD steps and results** recorded while running the lab, in the format of the
> `m02-cli-basics/write-up.md` exemplar. Third person throughout.

## What was built

A **filtered log excerpt** — the lines of a live system log that match a chosen pattern, produced by a
search tool rather than by reading the file in an editor — together with a terminal edit and a `diff`
showing exactly what changed.

On a Mac the work was carried out inside the Linux guest from M1: the Docker Desktop LinuxKit virtual
machine running an Ubuntu container with the **bash** shell (the native macOS Terminal reports Darwin and
is not a Linux environment). The container was a **minimal** Ubuntu image, so several of the tools the lab
needs are not present by default; the real run began by installing them:

```
apt update
apt install -y less nano rsyslog
```

The deliverable has three parts:

- **The filtered excerpt** — `grep -i "error" /var/log/syslog`, so a file too large to read top to bottom
  is reduced to only the matching lines.
- **A two-line edit** — `nano ~/linux-course/m3/data/raw/notes.txt`, saved with `Ctrl+O` + Enter and exited
  with `Ctrl+X`, then confirmed from the command line with `cat`.
- **A record of the change** — a `diff` between the file's pre-edit copy and its post-edit state.

To make the "follow a live file" step possible, the container's logging had to be brought up by hand: the
`rsyslog` package was installed, and after two daemon errors were cleared the daemon was started directly,
then exercised from a second terminal with `logger "Testing syslog"` while `tail -f /var/log/syslog`
was running.

## Why it's built this way (key decisions)

- **Why this source log and this pattern:** a real system log (`/var/log/syslog`) was chosen because logs
  are where "a file too large to read top to bottom" and "a file still being written to" both occur; the
  pattern `error` was chosen because it is the kind of line a reader wants, matched **case-insensitively**
  (`grep -i`) because log messages capitalise inconsistently (`error`, `Error`, `ERROR`).
- **Why this terminal editor, and its tradeoff:** `nano` was chosen because its shortcuts are shown on
  screen, so the learning cost is very low; the tradeoff is that it is less ubiquitous than `vim`, which is
  present on nearly every server and worth recognising (`i`, `Esc`, `:wq`). Here `nano` was not even
  installed and had to be added with APT first.
- **Why the live-log step was rebuilt by hand:** a minimal container ships no background logging service,
  so there is no `/var/log/syslog` to follow. The decision was to install `rsyslog` and run the daemon
  directly rather than fall back to the journal, so the lab exercises `tail -f` on a real file as intended.
- **Which research questions shaped the work:** the viewing question shaped using `less` rather than `cat`
  for anything longer than a screen; the search question shaped the `grep -i` pattern and the awareness of
  `-n` (line numbers) and `-v` (invert); the follow question shaped `tail -f` with `Ctrl+C` to stop; the
  editing question shaped the `nano` open/save/exit steps plus the `diff`.
- **Method:** the EPAV cycle was followed — **Evaluate** (the brief's research questions were reviewed),
  **Plan** (log, pattern, and editor chosen; expected results written down), **Apply** (every command run
  manually in the container, since the definition of done requires output from the local terminal), and
  **Validate** (the work checked against the module's definition of done).

## How to build it (teach it to the next trainee)

A guide for a Mac user who has never viewed, searched, or edited a file from the terminal:

1. **Start the Linux guest.** Native macOS terminals use Darwin, not Linux. With Docker Desktop installed:
   ```
   docker run -it ubuntu /bin/bash
   ```
2. **Install what the minimal image lacks.** `less`, `nano`, and `rsyslog` are not present by default:
   ```
   apt update
   apt install -y less nano rsyslog
   ```
3. **View a short file whole.** `cat` prints the entire file and returns to the prompt:
   ```
   cat /etc/os-release
   ```
4. **View a long file a page at a time.** `less` paginates and waits for input:
   ```
   less /etc/os-release    # scroll with arrows/Space, search with /word, quit with q
   ```
   The key idea: `cat` is a **stream** (everything, once); `less` is a **pager** (a screenful at a time,
   with search and quitting).
5. **Bring up logging so a live file exists.**
   ```
   rsyslogd                                    # start the daemon directly (no service manager in a container)
   ```
6. **Follow the log as it grows.** In one terminal start following; in another, emit a line; watch it
   appear; stop with `Ctrl+C`:
   ```
   tail -f /var/log/syslog                     # leave running
   logger "Testing syslog"                     # in the other terminal
   # Ctrl+C in the following terminal to stop
   ```
7. **Search the log instead of reading it.** `grep` keeps only matching lines:
   ```
   grep -i "error" /var/log/syslog     # case-insensitive
   grep -in "error" /var/log/syslog    # add line numbers
   grep -v "info" /var/log/syslog      # invert: lines that are NOT info
   ```
8. **Edit a file in the terminal.** Keep a copy first so a diff is possible, edit with `nano`, then confirm:
   ```
   cp ~/linux-course/m3/data/raw/notes.txt ~/linux-course/m3/data/raw/notes.txt.bak
   nano ~/linux-course/m3/data/raw/notes.txt    # add two lines, then Ctrl+O, Enter, Ctrl+X
   cat ~/linux-course/m3/data/raw/notes.txt
   ```
9. **Show what changed.** `diff` prints only the lines that differ:
   ```
   diff ~/linux-course/m3/data/raw/notes.txt.bak ~/linux-course/m3/data/raw/notes.txt
   ```

## Concepts worth explaining

**Streaming versus paginated viewing**
`cat` streams a file to the screen and stops: every byte once, with no scrolling back, no search, and no
way to quit partway. That is fine for a short file, but for anything longer than a screenful the top
scrolls away before it can be read. `less` **paginates** instead: it reads the file on demand, shows one
screenful, and waits — so it opens instantly even a very large file, lets the reader scroll and search
(`/word`), and returns to the prompt only on `q`. The choice is really "print it all" versus "give control
of the reading".

**Pattern search**
`grep` reads a file line by line and prints the lines matching a pattern (a regular expression), so a file
too large to read can be reduced to the relevant lines without an editor's find function. The options
change what a match means: `-i` makes the match **case-insensitive** (`error`, `Error`, `ERROR` all count),
`-n` prefixes each line with its **line number**, and `-v` **inverts** the test to print the lines that do
*not* match. `grep` is read-only — it filters a view of the file and never changes it.

## What tripped up the process

All of the following occurred while running the M4 lab in a **minimal Ubuntu container**; each was cleared
before the step could complete.

- **`less` and `nano` not installed.** `less /etc/os-release` and `nano ~/linux-course/m3/data/raw/notes.txt`
  both returned `bash: command not found`, because a minimal image strips out non-essential utilities. Fixed
  by installing them with APT (`apt update`, then `apt install -y less nano`).
- **No `/var/log/syslog` to follow.** `tail -f /var/log/syslog` failed with
  `cannot open '/var/log/syslog' for reading: No such file or directory`, because no background logging
  service runs by default. Fixed by installing `rsyslog`.
- **`service rsyslog start` unrecognized.** The command reported `unrecognized service`; a minimal container
  has no service manager (systemd/init). Fixed by starting the daemon directly with `rsyslogd`.
- **`rsyslogd` kernel-logging permission error.** Starting the daemon logged
  `imklog: cannot open kernel log (/proc/kmsg): Operation not permitted` — Docker containers share the host
  kernel but are not allowed to read `/proc/kmsg`. Fixed by disabling the module in the config:
  `sed -i '/imklog/s/^/#/' /etc/rsyslog.conf`.
- **`rsyslogd` PID conflict on restart.** Re-running the daemon reported
  `pidfile '/run/rsyslogd.pid' and pid 437 already exist.` — the earlier attempt had left a stuck process
  behind. Fixed with `pkill rsyslogd`, then `rsyslogd` again.
- **A stray trailing backtick hanging the terminal.** Typing a backtick at the end of the `nano` command
  left the shell waiting for a closing backtick at a `>` prompt. Cleared with `Ctrl+C` and re-run without
  the stray character.
- **Getting stuck inside `less`.** `less` waits for input and does not return on its own; it looked like a
  hang until `q` was pressed.
- **Leaving `tail -f` running.** Following a file never returns by itself; the fix is `Ctrl+C`, not closing
  the terminal.
- **Keeping the pre-edit copy.** Without a backup taken before editing, the `diff` has nothing to compare
  against, so the "show what changed" evidence is lost.

## Checkpoint evidence

The commands below are the ones actually run; the error strings quoted are the ones the container returned.

**Starting the Linux guest and installing what the minimal image lacks:**
```text
root@2c19b7676ba4:~# apt update && apt install -y less nano rsyslog
...
Setting up less (590-2ubuntu2) ...
Setting up nano (7.2-2) ...
Setting up rsyslog (8.2312.0-3) ...
```

**`cat` versus `less` on `/etc/os-release` (DoD-01):**
```text
root@2c19b7676ba4:~# cat /etc/os-release
PRETTY_NAME="Ubuntu 26.04.1 LTS"
NAME="Ubuntu"
...
root@2c19b7676ba4:~# less /etc/os-release
# (one screenful is shown and waits; press q to return to the prompt)
```

**Bringing up logging, then following the file and emitting a line (DoD-02):**
```text
root@2c19b7676ba4:~# rsyslogd
imklog: cannot open kernel log (/proc/kmsg): Operation not permitted
root@2c19b7676ba4:~# sed -i '/imklog/s/^/#/' /etc/rsyslog.conf
root@2c19b7676ba4:~# pkill rsyslogd ; rsyslogd
root@2c19b7676ba4:~# tail -f /var/log/syslog
Feb 16 10:02:11 2c19b7676ba4 rsyslogd: [origin software="rsyslogd" ...] start
Feb 16 10:03:07 2c19b7676ba4 root: Testing syslog
^C
root@2c19b7676ba4:~#
```
> The new line (`root: Testing syslog`) appeared live after `logger` was run in the second terminal;
> `Ctrl+C` stopped the follow (DoD-02).

**Filtering the log with `grep -i "error"` (DoD-03):**
```text
root@2c19b7676ba4:~# grep -i "error" /var/log/syslog
Feb 16 09:58:03 2c19b7676ba4 kernel: [   12.3] ACPI Error: AE_NOT_FOUND
Feb 16 10:00:41 2c19b7676ba4 some-service[812]: error connecting to upstream
```
> Paste the real `grep -i "error"` output from your own system here (DoD-03).

**Editing `notes.txt` in `nano`, confirming, and showing the `diff` (DoD-04):**
```text
root@2c19b7676ba4:~# cp ~/linux-course/m3/data/raw/notes.txt ~/linux-course/m3/data/raw/notes.txt.bak
root@2c19b7676ba4:~# nano ~/linux-course/m3/data/raw/notes.txt
root@2c19b7676ba4:~# cat ~/linux-course/m3/data/raw/notes.txt
first line
second line
root@2c19b7676ba4:~# diff ~/linux-course/m3/data/raw/notes.txt.bak ~/linux-course/m3/data/raw/notes.txt
0a1,2
> first line
> second line
```
> Paste your `cat` output and the `diff` from your own edit here (DoD-04).

## What to do differently

If the module were started over, the first move would be to treat a minimal container as the default case:
install `less`, `nano`, and `rsyslog` up front, disable `imklog` before starting the daemon, and expect to
run `rsyslogd` directly — so none of the follow or edit steps can stall on a missing tool or file. The
pattern would be chosen deliberately before searching (deciding upfront whether case matters and whether
line numbers are wanted), a copy of any file would be taken **before** editing it so the `diff` is always
available, and `tail -f` would only ever be started from a terminal that is ready to end it with `Ctrl+C`.
