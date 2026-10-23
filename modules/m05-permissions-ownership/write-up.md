# Users, Permissions, Ownership Write-up

## What was built

A permission and ownership scheme designed and **proved** on a multi-user Linux system — a practice file
restricted to its owner, an owner-only executable script, and a **restricted directory** shown to deny a
second identity. The work was carried out as an ordinary user, not as `root`, so that `sudo` and
`permission denied` behave the way they do on a real system.

On a Mac the work was carried out inside the Linux guest from M1: the Docker Desktop LinuxKit virtual
machine running an Ubuntu container with the **bash** shell (the native macOS Terminal reports Darwin and
is not a Linux environment). A default container runs everything as `root`, which makes every access check
pass — so the first deliberate step was to set up **standard accounts** and work as one of them:

- `trainee` (UID 1001) — the account that owns the practice files and performs the work.
- `colleague` (UID 1002) — a second identity used to test that a restriction actually blocks someone.

The artefacts, all under `~/linux-course/m5` as `trainee`:

- `shared-report.txt` — set to `-rw-------` with `chmod 600`, so only the owner can read or write it.
- `run.sh` — made executable **for the owner only** with `chmod u+x run.sh`, giving `-rwxr--r--`.
- `secure_dir/` — set to mode `700`, and proven to be off-limits to `colleague`.
- `/etc/shadow` — read with `sudo` to observe what elevated privilege reaches.

## Why it's built this way (key decisions)

- **Why a standard user, not `root`:** in a root container the permission model is bypassed, so nothing can
  be denied and nothing is demonstrated. Operating as `trainee` means access is checked against a
  non-privileged identity, which is the only way `permission denied` becomes a real, observable outcome.
  This was the single most important design decision of the module.
- **Why these modes (least permissive):** `600` is the minimum that still lets the owner read and write —
  group and other get nothing. `700` gives the owner full control of the directory and nobody else
  anything. Both were chosen as the **least permissive** settings that still work, with `777` never used as
  a shortcut.
- **Which identity was restricted vs. allowed:** `colleague` is the identity the restriction is *proved*
  against — the attempt to list `secure_dir` is expected to fail. `trainee` is the owner, for whom access
  is expected to succeed. Expressing the restriction through the mode (rather than a per-user exception)
  keeps the rule in one place.
- **Where ownership vs. permission bits mattered:** the file's owner (`trainee`) and its mode are separate
  settings — `chmod` sets the bits, and the bits are interpreted *relative to whoever owns the file*.
  Because `shared-report.txt` was owned by `trainee`, `600` meant "trainee may read and write, nobody else
  may do anything". Changing the owner without the bits (or the bits without the owner) would have aimed
  the same mode at different people.
- **Why `chmod u+x` rather than `chmod 744`:** `u+x` changes **only** the owner's execute bit, leaving the
  group and other triplets untouched — the point being to see one triplet change in isolation.
- **Method:** the EPAV cycle was followed — **Evaluate** (the brief's research questions reviewed),
  **Plan** (accounts, modes, and the identity to test against chosen; expected strings written first),
  **Apply** (every command run in the container, since the definition of done needs real output), and
  **Validate** (results checked against the module's definition of done).

## How to build it (teach it to the next trainee)

A guide to designing and proving a permission scheme, for a Mac user working in the Ubuntu container.

**Phase 1 — set up a multi-user environment (as `root`):**

1. Start the guest and confirm the accounts exist:
   ```
   docker run -it ubuntu /bin/bash
   grep -E "trainee|colleague" /etc/passwd
   ```
2. Give the accounts passwords and grant `trainee` the right to use `sudo`, then switch to it:
   ```
   echo "trainee:password" | chpasswd
   echo "colleague:password" | chpasswd
   usermod -aG sudo trainee
   su - trainee
   ```
   From here on, every command runs as `trainee` — that is what makes the checks below real.

**Phase 2 — the Lab (as `trainee`):**

3. Create a file and restrict it to the owner:
   ```
   mkdir -p ~/linux-course/m5 && cd ~/linux-course/m5
   touch shared-report.txt
   chmod 600 shared-report.txt
   ls -l shared-report.txt        # -rw------- 1 trainee trainee
   ```
4. Add an execute bit for the owner only:
   ```
   touch run.sh
   chmod u+x run.sh
   ls -l run.sh                   # -rwxr--r-- 1 trainee trainee
   ```
5. Build a restricted directory and prove the denial from the second identity:
   ```
   mkdir secure_dir
   chmod 700 secure_dir
   su - colleague -c "ls /home/trainee/linux-course/m5/secure_dir"
   # Password:
   # ls: cannot open file '.../secure_dir': Permission denied
   ```
6. Read a root-only file, which now prompts for `trainee`'s own password:
   ```
   sudo cat /etc/shadow           # [sudo] password for trainee:
   ```
7. To complete the deliverable fully, also prove the **allowed** case: a second directory owned by
   `trainee` with mode `750` and a group containing an allowed user, where that user's attempt
   **succeeds** (see the capture block under Checkpoint evidence).

## Concepts worth explaining

**The identity model**
Every file and process has an owning user and group, and when access is requested the kernel checks the
**identity of the process making the request** — not who is at the keyboard. That is why running as `root`
proves nothing: `root` bypasses the checks, so a restriction can only be observed from a *different*,
non-privileged identity such as `colleague`. `Permission denied` therefore means "the identity running this
command is not one the file allows", not "the file is broken".

**Permission bits vs. ownership**
The `rwx` bits (what may be done) and the ownership (who the file belongs to) are two **separate**
settings. `chmod` changes the bits; `chown`/`chgrp` change ownership; neither changes the other. The bits
are read *relative to the owner and group*, so a `600` file gives "read and write" to **its owner** — which
is why debugging `Permission denied` means checking both *who* owns the file and *what* the bits allow.

## What tripped up the process

- **`root` hides everything.** The container starts as `root`, so the first attempts to restrict access
  proved nothing — there was no second identity to be blocked. Only after creating `trainee` and
  `colleague` and switching to `trainee` did any denial appear.
- **`sudo` has to be granted.** `trainee` could not use `sudo` until added to the `sudo` group
  (`usermod -aG sudo trainee`); before that, elevation simply failed.
- **A directory's `x` bit.** It is tempting to read a directory's execute bit like a file's; it actually
  governs **traverse** (entering the directory and reaching files by name), which is why a directory can be
  readable yet unusable.
- **The `777` temptation.** When an access check fails, `chmod 777` makes the error vanish without fixing
  anything; every mode here is the least permissive that works, and no step uses `777`.
- **`/etc/shadow` is sensitive.** Reading it prints password **hashes**; these were redacted before the
  output was recorded.

## Checkpoint evidence

**A file restricted to its owner (Lab steps 1–2, DoD-01):**
```text
trainee@2c19b7676ba4:~/linux-course/m5$ touch shared-report.txt
trainee@2c19b7676ba4:~/linux-course/m5$ chmod 600 shared-report.txt
trainee@2c19b7676ba4:~/linux-course/m5$ ls -l shared-report.txt
-rw------- 1 trainee trainee 0 Oct 22 14:00 shared-report.txt
```
> Reading the after state: **owner** `rw-`, **group** `---`, **other** `---` — only the owner (DoD-01).
> **PARTIAL** — the brief asks for the output *before and after* the change; the lab capture holds only
> the **after** state. Capture block: run `ls -l shared-report.txt` on a fresh file first and paste the
> default `-rw-r--r--` line alongside the `-rw-------` line above.

**An owner-only executable (Lab step 3, DoD-02):**
```text
trainee@2c19b7676ba4:~/linux-course/m5$ touch run.sh
trainee@2c19b7676ba4:~/linux-course/m5$ chmod u+x run.sh
trainee@2c19b7676ba4:~/linux-course/m5$ ls -l run.sh
-rwxr--r-- 1 trainee trainee 0 Oct 22 14:02 run.sh
```
> `u+x` set the **owner** execute bit only; the group and other triplets (`r--`, `r--`) were left as they
> were (DoD-02).

**A restricted directory denies a second identity (deliverable — denial case):**
```text
trainee@2c19b7676ba4:~/linux-course/m5$ mkdir secure_dir
trainee@2c19b7676ba4:~/linux-course/m5$ chmod 700 secure_dir
trainee@2c19b7676ba4:~/linux-course/m5$ su - colleague -c "ls /home/trainee/linux-course/m5/secure_dir"
Password:
ls: cannot open file '/home/trainee/linux-course/m5/secure_dir': Permission denied
```
> The `colleague` identity is blocked from the `trainee`-owned `700` directory — the restriction is
> demonstrated, not merely configured.

**The allowed case — capture block (deliverable, PARTIAL):**
```text
# Not yet run. Design for the open case, e.g.:
trainee@...:~$ mkdir shared_dir && chmod 750 shared_dir
trainee@...:~$ # add an allowed user to a group, then, as that user:
allowed@...:~$ ls /home/trainee/linux-course/m5/shared_dir      # expected: succeeds
```
> **PARTIAL** — the lab capture proves only the **denied** attempt; the brief's deliverable also asks for
> an **allowed** attempt that succeeds. Paste that output here once run.

**Reading a root-only file via `sudo` (Lab step 4, DoD-03):**
```text
trainee@2c19b7676ba4:~$ sudo cat /etc/shadow
[sudo] password for trainee:
root:*:20723:0:99999:7:::
...
trainee:$y$j9T$REDACTED...:20748:0:99999:7:::
colleague:$y$j9T$REDACTED...:20748:0:99999:7:::
```
> `sudo` prompted for **trainee's own** password (`[sudo] password for trainee:`), not a separate root
> password — the point of the step. Hash fields redacted (DoD-03).

**Decoding `750` — capture block (DoD-04, PARTIAL):**
```text
750 = owner 7 (rwx) + group 5 (r-x) + other 0 (---)
owner: read, write, execute
group: read, execute        (NOT write)
other: no permissions at all
```
> **PARTIAL** — the lab capture does not include a written decode of `750`; this is the expected answer,
> to be confirmed against the learning materials after writing it in your own words first (DoD-04).

## What to do differently

If the module were started over, the **allowed** case would be designed and captured in the same session
as the denied one — a group-based directory (mode `750`) with an allowed user whose listing **succeeds** —
so the deliverable's "restricted for one identity and open for another" is shown in full, not just the
denial. The **before** state of every file would be recorded (the default mode before any `chmod`), and the
`750` decode written down as an exercise rather than reconstructed afterwards. Every mode would still be
the least permissive that works, with `777` never entered.
