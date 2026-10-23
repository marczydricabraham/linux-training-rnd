# Linux Lab: Module 5 (Multi-User Environment Approach)

This document records the steps taken to complete Module 5 using a realistic, multi-user environment. By operating as a standard user rather than `root`, we accurately demonstrate real-world privilege escalation (via `sudo`) and permission denials.

## Phase 1: Environment Setup (Run as `root`)

Before starting the lab, we verified and configured standard user accounts to simulate a real-world system.

**1. Verify User Accounts Exist:**
```bash
root@2c19b7676ba4:~/linux-course/m5# grep -E "trainee|colleague" /etc/passwd
trainee:x:1001:1001::/home/trainee:/bin/bash
colleague:x:1002:1002::/home/colleague:/bin/bash
```

**2. Set Passwords and Grant Sudo Privileges:**
```bash
root@2c19b7676ba4:~/linux-course/m5# echo "trainee:password" | chpasswd
root@2c19b7676ba4:~/linux-course/m5# echo "colleague:password" | chpasswd
root@2c19b7676ba4:~/linux-course/m5# usermod -aG sudo trainee
```

**3. Switch to the Standard User:**
```bash
root@2c19b7676ba4:~/linux-course/m5# su - trainee
trainee@2c19b7676ba4:~$ 
```

---

## Phase 2: Module 5 Execution (Run as `trainee`)

### Step 1 & 2: Create a File and Restrict to Owner (Least Permissive)
We created a practice file and immediately locked it down using the least permissive mode that still allows the owner to work with it (`600`), avoiding shortcuts like `777`.

```bash
trainee@2c19b7676ba4:~$ mkdir -p ~/linux-course/m5
trainee@2c19b7676ba4:~$ cd ~/linux-course/m5
trainee@2c19b7676ba4:~/linux-course/m5$ touch shared-report.txt
trainee@2c19b7676ba4:~/linux-course/m5$ chmod 600 shared-report.txt
trainee@2c19b7676ba4:~/linux-course/m5$ ls -l shared-report.txt
-rw------- 1 trainee trainee 0 Oct 22 14:00 shared-report.txt
```

### Step 3: Create an Executable Script
We created a script and added executable permissions specifically for the owner using the symbolic method.

```bash
trainee@2c19b7676ba4:~/linux-course/m5$ touch run.sh
trainee@2c19b7676ba4:~/linux-course/m5$ chmod u+x run.sh
trainee@2c19b7676ba4:~/linux-course/m5$ ls -l run.sh
-rwxr--r-- 1 trainee trainee 0 Oct 22 14:02 run.sh
```

### Step 4: Design and Prove a Restricted Directory
To prove permissions work naturally across different accounts, we created a secure directory and tested access from the `colleague` account.

```bash
trainee@2c19b7676ba4:~/linux-course/m5$ mkdir secure_dir
trainee@2c19b7676ba4:~/linux-course/m5$ chmod 700 secure_dir
trainee@2c19b7676ba4:~/linux-course/m5$ su - colleague -c "ls /home/trainee/linux-course/m5/secure_dir"
Password: 
ls: cannot open file '/home/trainee/linux-course/m5/secure_dir': Permission denied
```
*Result:* The `Permission denied` output perfectly validates that the `colleague` account is blocked from viewing the `trainee`'s restricted directory.

### Step 5: View a Restricted System File via Sudo
Operating as a standard user required us to elevate privileges to view sensitive system files, triggering a natural password prompt for the `trainee` user.

```bash
trainee@2c19b7676ba4:~/linux-course/m5$ sudo cat /etc/shadow
[sudo] password for trainee: 
root:*:20723:0:99999:7:::
...
trainee:$y$j9T$XsQuOrijTB38etCV5VVCA0$nPpzgiqNXIoh5yX...:20748:0:99999:7:::
colleague:$y$j9T$edc7XBoik5xbscLNqfjWR.$9WOnD2PPBvE8h...:20748:0:99999:7:::
```
*Result:* The hashed passwords for the newly created accounts can be seen at the bottom of the secure system file.