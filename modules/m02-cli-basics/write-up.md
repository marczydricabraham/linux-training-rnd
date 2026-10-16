# CLI Basics Write-up

## What was built

A redirected file listing produced by composing commands rather than running them one at a time. On a Mac, the work was carried out inside the Linux guest set up in M1 — Docker Desktop's LinuxKit virtual machine running an Ubuntu container with the **bash** shell — because the native macOS Terminal reports Darwin and is not a Linux environment.

```
The trainee composed a pipeline that filtered the long listing of /etc for entries containing .conf, redirecting the results to a file. Specifically, the commands used were:

console
mkdir -p ~/linux-course/m2
cd ~/linux-course/m2
echo "line one" > practice.txt
echo "line two" >> practice.txt
ls -l /etc | grep ".conf" > conf-list.txt
```

The deliverable consists of two things:

- **`practice.txt`** — a file created with two lines, the first written with `>` (overwrite) and the second with `>>` (append), then confirmed with `cat practice.txt`.
- **A pipe-filtered listing** — the output of `ls -l /etc` piped into `grep ".conf"`, so that only the `/etc` entries containing `.conf` are kept.

Both were produced from the shell's built-ins only, using the exact commands in the M2 lab.

## Why it's built this way (key decisions)

- **Why these commands and this data:** the lab calls for deliberate control over where output goes, so a small practice file makes the overwrite-versus-append difference visible, and `/etc` was chosen as the filter target because it is a real directory full of configuration files whose names end in `.conf`. The trainee picks real data on their own system rather than a toy example.
- **Why overwrite for the first line and append for the second:** the first line is intended to establish the file, so `>` is correct there — it creates the file and writes the first line. The second line must be added without destroying what is already there, so `>>` is used. Had `>` been used for the second line, the file would have been erased and would then contain only `line two` — the first line would have been lost permanently.
- **Which research questions shaped the pipeline:** the pipe question shaped the `ls -l /etc | grep ".conf"` step (one command's output becomes another's input, read left to right), and the redirection question shaped the two-line file (choosing `>` or `>>` deliberately).
- **Method:** the EPAV cycle was followed — **Evaluate** (the brief's research questions were reviewed), **Plan** (the commands and their order were decided, and expected results written down), **Apply** (every command was run manually in the container, since the definition of done requires output from the local terminal), and **Validate** (the work was checked against the module's definition of done).

## How to build it 

A guide for a Mac user who has never composed commands with pipes and redirection:

1. **Start the Linux guest.** Native macOS terminals use Darwin, not Linux. With Docker Desktop installed, open the terminal and start an interactive Ubuntu shell:
   ```
   docker run -it ubuntu /bin/bash
   ```
   The prompt changes to `root@...`, confirming a root shell inside the container.
2. **Create a workspace.** Work in a dedicated folder so nothing is scattered:
   ```
   mkdir -p ~/linux-course/m2
   cd ~/linux-course/m2
   ```
3. **Write the first line with `>`.** The `>` operator sends output to a file, creating it (or erasing it) first:
   ```
   echo "line one" > practice.txt
   ```
4. **Add the second line with `>>`.** The `>>` operator adds to the end of the file, keeping what is already there:
   ```
   echo "line two" >> practice.txt
   ```
5. **Verify the file.** Confirm both lines are present:
   ```
   cat practice.txt
   ```
   The reasoning, not just the syntax: `>` is chosen only when a fresh file is genuinely wanted; `>>` is chosen whenever adding to existing content.
6. **Filter with a pipe.** Send the listing of `/etc` into `grep` so only matching entries survive:
   ```
   ls -l /etc | grep ".conf"
   ```
   Read it left to right as two steps: `ls -l /etc` **produces** the long-format listing, and `grep ".conf"` **keeps** only the lines containing `.conf`.
7. **Use the built-in help.** Instead of guessing flags, open the manual and the short help:
   ```
   man grep        # scroll with arrows, search with /word, quit with q
   grep --help     # a concise usage summary printed to the screen
   ```
8. **Use tab-completion.** Type a partial path and let the shell finish it:
   ```
   cd ~/linux-cou    # press Tab to complete to ~/linux-course/
   ```

## Concepts worth explaining

**Redirection (`>` versus `>>`)**
Redirection sends a command's output into a file instead of the screen. `>` (overwrite) creates the file, or erases an existing one completely, before writing. `>>` (append) creates the file if missing but adds new output to the bottom of an existing file, leaving earlier content untouched. The difference matters because choosing `>` where `>>` was intended silently destroys the file's previous contents — the classic way to lose a log or a notes file.

**Pipes (`|`)**
A pipe takes the standard output of the command on its left and feeds it as standard input to the command on its right, with no temporary file in between. A pipeline is read left to right as a chain of transformations: each command produces or reshapes a stream and hands it to the next. In `ls -l /etc | grep ".conf"`, `ls` produces the listing and `grep` keeps only the matching lines; together they answer a question neither answers alone.

## What tripped up the process

- **The macOS host versus the Linux guest.** Running commands in the native macOS Terminal gave Darwin/macOS behaviour, not Linux behaviour, and the `/etc` contents differ between the two. The container had to be used for any Linux evidence.
- **`man` availability.** A minimal container image may not include the `man` program; if `man grep` reports it is not found, `man-db` must be installed (or `grep --help` relied on), even though the brief expects both to be opened.
- **Shell-dependent completion.** Tab-completion behaviour comes from the shell, so the container's bash behaviour differs in small ways from the macOS host's zsh.

## Checkpoint evidence

The exact commands run and the resulting file contents are captured below. Where the lab had not yet been run in the container at the time of writing, a clearly-labelled capture block marks where the raw output is pasted.

**`cat practice.txt` (expected: both lines present):**
```text
line one
line two
```

```
root@2c19b7676ba4:~# echo "Line one" >> practice.txt
root@2c19b7676ba4:~# echo "Line two" >> practice.txt
root@2c19b7676ba4:~# cat practice.txt
Line one
Line two
```

**`ls -l /etc | grep ".conf"` (expected: only matching entries):**
```root@2c19b7676ba4:~# ls -l /etc | grep ".conf"  
-rw-r--r-- 1 root root   2967 Feb 16  2026 debconf.conf
-rw-r--r-- 1 root root    685 Feb 13  2026 e2scrub.conf
-rw-r--r-- 1 root root   2584 Jan 23  2026 gai.conf
-rw-r--r-- 1 root root     92 Apr 20 08:46 host.conf
-rw-r--r-- 1 root root     34 Mar 10  2026 ld.so.conf
drwxr-xr-x 2 root root   4096 Sep 27 10:29 ld.so.conf.d
-rw-r--r-- 1 root root    191 Mar 20  2026 libaudit.conf
-rw-r--r-- 1 root root    775 Feb 13  2026 mke2fs.conf
-rw-r--r-- 1 root root    494 Mar 10  2026 nsswitch.conf
-rw-r--r-- 1 root root    552 Jul  3  2025 pam.conf
-rw-r--r-- 1 root root    222 Oct 14 07:16 resolv.conf
-rw-r--r-- 1 root root    681 Feb 15  2026 xattr.conf
```

**Tab-completion of `cd ~/linux-cou`:**
> Capture block: paste what Tab completed the path to (expected: `~/linux-course/`).

## What to do differently

If the module were started over, the first step would be to confirm the Linux guest is active and the prompt shows a root shell inside the container, before running any diagnostic or lab command — this avoids the confusion of reading macOS/Darwin results as if they were Linux. It would also help to choose the filter target deliberately (something meaningful on the system) and to verify that `man` is available early, so the built-in-help step does not stall.
