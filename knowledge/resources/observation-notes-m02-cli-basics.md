# 📘 Research Notes: M2 — CLI Basics

> **Audience:** these notes are written for a trainee on a **Mac**. A Mac runs macOS (its kernel is
> **Darwin**), not Linux, so every command below is run inside the Linux guest set up in M1 — a
> **Docker Desktop** LinuxKit virtual machine running an **Ubuntu** container, whose shell is **bash**.
> The native macOS Terminal is not used for any Linux evidence, and the WSL2 path does not apply on a
> Mac. Everything here uses only the shell's own built-ins.

## 1. Objective

**What we are trying to achieve:**
Learn to compose commands together — joining them with pipes and redirecting their output — instead of running them one at a time, so the output can be produced and reshaped exactly as intended. The goal is a **redirected file listing**: a file whose contents were controlled deliberately through redirection and a pipe.

---

## 2. Core Concepts & Beginner Glossary

*A quick beginner's guide to the foundational terminology needed for this research. Each term is defined in plain language the first time it is used.*

- **Command:** a program the shell runs when its name is typed, for example `ls` or `grep`.
- **Option (flag):** a modifier that changes how a command behaves, for example `-l` in `ls -l`.
- **Short flag:** an option that starts with a single dash and a single letter, for example `-l` or `-a`. Short flags can usually be grouped, so `-l -a` becomes `-la`.
- **Long flag:** an option that starts with a double dash and a full word, for example `--all` or `--human-readable`. Long flags cannot be grouped, but they are easier to read.
- **Argument:** the thing the command acts on — a file path, a directory, or a piece of text, for example `/etc` in `ls -l /etc`.
- **Standard output (stdout):** the stream where a command normally writes its results, which by default appears on the screen.
- **Standard input (stdin):** the stream a command reads from, which by default is the keyboard.
- **Pipe (`|`):** a connector that takes the standard output of the command on its left and feeds it as standard input to the command on its right, with no temporary file in between.
- **Redirection:** sending a command's output into a file instead of the screen, using `>` or `>>`.
- **Overwrite (`>`):** redirection that creates the file, or erases an existing file completely, before writing the new output.
- **Append (`>>`):** redirection that creates the file if missing, but adds new output to the end of an existing file, keeping everything already there.
- **Man page:** the official reference manual for a command, opened with `man`.
- **`--help`:** a flag most commands support that prints a short, on-screen summary of usage and common flags.
- **Tab-completion:** pressing the `Tab` key to let the shell finish a partially typed command, file name, or path.

---

## 3. Scope Research & Answers

### Theme 1 — Command anatomy

- **The Question/Topic:** Nearly every command has the same shape (a name, options that change its behaviour, and arguments it acts on). What is the difference between a short flag and a long flag, and what is an argument?

- **The Detailed Explanation:**

  - Nearly every command follows one shared pattern: **command name + options (flags) + arguments**. Because the shape is shared, one habit — read left to right, recognise the name, the flags, then the argument — works across thousands of different tools.
    - **Command name:** the program to run, for example `ls`, `grep`, or `cd`.
    - **Options (flags):** modifiers that change how the command behaves.
    - **Arguments:** the target the command acts on.

  - **Short flag versus long flag:**
    - A **short flag** starts with a single dash (`-`) and a single letter, for example `-l` (long format) or `-h` (human-readable sizes). Short flags can usually be **grouped** into one token, so `-l -h` is the same as `-lh`.
    - A **long flag** starts with a double dash (`--`) and a descriptive word, for example `--all` or `--human-readable`. Long flags are **not** grouped, but they spell out their meaning, which makes them clearer in scripts and easier to remember.

  - **What an argument is:** an argument is the thing the command acts upon — a file path, a directory, or a piece of text. In `ls -l /etc`, the argument is `/etc` (the directory to list). In `grep ".conf"`, the argument is the text pattern to search for.

  - **Worked example** (from the module reference):
    ```bash
    ls -lh --color /var/log
    ```
    - `ls` — the command (list directory contents).
    - `-lh` — two grouped short flags: `-l` (long format) and `-h` (human-readable sizes).
    - `--color` — a long flag that colourises the output.
    - `/var/log` — the argument: the directory to list.

### Theme 2 — Pipes

- **The Question/Topic:** One command's output can become another's input without an intermediate file, so small single-purpose tools combine into something larger. What does `|` do, and how is a pipeline read left to right as a sequence of transformations?

- **The Detailed Explanation:**

  - The **pipe symbol `|`** takes the **standard output** of the command on its left and immediately feeds it as **standard input** to the command on its right. No temporary file is created — the data flows directly from one program to the next.

  - A pipeline is read and executed **left to right**, as a sequence of transformations: the first command produces a stream, the second reshapes it, the third reshapes what the second produced, and so on. Reading it in that direction is what makes a pipeline understandable: each `|` is one hand-off.

  - **Worked example** (from the module reference):
    ```bash
    cat server.log | grep "ERROR" | wc -l
    ```
    - `cat server.log` — reads the whole file and outputs it.
    - The first `|` hands that text to `grep "ERROR"`, which keeps only the lines containing `ERROR`.
    - The second `|` hands those kept lines to `wc -l`, which counts the lines and outputs the total number of error events.

  - **Read left to right:** produce (cat) → filter (grep) → count (wc). Each stage does one small job, and together they answer a question ("how many errors?") that no single one of them answers alone.

### Theme 3 — Redirection

- **The Question/Topic:** Output can go to a file instead of the screen, and whether it overwrites or appends decides whether earlier content is kept. What do `>` and `>>` each do, and what is lost when the wrong one is chosen?

- **The Detailed Explanation:**

  - **`>` (overwrite):** sends a command's output into a file. If the file does not exist, it is created. If it **does** exist, its existing contents are **erased completely** before the new output is written. Using `>` on a file that already holds data destroys that data permanently — there is no undo at the shell prompt.

  - **`>>` (append):** also sends output into a file and creates it if missing, but if the file already exists, the new output is added to the **very bottom**, leaving all earlier content untouched.

  - **What is lost with the wrong choice:** if `>>` was intended but `>` is used, everything the file already contained is gone — the command silently replaces the whole file instead of adding to it. This is the classic way to lose a log or a notes file.

  - **Worked example** (from the module reference):
    ```bash
    # Overwrites the file with the current date and time
    date > system_status.txt

    # Appends the current disk usage to the bottom of the same file
    df -h >> system_status.txt
    ```
    The first command replaces the file with the date. The second adds the disk usage beneath it, so the file ends up holding both.

  - **Rule of thumb:** use `>` when a fresh file is genuinely wanted; use `>>` when adding to a log or to any file that already has content.

### Theme 4 — Built-in help

- **The Question/Topic:** `man`, `--help`, and tab-completion allow work without memorising every flag, which matters on a server with no internet. How is a man page navigated and searched, and what does Tab do when more than one match exists?

- **The Detailed Explanation:**

  - **`man` (manual pages):** opens the official, comprehensive manual for a command. Inside a man page:
    - Scroll with the up and down arrow keys (and Page Up / Page Down).
    - Search by typing `/` followed by a word, then Enter, for example `/ignore-case`. Press `n` to jump to the next match.
    - Quit with `q`.
    This matters on a server without internet: the manual ships with the system, so help is always at hand.

  - **`--help` flag:** most commands accept `--help`, which prints a short summary of usage and common flags straight to the screen, without opening the full manual. It is faster when only a quick reminder is needed.

  - **Tab-completion:** pressing `Tab` while typing asks the shell to finish the word.
    - If there is **exactly one** possible completion, the shell completes it immediately.
    - If there is **more than one** match, a first `Tab` usually completes as far as the matches agree and may do nothing visible; a **second `Tab`** prints the list of all possible matches, so the available options can be seen and the typing continued with confidence.
    - **Shell note:** completion behaviour is provided by the shell. Inside the Ubuntu container the shell is **bash**, whose tab-completion differs in small ways from the `zsh` used by the macOS host's Terminal. The behaviour described here is the container's bash behaviour.

  - **Worked example** (from the module reference):
    ```bash
    # To find how to make 'grep' ignore case:
    man grep
    # Inside the manual, typing '/ignore-case' shows the short flag is -i.

    # A quick check on how to use 'mkdir':
    mkdir --help

    # Navigate quickly (typing 'cd /Do' and pressing Tab completes the path):
    cd /Documents/
    ```

---

## 4. Lab: reproduce the deliverable

**Goal of the lab:** compose commands with redirection, pipes, and the built-in help, and keep evidence of what each one did. The deliverable is a **redirected file listing**.

**Run these in the Ubuntu container** (start it on the Mac with `docker run -it ubuntu /bin/bash` if it is not already running). The commands are written verbatim so the result can be reproduced.

```bash
# 1. Create a working directory for the course, then create a file with two lines.
mkdir -p ~/linux-course/m2
cd ~/linux-course/m2
echo "line one" > practice.txt      # '>' overwrites: creates the file with this first line
echo "line two" >> practice.txt     # '>>' appends: adds the second line beneath the first

# 2. Confirm both lines are present.
cat practice.txt

# 3. Pipe the output of `ls -l` on /etc through `grep` to keep only .conf entries.
ls -l /etc | grep ".conf"

# 4. Open the manual for grep, close it, then read the short help.
man grep          # navigate with arrows, search with /word, quit with q
grep --help       # concise usage summary, printed to the screen

# 5. Practise tab-completion.
cd ~/linux-cou     # type only this much, then press Tab to complete the path
```

**Expected results and the evidence they provide:**

- **DoD-01 —** `practice.txt` exists on disk and `cat practice.txt` prints **both** lines:
  ```text
  line one
  line two
  ```
  (Capture the real `cat` output from the container here.)
- **DoD-02 —** `line one` was written with **`>`** and `line two` with **`>>`**. If `>` had been used for the second line, the file would have been erased and would then contain only `line two` — the first line would have been lost.
- **DoD-03 —** `ls -l /etc | grep ".conf"` prints only the `/etc` entries whose names contain `.conf`. Read left to right as two steps: **`ls -l /etc` produces** a long-format listing of everything in `/etc`; **`grep ".conf"` keeps** only the lines matching `.conf`. (Capture the real piped output here; note it is the container's `/etc`, which differs from macOS.)
- **DoD-04 —** both `man grep` and `grep --help` were opened, and one thing learned that was not already known is recorded here. For example, `man grep` explains that `-i` makes the search ignore case and `-r` searches a directory tree recursively. (Record the specific discovery made.)
- **DoD-05 —** typing `cd ~/linux-cou` and pressing Tab completed the path to `~/linux-course/` (in the container's bash shell). (Record exactly what Tab produced.)

**Note on `man`:** a minimal container image may not include the `man` program. If `man grep` reports that the command is not found, install it (`apt-get update && apt-get install -y man-db`) or rely on `grep --help`; the brief requires both to be opened, so `man` should be made available if missing.

---

## 5. How the next trainee recreates this from scratch

From a clean Ubuntu container on a Mac, these steps reproduce the whole M2 deliverable unaided:

1. Start the Linux guest: `docker run -it ubuntu /bin/bash`.
2. Create the workspace and the practice file with two lines (`mkdir -p ~/linux-course/m2`, `cd ~/linux-course/m2`, then the two `echo` commands above — one with `>`, one with `>>`).
3. Confirm with `cat practice.txt` — both lines must appear.
4. Produce the redirected/pipe-filtered listing with `ls -l /etc | grep ".conf"`.
5. Open `man grep`, then `grep --help`, and note one new fact.
6. Type `cd ~/linux-cou` and press Tab to see the path completed.

Following those six steps on a fresh machine yields the same `practice.txt` and the same filtered listing: that is the test that these notes are complete.

---

## 6. Key Takeaways & Common Traps

- **The Big Picture:** small single-purpose tools combine. A pipe turns one command's output into the next command's input, and redirection sends output into a file. Together they let simple commands build behaviour larger than any one of them alone.

- **Common Beginner Trap (data loss):** reaching for `>` when `>>` was meant. `>` erases an existing file before writing, so using it on a log or notes file silently deletes everything already there. Choose `>` only for a deliberately fresh file, and `>>` whenever adding to existing content.

- **Common Beginner Trap (environment):** expecting the macOS host's behaviour inside the Linux guest. The container's shell is bash and its `/etc` differs from macOS, so completion behaviour and `ls -l /etc` results are the container's, not the Mac's. Keep the two environments distinct.

---

## 7. References for Further Study

*Verify these concepts or read more here:*

- **[1] Manual Pages:** run `man ls`, `man grep`, `man bash`, and `man 1 echo` inside the container for the authoritative descriptions of every flag used in this module.
- **[2] Command anatomy and coreutils:** [GNU Core Utilities manual](https://www.gnu.org/software/coreutils/manual/coreutils.html) — the official reference for `ls`, `echo`, `cat`, and related tools.
- **[3] Pipes and redirection:** [Bash Guide for Beginners (TLDP)](https://tldp.org/LDP/Bash-Beginners-Guide/html/) — explains `|`, `>`, `>>`, and how the shell connects stdin and stdout.
- **[4] `grep`:** [GNU Grep manual](https://www.gnu.org/software/grep/manual/grep.html) — the full list of `grep` flags and how pattern matching works.
- **[5] Reading man pages:** [man(1) — Linux man pages](https://man7.org/linux/man-pages/man1/man.1.html) — how to navigate and search a manual page.
- **[6] Command breakdowns:** [ExplainShell](https://explainshell.com/) — paste a command and see what each part does.
- **[7] Learning the shell:** [The Linux Command Line (W. Shotts)](https://linuxcommand.org/tlcl.php) — a beginner-friendly, free book covering the shell, pipes, and redirection.
- **[8] File descriptor redirection:** [Bash Reference Manual — Redirections](https://www.gnu.org/software/bash/manual/html_node/Redirections.html) — the precise rules for `>` and `>>`.
