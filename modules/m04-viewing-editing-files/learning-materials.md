# M4 Learning Materials — Viewing & Editing Files

> **Purpose.** A self-contained study guide for the tools and ideas named in [`brief.md`](./brief.md):
> viewing files, searching their contents, following a file as it grows, and editing in the terminal. It
> teaches the material; it does **not** replace the lab. Work with a real log in your own Linux
> environment and record what really happened in your own `write-up.md` (created from
> [`write-up-template.md`](./write-up-template.md)).
>
> **Audience.** Beginners who have completed M1–M3 and can navigate the filesystem, compose commands, and
> use pipes and redirection. No prior experience with pagers, log following, or terminal editors is
> assumed.
>
> **Scope note.** All command output below is **illustrative teaching output**, not a record of a
> completed lab. Replace every example with your own real state before treating anything here as evidence.
>
> **How to use this guide.** Each section follows **Understand → Try → Verify → Continue**, and each
> answers the **"Research:"** question posed under the matching Scope bullet in the brief. The guided path
> takes roughly **70–100 minutes**. M4 is about inspecting and changing files **without a GUI** — the
> normal situation on a server.

## Contents

1. [Viewing a file](#1-viewing-a-file)
2. [Searching contents](#2-searching-contents)
3. [Following a live file](#3-following-a-live-file)
4. [Editing in the terminal](#4-editing-in-the-terminal)
5. [Lab walkthrough](#5-lab-walkthrough)
6. [Best practices and troubleshooting](#6-best-practices-and-troubleshooting)
7. [References](#7-references)

---

## 1. Viewing a file

_Estimated: 15–20 minutes._

### Understand

A small set of tools covers almost every day-to-day case. Which one fits depends on **how big the file
is** and **whether it is still changing**:

| Tool | What it does | Best when |
| --- | --- | --- |
| `cat` | Prints the whole file at once | The file is **short** (fits on one screen) |
| `less` | Shows one **page at a time**, scrollable, searchable | The file is **long** |
| `head` | Shows the **first** lines (default 10) | Only the top matters |
| `tail` | Shows the **last** lines (default 10) | Only the newest lines matter |
| `tail -f` | Shows the last lines and **follows** new ones as they arrive | The file is **still growing** (a live log) |

`cat` streams the file's bytes to the screen and stops; there is no paging, no search, and no way back
once the top scrolls off. `less` **paginates**: it reads on demand, so it opens even a huge file instantly,
lets you scroll, and supports in-page search.

> **Research answer — how do `cat`, `less`, `head`, `tail`, and `tail -f` differ, and when is each
> appropriate?**
> `cat` dumps the whole file (fine for short files, unusable for long ones). `less` shows one screenful at
> a time and pages through on demand (the default for anything longer than a screen). `head` and `tail`
> show just the start or the end (a fixed slice) — useful when only the top or the newest part matters.
> `tail -f` keeps the file open and prints new lines as they are appended, for watching a log while a
> process runs. Rule of thumb: **short → `cat`; long → `less`; top → `head`; bottom → `tail`; growing →
> `tail -f`.**

### Try

```bash
cat /etc/os-release       # whole file, printed and done
less /etc/os-release      # one page; scroll with arrows/Space, search with /word, quit with q
head -n 5 /etc/os-release # just the first five lines
tail -n 5 /etc/os-release # just the last five lines
```

### Verify

- `cat` printed everything immediately and returned to the prompt; there was no scrolling back.
- `less` filled the screen, waited for input, and returned to the prompt only after `q`.
- `head`/`tail` printed a fixed slice, not the whole file.

### Continue

When a file is too big to page through at all, the next step is to pull out just the lines that matter.

---

## 2. Searching contents

_Estimated: 15 minutes._

### Understand

`grep` prints the lines of a file that **match a pattern**, one line at a time. The pattern is a
**regular expression**, so a plain word is a pattern that matches any line containing it. This turns a
file too large to read into an answer: filter the lines instead of scanning them by eye.

Commonly needed options:

- **`-i`** — **case-insensitive** match (`error`, `Error`, and `ERROR` all match).
- **`-n`** — prefix each matching line with its **line number**.
- **`-v`** — **invert** the match (print lines that do **not** match).

> **Research answer — what does `grep` match, and what do its case, line-number, and invert options do?**
> `grep` matches **lines** whose text fits the given pattern (a regular expression), printing each matching
> line. `-i` makes matching **case-insensitive**; `-n` shows the **line number** of each match; `-v`
> **reverses** the sense of the test, printing lines that do *not* match. The line-based behaviour is the
> key point: `grep` never edits the file — it is a read-only filter.

**Names vs. contents:** `grep` searches **inside** files; `find` (M3) locates files **by name**. Reach for
`grep` to answer "which lines say X?", and `find` to answer "where is the file called X?".

### Try

```bash
grep -i "error" /var/log/syslog        # any line containing error/Error/ERROR
grep -in "error" /var/log/syslog       # same, with line numbers
grep -v "info" /var/log/syslog         # everything that is NOT an info line
ls -l /var/log | grep "syslog"         # filter a command's output (M2 pipe pattern)
```

### Verify

- Each printed line contains the pattern (for `-i`, in any capitalisation).
- With `-n`, every line is prefixed by its position in the file.
- With `-v`, none of the printed lines contain the pattern.

### Continue

`grep` answers questions about a file that already exists. The next section watches a file that is being
written **right now**.

---

## 3. Following a live file

_Estimated: 10–15 minutes._

### Understand

Logs are how a system's behaviour is observed, and they grow while being watched. `tail -f` prints the
last few lines of a file and then **keeps the file open**, printing each new line the moment it is
appended — so a running process's messages appear live.

Following is a foreground activity: it never returns to the prompt on its own, which is exactly the point
— it waits for new output. It is stopped by interrupting it.

> **Research answer — how do you follow a file as it grows, and how do you stop?**
> Run `tail -f <file>`: it shows the tail and continues to print lines as they are appended. It runs until
> interrupted; press **Ctrl+C** to stop following and return to the prompt. (Awareness only: `tail -F`
> re-opens the file if it is rotated or replaced, which matters for long-lived log watching.)

### Try

```bash
tail -f /var/log/syslog        # starts following; leave this terminal running
# ...in another terminal, generate activity, e.g.:
sudo apt update
# the new lines appear below in the tail -f output
# press Ctrl+C to stop
```

### Verify

- New lines appeared **after** `tail -f` started, without re-running the command — proof the file was
  followed, not just read once.
- Ctrl+C returned control to the prompt.

### Continue

Viewing, searching, and following are all **read-only**. The last theme changes the file on disk.

---

## 4. Editing in the terminal

_Estimated: 15–20 minutes._

### Understand

Configuration on a server is routinely edited with no GUI, so a **terminal editor** is essential. `nano`
is the friendliest to start with; `vim` is everywhere and worth recognising. Both read a file into a
buffer, let it be changed, and write the buffer back on save.

**`nano`**
- Open: `nano <file>`
- Type to edit; the bottom bar lists the shortcuts (`^` means Ctrl).
- Save: **Ctrl+O**, then **Enter** to confirm the name.
- Exit: **Ctrl+X**.

**`vim`** (awareness — the brief's best practices mention `:wq`)
- Open: `vim <file>`
- Enter insert mode with **`i`**; leave it with **`Esc`**.
- Save and quit: **`:wq`** (or `:x`); quit without saving: **`:q!`**.

> **Research answer — how do you open, change, save, and exit in the chosen editor, and how do you show
> what changed?**
> In `nano`: open with `nano <file>`, type the change, save with `Ctrl+O` + Enter, exit with `Ctrl+X`. In
> `vim`: open with `vim <file>`, press `i` to insert, `Esc` to stop inserting, then `:wq` to write and quit.
> To **show what changed**, keep a copy before editing and compare, e.g. `cp file file.bak` before the
> edit and `diff file.bak file` after — `diff` prints only the lines that differ. (In a Git repo the same
> evidence comes from `git diff`.)

### Try

```bash
cp /tmp/notes.txt /tmp/notes.txt.bak      # keep a copy first, so a diff is possible
nano ~/linux-course/m3/data/raw/notes.txt # add two lines, then Ctrl+O, Enter, Ctrl+X
cat ~/linux-course/m3/data/raw/notes.txt  # confirm from the command line
diff /tmp/notes.txt.bak /tmp/notes.txt    # show exactly what changed (if a backup was kept)
```

### Verify

- The file reopened with `cat` shows the added lines — the edit was written to disk, not just on screen.
- `diff` (or `git diff`) shows the added lines and nothing else.

### Continue

The lab puts viewing, following, searching, and editing together on a real log. Capture the real output.

---

## 5. Lab walkthrough

> **Reminder.** The output above is illustrative. Perform these steps on your own machine and capture
> your own raw output.

_Estimated: 20 minutes._

**Goal.** View, follow, search, and edit files entirely from the terminal.

1. View a system file in full, then a page at a time:
   ```bash
   cat /etc/os-release
   less /etc/os-release      # press q to exit
   ```
2. In a **separate terminal or tab**, follow the system log, trigger activity in the first terminal, and
   confirm the new line appears:
   ```bash
   tail -f /var/log/syslog   # terminal B
   sudo apt update           # terminal A
   # the new line appears in terminal B; stop it there with Ctrl+C
   ```
3. Search the same log for a case-insensitive pattern:
   ```bash
   grep -i "error" /var/log/syslog
   ```
4. Create and edit a file with `nano`, adding two lines, saving, and exiting:
   ```bash
   nano ~/linux-course/m3/data/raw/notes.txt   # then Ctrl+O, Enter, Ctrl+X
   ```
5. Confirm the file's contents from the command line:
   ```bash
   cat ~/linux-course/m3/data/raw/notes.txt
   ```

> **Path note.** Step 4 targets the tree built in M3 (`~/linux-course/m3/data/raw/`). If that tree does not
> exist, create it first with `mkdir -p ~/linux-course/m3/data/raw`, or substitute another writable path.

**Capture.** How `cat` and `less` differed, the new line that appeared in the `tail -f` output, the `grep`
result, and the contents of `notes.txt`.

**Clean-up.** Stop `tail -f` with **Ctrl+C**. The practice file stays under `~/linux-course/`.

**Definition of done (self-check).**

- [ ] DoD-01 — the notes describe how `cat` and `less` behaved **differently** on `/etc/os-release` and
      when each would be chosen.
- [ ] DoD-02 — the pasted `tail -f` output shows the new line produced by the trainee's own
      `sudo apt update`, proving a file was followed as it grew and then stopped.
- [ ] DoD-03 — the `grep -i "error" /var/log/syslog` output contains **only** matching lines, with no
      manual trimming.
- [ ] DoD-04 — `notes.txt` exists with the two lines written in `nano`, and the pasted `cat` output
      confirms it from the command line.

---

## 6. Best practices and troubleshooting

_Estimated: 5 minutes._

### Best practices

- Use **`less`** rather than `cat` for anything longer than a screenful.
- Use **`tail -f`** to watch a log during an active process (and `Ctrl+C` to stop).
- Remember **`Esc`** and **`:wq`** when first learning `vim`.
- Search logs with **`grep -i`**, since log messages are inconsistently capitalised.

### Troubleshooting

| Symptom | Likely cause | Fix |
| --- | --- | --- |
| `cat` floods the screen | The file is longer than a screenful. | Use `less` instead, or `head`/`tail` for a slice. |
| Stuck inside `less` | `less` waits for input. | Press `q` to quit; `/word` searches, `n` finds the next match. |
| `tail -f` never returns | That is correct — it follows until interrupted. | Press `Ctrl+C` to stop. |
| `No such file or directory: /var/log/syslog` | Some images use a different log path or the systemd journal. | Try `/var/log/messages` or `journalctl -f`; or generate a log file earlier. |
| `nano: command not found` (or `vim`) | A slim container lacks the editor. | Install it if allowed, or use the editor that is present (`vi` is usually available). |
| `grep` prints nothing | No line matches the pattern. | Widen the pattern, drop `-i`, or confirm the file has content. |
| Pattern behaves oddly | The shell expanded `*`/special characters before `grep` saw them. | Quote the pattern: `grep -i "error" file`. |

---

## 7. References

- [GNU grep manual](https://www.gnu.org/software/grep/manual/grep.html) — patterns and every option (`-i`, `-n`, `-v`).
- [less(1) — Linux man page](https://man7.org/linux/man-pages/man1/less.1.html) — paging, searching, and quitting.
- [tail(1) — Linux man page](https://man7.org/linux/man-pages/man1/tail.1.html) — `-f` and `-F` following.
- [cat(1) / head(1) — GNU Coreutils manual](https://www.gnu.org/software/coreutils/manual/coreutils.html) — whole-file and slice viewing.
- [nano — official documentation](https://www.nano-editor.org/dist/latest/nano.html) — shortcuts for open/save/exit.
- [Vim — official documentation](https://www.vim.org/docs.php) — `i`, `Esc`, and `:wq`.
- [diff(1) — Linux man page](https://man7.org/linux/man-pages/man1/diff.1.html) — showing what changed between two files.
- [MIT Missing Semester — Command-line Environment](https://missing.csail.mit.edu/2020/command-line/) — paging, searching, and editing from the terminal.

For the module's deliverable, checkpoint set, and definition of done, see [`brief.md`](./brief.md); for an
ordered checklist, see [`tasks.md`](./tasks.md).
