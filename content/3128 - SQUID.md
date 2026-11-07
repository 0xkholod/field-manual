---
Primary_category: "[[PROTOCOLS AND SERVICES]]"
title: "3128 - SQUID"
draft: false
banner: "https://images.unsplash.com/photo-1589763472885-46dd5b282f52?q=80&w=1748&auto=format&fit=crop&ixlib=rb-4.0.3&ixid=M3wxMjA3fDB8MHxwaG90by1wYWdlfHx8fGVufDB8fHx8fA%3D%3D"
banner_y: 0.88286
tags:
cssclasses:
---

###### PRIMARY CATEGORY → [[PROTOCOLS AND SERVICES]]

#### *Theory*

> 🛠️⌛

---

#### *Enumeration*

##### *Port Scanning*

```bash
nmap -p3128 --open -sC -sV -v -n -Pn --disable-arp-ping <TARGET>
```

##### *Banner Grabbing*

```bash
nc -vn <TARGET> 3128
```

---

#### *Checking for Open Proxy*

```bash
curl --silent --location --request GET --proxy 'http://<TARGET>:3128' '<URL>'
```

---

#### *Port Scan via SQUID Proxy*

A port scanning can be carried out through a *SQUID* proxy to check which ports are open

- ***[Spose](https://github.com/aancw/spose)***

###### *Setup*

```bash
git clone https://github.com/aancw/spose Spose
cd !$ && python3 -m venv .venv
. !$/bin/activate && pip3 install -r requirements.txt
```

###### *Usage*

```bash
python3 spose.py --proxy 'http://<TARGET>:3128/' --target <TARGET>
```

> [!DANGER]- *e.g.*
>
> ```bash
> python3 spose.py --proxy 'http://192.168.111.189:3128/' --target 192.168.111.189
> ```
>

> [!TLDR]- *Expected Output*
>
> ```bash
> Scanning default common ports
> Using proxy address http://192.168.111.189:3128/
> 192.168.111.189:3306 seems OPEN
> 192.168.111.189:8080 seems OPEN
> ```
>