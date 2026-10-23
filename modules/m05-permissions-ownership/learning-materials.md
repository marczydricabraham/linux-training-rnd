# M5 Learning Materials — Users, Permissions, Ownership

> **Purpose.** A self-contained study guide for the ideas named in [`brief.md`](./brief.md): who a user
> and group are, what the permission bits on a file mean, how ownership differs from permission, and what
> `sudo` really does. It teaches the material; it does **not** replace the lab. Design a permission scheme
> on your own system and record what really happened in your own `write-up.md` (created from
> [`write-up-template.md`](./write-up-template.md)).
>
> **Audience.** Beginners who have completed M1–M4 and can navigate the filesystem, compose commands, use
> pipes and redirection, and view or edit files from the terminal. No prior experience with users,
> permission bits, or `sudo` is assumed.
>
> **Scope note.** All command output below is **illustrative teaching output**, not a record of a
> completed lab. Replace every example with your own real state before treating anything here as evidence.
>
> **How to use this guide.** Each section follows **Understand → Try → Verify → Continue**, and each
> answers the **"Research:"** question posed under the matching Scope bullet in the brief. The guided path
> takes roughly **70–100 minutes**. M5 is about **access control** — the model behind the third and most
> common "silent failure": `Permission denied`, and the "works on my machine" habit of reaching for a
> shortcut instead of designing the scheme.

## Contents

1. [Users and groups](#1-users-and-groups)
2. [Permission bits](#2-permission-bits)
3. [Ownership vs. permission](#3-ownership-vs-permission)
4. [Elevated privilege](#4-elevated-privilege)
5. [Lab walkthrough](#5-lab-walkthrough)
6. [Best practices and troubleshooting](#6-best-practices-and-troubleshooting)
7. [References](#7-references)

---

## 1. Users and groups

_Estimated: 15–20 minutes._

### Understand

Every file and every process on a Linux system belongs to a **user** and a **group**, and it is that
identity — not "the person at the keyboard" — that the kernel checks when something asks for access. When
it refuses, the reason is almost always that the *identity running the process* is not the same as the
identity the file allows.

Accounts and groups are defined in plain text files:

| File | Holds |
| --- | --- |
| `/etc/passwd` | one line per account: name, UID, GID, home, shell |
| `/etc/group` | one line per group: name, GID, and its member accounts |
| `/etc/shadow` | password hashes — readable only by **root**, which is why `sudo` will be needed to view it |

A container usually runs everything as **`root`**, so restricting access for "everyone else" proves
nothing — one identity is always `root`. To demonstrate a real restriction, a **second identity** is
needed. This is the brief's stack constraint: at least one additional user or group beyond the default
account.

> **Research answer — how do you see which user and groups the current account has, and where are
> accounts and groups defined?**
> Run `whoami` to print the current user name, `id` to print the user's UID, primary group (GID), and every
> group it belongs to, and `groups` for the group list alone. Accounts are defined in `/etc/passwd`,
> groups in `/etc/group`, and password hashes in `/etc/shadow`; a created account gets a line in the first
> (and, once it has a password, the third).

### Try

```bash
whoami                        # the current user name
id                            # UID, primary group, and all groups — the full identity
groups                        # just the group list
cat /etc/passwd               # one line per account
grep root /etc/group          # which groups root belongs to
sudo useradd -m tester        # create a second identity to test restrictions with
id tester                     # confirm it exists and note its UID/GID
```

> **Slim-image note.** A minimal container may not ship the tool that creates accounts. If `useradd` is
> `command not found`, install it first: `apt update && apt install -y passwd`.

### Verify

- `id` printed a **UID** and a **GID** plus a list of groups — that set is the identity access is checked
  against.
- The new account (`tester`) appears in `/etc/passwd` and in `id tester`, with its own UID and a primary
  group.
- The current account is `root` (in a default container) — which is why a *different* identity is required
  to prove a restriction.

### Continue

An identity has been established. The next section reads the **permission bits** that say what each
identity may do.

---

## 2. Permission bits

_Estimated: 20–25 minutes._

### Understand

Each file carries three sets of three permissions — **read**, **write**, **execute** — for three
audiences: the **owner**, the file's **group**, and **everyone else**. There is no separate access-control
system on top by default: these nine bits, plus ownership, *are* the model.

`ls -l` prints them. Read the line left to right:

```text
-rw-r--r--  1  alice  reporting  1234  Feb 16 10:20  shared-report.txt
│└┬┘└┬┘└┬┘  │  └─┬─┘ └───┬───┘  └─┬┘  └────┬─────┘  └───────┬───────┘
│ │  │  │   │    │       │        │        │                └── name
│ │  │  │   │    │       │        │        └── last modified
│ │  │  │   │    │       │        └── size in bytes
│ │  │  │   │    │       └── group that owns it
│ │  │  │   │    └── user that owns it
│ │  │  │   └── link count
│ │  │  └── permissions for everyone else ("other")
│ │  └── permissions for the group
│ └── permissions for the owner
└── type: "-" file, "d" directory, "l" symlink
```

The same nine bits have two notations, and they map directly. Each bit is worth a number, and the three
numbers are added per audience:

| Symbolic | What it grants | Value |
| --- | --- | --- |
| `r` | read | 4 |
| `w` | write | 2 |
| `x` | execute (or, for a directory, traverse) | 1 |

So `rwx` = 4+2+1 = **7**, `rw-` = **6**, `r-x` = **5**, `r--` = **4**, and `---` = **0**. A three-digit
octal mode is just the owner's value, then the group's, then other's:

| Octal | Symbolic | Owner | Group | Other |
| --- | --- | --- | --- | --- |
| `600` | `rw-------` | read + write | none | none |
| `640` | `rw-r-----` | read + write | read | none |
| `750` | `rwxr-x---` | read + write + execute | read + execute | none |
| `755` | `rwxr-xr-x` | read + write + execute | read + execute | read + execute |

> **Research answer — how do you read the output of `ls -l`, and how do the symbolic and octal notations
> map to each other?**
> `ls -l` prints a type character, then three `rwx` triplets (owner, group, other), then the link count, the
> owning user, the owning group, the size, the date, and the name. The three triplets are the only
> permission information. Symbolic notation names the bits (`r`, `w`, `x`, or `-`); octal notation sums
> each triplet into one digit (`r`=4, `w`=2, `x`=1), giving a three-digit number where each digit is one
> audience's total. They are two spellings of the same nine bits: `rw-` is always `6`, and `600` always
> means `rw-------`.

### Try

```bash
mkdir -p ~/linux-course/m5
cd ~/linux-course/m5
touch shared-report.txt
ls -l shared-report.txt          # default mode, typically 644
chmod 600 shared-report.txt      # owner: read+write; group and other: nothing
ls -l shared-report.txt          # now -rw-------
chmod u=rw,g=r,o= shared-report.txt   # the same idea written symbolically
ls -l shared-report.txt          # -rw-r-----
```

### Verify

- Before the change the string read `-rw-r--r--` (`644`); after `chmod 600` it reads `-rw-------`.
- Reading the positions: **owner** `rw-`, **group** `---`, **other** `---` — only the owner can read or
  write.
- The symbolic form `u=rw,g=r,o=` produced the same result as the octal `640`.

### Continue

The bits are set. The next section separates **who owns** a file from **what the bits allow**.

---

## 3. Ownership vs. permission

_Estimated: 15–20 minutes._

### Understand

Two independent settings describe a file:

- **Ownership** — *which* user and group the file belongs to. Changed with `chown` (user) and `chgrp`
  (group). This decides **whose** owner/group bits apply.
- **Permission bits** — *what* the owner, group, and others may do. Changed with `chmod`.

They are separate: `chown` never changes the bits, and `chmod` never changes who owns the file. A common
mistake is to keep loosening the bits when the real problem is that the wrong identity owns the file —
both `chown` **and** `chmod` have to be checked when debugging "permission denied".

One subtlety: the **execute bit means different things for a file and a directory**.

- For a **file**, `x` means "this file may be run as a program".
- For a **directory**, `x` means **traverse** — you may pass *through* it (and access files inside it by
  name). A directory that is readable (`r`) but not traversable (`x`) can be listed, but you cannot `cd`
  into it or open a named file inside it. So a directory often needs `x` merely to be *used*, not to be
  executed.

> **Research answer — what do `chown` and `chmod` each change, and why does a directory's execute bit
> behave differently from a file's?**
> `chown` changes the **owner** of a file (and `chgrp` the group) — the identity the owner/group bits apply
> to. `chmod` changes the **permission bits** themselves. Changing ownership does not change permissions,
> and vice versa. A **file's** `x` bit grants the right to execute the file; a **directory's** `x` bit
> grants the right to *traverse* it — to enter it and reach files inside by name. That is why a directory
> can be readable but still unusable: `r` lets it be listed, but without `x` nothing inside can be opened
> or entered.

### Try

```bash
touch run.sh
chmod u+x run.sh                 # add execute for the owner only
ls -l run.sh                     # -rwxr--r--  (owner only changed)

sudo chown tester shared-report.txt   # change WHO owns the file (not the bits)
ls -l shared-report.txt               # owner column now shows tester; mode unchanged
sudo chgrp reporting shared-report.txt  # change the owning group
ls -l shared-report.txt
```

### Verify

- `chmod u+x run.sh` changed only the **owner** triplet — the group and other triplets were untouched.
- `chown` changed the **owner column** while the permission string stayed the same — proof the two are
  independent.
- `chgrp` changed the **group column** only.

### Continue

Even a correct scheme can be overridden by one identity: `root`. The last theme is how that override is
reached — and why it is not the first tool to reach for.

---

## 4. Elevated privilege

_Estimated: 10–15 minutes._

### Understand

`sudo` runs **a single command** as another user — `root` by default — after authenticating the caller
**as themselves**. It is not a separate root login, and it does not ask for a root password; it checks
*your* password to confirm that you are allowed to use it at all. `root`, once reached, bypasses the
permission model entirely, which is exactly why using `sudo` "to make the error go away" is a habit to
resist: it hides the mis-set permission instead of fixing it.

Elevated privilege is genuinely warranted for a small, specific set of actions — installing packages,
editing system configuration under `/etc`, or reading root-only files such as `/etc/shadow`. The right
shape is `sudo <command>` for the one command that needs it, **not** `sudo su` to become root for a whole
session.

> **Research answer — what does `sudo` actually do, and when is elevated privilege genuinely warranted?**
> `sudo` checks that the calling user is permitted (using their **own** password) and then executes the
> named command as another user, root by default, without starting a new shell. It is warranted when a task
> genuinely requires root's reach — installing software, changing system-wide configuration, or reading a
> file only root can read. It is *not* warranted merely to bypass a permission error on your own files;
> the correct fix there is to set ownership and permissions deliberately.

### Try

```bash
sudo cat /etc/shadow             # prompts for YOUR password; shows root-only hashes
sudo -l                          # list what you are allowed to run with sudo
sudo id                          # prints the identity sudo ran the command as (root)
```

> **Redaction.** `/etc/shadow` contains password **hashes**. Blur or replace them with `REDACTED` before
> pasting any output, as the trainer directs.

### Verify

- `sudo cat /etc/shadow` prompted for the **current user's own password**, not a separate root password —
  the point of the step.
- `sudo id` reported `uid=0(root)`, showing the command ran with elevated privilege.
- Without `sudo`, `cat /etc/shadow` returns `Permission denied` — proof the file is root-only.

### Continue

The lab now puts identity, permission bits, ownership, and `sudo` together and captures the real output.

---

## 5. Lab walkthrough

> **Reminder.** The output above is illustrative. Perform these steps on your own machine and capture your
> own raw output.

_Estimated: 25–30 minutes._

**Goal.** Read and change permission bits on real files, observe what `sudo` actually asks for, and state
what an octal mode means.

1. Create a practice file and inspect its default permissions:
   ```bash
   mkdir -p ~/linux-course/m5
   cd ~/linux-course/m5
   touch shared-report.txt
   ls -l shared-report.txt      # note the default mode, typically -rw-r--r--
   ```
2. Restrict it so that only the owner can read and write it, and confirm the change:
   ```bash
   chmod 600 shared-report.txt
   ls -l shared-report.txt      # now -rw-------
   ```
3. Create a second file, make it executable by the owner only, and confirm:
   ```bash
   touch run.sh
   chmod u+x run.sh
   ls -l run.sh                 # -rwxr--r--  (owner only)
   ```
4. Use `sudo` to view a file that only root can read by default, confirming that `sudo` prompts for the
   **trainee's own** password and not a separate root password:
   ```bash
   sudo cat /etc/shadow         # redact the hash fields before pasting
   ```
5. Write down, in your own words, what `750` means in terms of owner, group, and other read-write-execute
   bits — **before** checking it against §2 above.

**Deliverable — an access-restricted shared directory.** The brief's deliverable is a directory whose
scheme you designed, where access is **denied to one identity and allowed for another**, with both outcomes
*demonstrated*:

```bash
sudo groupadd reporting                    # a group to grant access to
sudo useradd -m alice                      # the allowed identity
sudo usermod -aG reporting alice           # "alice" is allowed; "tester" is not
sudo mkdir -p ~/linux-course/m5/shared
sudo chown root:reporting ~/linux-course/m5/shared
sudo chmod 750 ~/linux-course/m5/shared    # owner root rwx, group r-x, other none

# Denied: an identity outside the group cannot enter
sudo -u tester ls ~/linux-course/m5/shared    # Permission denied

# Allowed: a member of the group can
sudo -u alice ls ~/linux-course/m5/shared     # succeeds
```

> **Second-identity note.** The restriction only means something because `tester` is a *different* identity
> from the owner. Running as `root` for everything would hide the difference. If creating a user is not
> possible in your environment, use two distinct **groups** with `chgrp` and `su`-style switching instead,
> and say so in the write-up.

**Capture.** The `ls -l` output before and after each `chmod`, the `sudo` prompt behaviour (with hashes
redacted), your own-words explanation of `750`, and the denied **and** successful access attempts on the
shared directory.

**Clean-up.** Nothing to clean up. The practice files stay under `~/linux-course/`.

**Definition of done (self-check).**

- [ ] DoD-01 — the pasted `ls -l shared-report.txt` output before and after `chmod 600` shows the change,
      and the notes read the permission string for the owner, group, and other positions (Lab steps 1–2).
- [ ] DoD-02 — the pasted `ls -l run.sh` output shows the execute bit set for the owner **only**, and the
      notes state what `u+x` changed and what it left alone (Lab step 3).
- [ ] DoD-03 — the notes show that `sudo cat /etc/shadow` prompted for the trainee's **own** password
      rather than a root password, with the password hashes redacted (Lab step 4).
- [ ] DoD-04 — the own-words explanation of `750` was written **before** it was checked against §2, and
      states what owner, group, and other can each do (Lab step 5).
- [ ] DoD-05 — no step uses a blanket `777` "open everything" shortcut; every mode set is the **least
      permissive** one that still works.

---

## 6. Best practices and troubleshooting

_Estimated: 5 minutes._

### Best practices

- Use **`sudo <command>`** for the one command that needs it, not `sudo su`.
- Default to the **least permissive** setting that still works.
- **Never** treat **`777`** as a quick fix for a permission error.
- When debugging "permission denied", check **both** ownership (`chown`) and permission bits (`chmod`).

### Troubleshooting

| Symptom | Likely cause | Fix |
| --- | --- | --- |
| `Permission denied` on your own file | The identity running the command lacks the bit it needs, or the file is owned by someone else. | Check `ls -l`, then grant the **least** bit with `chmod`, or fix ownership with `chown`. |
| Tempted to `chmod 777` | The fastest-looking fix. | Do not. Identify which identity needs access and grant only that bit (`640`, `660`, `750`…). |
| `useradd: command not found` | A slim container lacks the account tooling. | `apt update && apt install -y passwd`. |
| `chmod: invalid mode` | Mixing symbolic and octal syntax in one call. | Use one form: `chmod 640 file` or `chmod u=rw,g=r,o= file`. |
| A directory lists but cannot be entered | The directory lacks the **traverse** (`x`) bit for your identity. | Add `x` for that identity: `chmod u+x dir` or `chmod g+x dir`. |
| `sudo: command not found` | `sudo` is absent in a minimal image. | `apt install -y sudo`, or run the single command as `root` directly. |
| `/etc/shadow` is empty | A fresh container has no accounts with passwords yet. | Create a user first; hashes appear once accounts have passwords. |
| `sudo cat /etc/shadow` prints hash fields | Expected — the file is readable only by root. | Redact the hashes before pasting anywhere. |

---

## 7. References

- [chmod(1) — Linux man page](https://man7.org/linux/man-pages/man1/chmod.1.html) — symbolic and octal modes.
- [chown(1) — Linux man page](https://man7.org/linux/man-pages/man1/chown.1.html) — changing file owner and group.
- [sudo(8) — Linux man page](https://man7.org/linux/man-pages/man8/sudo.8.html) — running one command as another user.
- [passwd(5) — Linux man page](https://man7.org/linux/man-pages/man5/passwd.5.html) — the account file format.
- [shadow(5) — Linux man page](https://man7.org/linux/man-pages/man5/shadow.5.html) — the password-hash file.
- [ls(1) — Linux man page](https://man7.org/linux/man-pages/man1/ls.1.html) — reading the long listing.
- [Red Hat — Linux File Permissions Explained](https://www.redhat.com/sysadmin/linux-file-permissions-explained) — bits, ownership, and the `777` anti-pattern.
- [MIT Missing Semester — Command-line Environment](https://missing.csail.mit.edu/2020/command-line/) — permissions and process control from the shell.

For the module's deliverable, checkpoint set, and definition of done, see [`brief.md`](./brief.md); for an
ordered checklist, see [`tasks.md`](./tasks.md).
