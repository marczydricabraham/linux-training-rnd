# M2 Learning Materials — CLI Basics

> **Purpose.** A self-contained study guide for the concepts named in [`brief.md`](./brief.md): command
> anatomy, pipes, redirection, and built-in help. It teaches the material; it does **not** replace the
> lab. Work the lab in your own Linux environment and record what really happened in your own
> `write-up.md` (created from [`write-up-template.md`](./write-up-template.md)).
>
> **Audience.** Beginners who have completed M1 and can open a shell. No prior experience with pipes,
> redirection, or manual pages is assumed.
>
> **Scope note.** All command output below is **illustrative teaching output**, not a record of a
> completed lab. Replace every example with your own real state before treating anything here as
> evidence.
>
> **Documentation vs. lab.** This guide is the *learning material* — a documentation deliverable. The
> **redirected file listing** (a file produced by piping and/or redirecting command output), the exact
> `cat` output of `practice.txt`, the piped `ls -l /etc | grep ".conf"` result, the thing `man`/`--help`
> revealed, and the Tab-completed path are a **separate lab deliverable that you build and capture
> yourself**. Nothing here is a ready-to-submit solution; see the checkpoint set in
> [§5](#5-lab-walkthrough).
>
> **How to use this guide.** Each section follows **Understand → Try → Verify → Continue**. The guided
> path takes roughly **60–90 minutes**. M2 is a small, foundational module; the goal is fluency in
> *composing* commands, not memorising flags.
>
> **Prerequisites.** Complete M1 first: a working Linux environment (this course uses **Docker Desktop**
> — a LinuxKit virtual machine running an **Ubuntu** container — accessed from the macOS terminal), a
> shell prompt you can type into, and the M1 habit of reading real output rather than paraphrasing it.
> No tool beyond the shell's own built-ins is required.

## Contents

1. [Command anatomy](#1-command-anatomy)
2. [Pipes (`|`)](#2-pipes-)
3. [Redirection (`>` vs `>>`)](#3-redirection--vs-)
4. [Built-in help: `man`, `--help`, and Tab](#4-built-in-help-man---help-and-tab)
5. [Lab walkthrough](#5-lab-walkthrough)
6. [Best practices and troubleshooting](#6-best-practices-and-troubleshooting)
7. [References](#7-references)

---

## 1. Command anatomy

_Estimated: 10–15 minutes._

### Understand

Nearly every command has the same three-part shape:

- the **command name** — the program to run, e.g. `ls`;
- **options** (also called *flags*) — tokens that change the command's behaviour, e.g. `-l`;
- **arguments** — the things the command acts on, e.g. `/etc`.

```text
ls        -l        /etc
name      option    argument
```

That shared shape is why one habit works across thousands of tools: learn to read the three parts and a
new command becomes guessable.

Options come in two forms:

- **Short flags** — a single dash and one letter, e.g. `-l`. Several can be combined: `ls -la` means
  `-l -a`.
- **Long flags** — a double dash and a word, e.g. `--all`. Long flags are self-describing; short flags
  are quicker to type.

An **argument** is anything the command operates on that is not an option — usually a file, directory,
or value. Whether a token is an option or an argument depends on the command, but the leading `-`/`--`
is the tell.

### Try

```bash
ls            # name only: list the current directory
ls -l         # add a short flag: long listing format
ls -l /etc    # add an argument: list that directory instead
ls --all      # a long flag: include hidden entries
```

### Verify

- Adding `-l` changes the *format*; adding `/etc` changes the *target*. The name stayed the same.
- `ls --all` and `ls -a` do the same thing with different spellings.

### Continue

Now that output exists in a predictable place, the next step is to send it somewhere other than the
screen — starting with pipes.

---

## 2. Pipes (`|`)

_Estimated: 10–15 minutes._

### Understand

A **pipe** (`|`) connects one command's output to the next command's input **without an intermediate
file**. The left command writes to standard output; the shell feeds that straight into the right
command's standard input.

Read a pipeline **left to right as a sequence of transformations**: each stage reshapes what the
previous stage produced.

```text
ls -l /etc    |    grep ".conf"
produce all    then  keep only
entries              matching lines
```

This is the Unix idea of small, single-purpose tools that combine into something larger: `grep` does
one job (filter lines), `ls` does one job (list entries), and `|` lets them cooperate.

### Try

```bash
ls -l /etc | grep ".conf"
ls -l /etc | grep ".conf" | wc -l    # add a third stage: count the matches
```

### Verify

- The output contains **only** lines matching `.conf` — the other `/etc` entries are gone.
- The pipeline reads as two (or three) steps, each transforming the previous result.

### Continue

Pipes move data between programs. When the destination is a **file** rather than another program,
redirection is used instead.

---

## 3. Redirection (`>` vs `>>`)

_Estimated: 10–15 minutes._

### Understand

Pipes connect programs; **redirection** sends a command's output to a file:

- `>` **overwrites** the file — its previous contents are discarded and replaced.
- `>>` **appends** to the file — previous contents are kept and the new output is added at the end.

The one-character difference decides whether earlier content survives. Choosing `>` by mistake silently
destroys what was already there; that is *what is lost when the wrong one is chosen*.

Neither operator is about the command — `echo`, `ls`, and `grep` all behave the same; redirection is the
shell's job, applied before the command runs.

### Try

```bash
mkdir -p ~/linux-course/m2
cd ~/linux-course/m2
echo "line one" >  practice.txt    # create the file (overwrite; fine when it is new/empty)
echo "line two" >> practice.txt    # add a second line (append; keeps "line one")
cat practice.txt
```

Illustrative output:

```text
line one
line two
```

### Verify

- If both lines had used `>`, the second command would have erased `line one`, leaving only `line two`.
- After the append, `cat` shows **both** lines — so the append preserved the first.

### Continue

When a flag's behaviour is uncertain, the next section shows how to look it up without leaving the
terminal.

---

## 4. Built-in help: `man`, `--help`, and Tab

_Estimated: 10–15 minutes._

### Understand

Three habits let you work without memorising every flag — which matters on a server with **no internet**
and no browser:

- **`man <command>`** — the full manual page: description, all options, examples. Navigate with the
  arrow keys / `Space` (page down), `q` to quit, and **`/` to search** (press `n` for the next match).
- **`<command> --help`** — the same command's short summary, printed straight to the screen; faster than
  the full manual.
- **Tab-completion** — pressing `Tab` asks the shell to finish the word. With **one** match it completes
  immediately; with **multiple** matches it lists the candidates (press `Tab` twice on some shells) so
  you can type more characters to disambiguate.

### Try

```bash
man grep          # opens the manual; search inside with /, quit with q
grep --help       # short summary, printed to stdout
cd ~/linux-cou    # type this much, then press Tab
```

Illustrative Tab behaviour: typing `cd ~/linux-cou` and pressing `Tab` completes to
`cd ~/linux-course/`, revealing the directory name without typing it all.

### Verify

- `man grep` opened a scrollable page that closed on `q`; a search for a term jumped to its explanation.
- `grep --help` produced a concise option list without leaving the prompt.
- Tab expanded a partial path to a full one — proof the shell is doing the completion, not the program.

### Continue

The lab below puts all four ideas together. Capture the real output as you go.

---

## 5. Lab walkthrough

> **Reminder.** The output above is illustrative. Perform these steps on your own machine and capture
> your own raw output.

_Estimated: 15–20 minutes._

**Goal.** Compose commands with redirection, pipes, and the built-in help, and keep evidence of what
each one did.

1. Create a working directory and a two-line file with redirection — the first line with `>`, the second
   with `>>`:
   ```bash
   mkdir -p ~/linux-course/m2
   cd ~/linux-course/m2
   echo "line one" >  practice.txt
   echo "line two" >> practice.txt
   ```
2. Confirm both lines are present:
   ```bash
   cat practice.txt
   ```
3. Pipe `ls -l` on `/etc` through `grep` to keep only `.conf` entries:
   ```bash
   ls -l /etc | grep ".conf"
   ```
4. Open the manual for `grep`, close it, then read the short help:
   ```bash
   man grep      # then press q to close
   grep --help
   ```
5. Practise Tab-completion: type `cd ~/linux-cou` and press `Tab`.

**Capture.** The contents of `practice.txt`, the output of the piped command, one thing that `man grep`
or `grep --help` revealed, and the path that Tab completed.

**Clean-up.** Nothing to clean up. The practice files stay under `~/linux-course/`.

**Definition of done (self-check).**

- [ ] DoD-01 — pasted `cat practice.txt` output showing **both** lines; re-running the documented
      commands reproduces the file.
- [ ] DoD-02 — the notes state which line used `>` and which used `>>`, and what `>` would have lost if
      used for both.
- [ ] DoD-03 — pasted `ls -l /etc | grep ".conf"` output showing only matches, read left to right as two
      steps (`ls -l` produces; `grep` keeps).
- [ ] DoD-04 — proof that both `man grep` and `grep --help` were opened, plus one thing one of them
      revealed you did not already know.
- [ ] DoD-05 — what Tab-completion did with the partial path `~/linux-cou`.

---

## 6. Best practices and troubleshooting

_Estimated: 5 minutes._

### Best practices

- Prefer `>>` over `>` when appending to a log or an existing file — `>` silently destroys earlier
  content.
- Read a pipeline **left to right** as a sequence of transformations, one small tool at a time.
- Check `man` or `--help` **before** guessing a flag's behaviour.

### Troubleshooting

| Symptom | Likely cause | Fix |
| --- | --- | --- |
| `practice.txt` only shows one line | Both commands used `>` (overwrite). | Re-run the second with `>>`, or rebuild from a fresh file. |
| `ls -l /etc \| grep ".conf"` prints nothing | `/etc` has no matching entries (rare on a minimal image). | Filter a directory you know contains matches, or widen the pattern. |
| `man: command not found` | A very slim container image lacks `man`. | Use `<command> --help`; optionally install `man-db` if the image allows. |
| `grep --help` scrolls past too fast | Long output on a small window. | Pipe it: `grep --help \| less`. |
| Tab does nothing | The partial word is ambiguous or misspelled. | Press `Tab` twice to list candidates, then type more characters. |
| A pipe "hangs" | The right-hand command is waiting for input it never gets. | Ensure the left command produces output; end with `q` if inside `less`. |

---

## 7. References

- [GNU Coreutils manual — `ls`, `echo`, `cat`](https://www.gnu.org/software/coreutils/manual/coreutils.html) — authoritative command behaviour.
- [GNU grep manual](https://www.gnu.org/software/grep/manual/grep.html) — patterns, options, and examples.
- [Bash Reference Manual — Redirections](https://www.gnu.org/software/bash/manual/html_node/Redirections.html) — how `>` and `>>` work in the shell.
- [Bash Reference Manual — Pipelines](https://www.gnu.org/software/bash/manual/html_node/Pipelines.html) — how `|` connects commands.
- [man page (Wikipedia)](https://en.wikipedia.org/wiki/Man_page) — the manual system and how to navigate it.
- [ExplainShell](https://explainshell.com/) — paste a command and see what each part does.
- [MIT Missing Semester — The Shell](https://missing.csail.mit.edu/2020/course-shell/) — a beginner-friendly treatment of pipes and redirection.

For the module's deliverable, checkpoint set, and definition of done, see [`brief.md`](./brief.md); for
an ordered checklist, see [`tasks.md`](./tasks.md).
