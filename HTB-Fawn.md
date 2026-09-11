
# HTB — Fawn
<img width="1193" height="690" alt="image" src="https://github.com/user-attachments/assets/a037adaa-4ffa-4855-a8ec-287824c1ca24" />


## Introduction

Fawn is an easy-level machine from Hack The Box. The main objective is to identify the exposed FTP service, determine whether anonymous access is enabled, and retrieve the flag from the server.

---

## 1. Initial Connectivity Test

Before starting the enumeration, I verified that the target machine was reachable by sending ICMP packets to the target IP address.

```bash
ping 10.129.111.9
```

The host responded successfully with no packet loss, confirming that the target was reachable and ready for further enumeration. 

<img width="768" height="237" alt="image" src="https://github.com/user-attachments/assets/858c3b07-7d14-4f0f-af41-e572ef473b15" />

---

## 2. Port and Service Enumeration

After confirming connectivity, I performed an Nmap scan to identify open ports and running services.

```bash
nmap -A 10.129.111.9
```

The scan showed that **FTP was exposed on port 21** and was running **vsftpd 3.0.3**. Nmap also indicated that **anonymous FTP login was allowed**, which immediately became the primary point of interest. 

The relevant finding was:

```text
21/tcp open  ftp  vsftpd 3.0.3
ftp-anon: Anonymous FTP login allowed
```

Since anonymous FTP access can allow unauthenticated users to access files on the server, I proceeded to investigate the service manually.

<img width="876" height="515" alt="image" src="https://github.com/user-attachments/assets/83267e34-127b-4e01-a2b0-d56fc29ed94c" />

---

## 3. Investigating the FTP Client

I first checked the available options of the FTP client to understand the commands that could be used during the interaction with the server.

```bash
ftp
```

The client provides functionality for connecting to remote FTP servers, authenticating, listing directories, downloading files, and transferring data. 

<img width="512" height="527" alt="image" src="https://github.com/user-attachments/assets/591133c6-87e9-45fb-9b7c-00bdd4fc0f5f" />

---

## 4. Connecting to the FTP Service

I then connected directly to the FTP service running on the target.

```bash
ftp 10.129.111.9
```

The server requested a username. Based on the Nmap result indicating that anonymous authentication was enabled, I used:

```text
Name: Anonymous
```

No real password was required, and the login was accepted successfully. 

The server confirmed:

```text
230 Login successful.
Remote system type is UNIX.
Using binary mode to transfer files.
```

This confirmed that the FTP service allowed unauthenticated access through the anonymous account.

<img width="375" height="210" alt="image" src="https://github.com/user-attachments/assets/d1ba3d87-6b45-43d7-a75a-1bac96279385" />

---

## 5. Enumerating the FTP Directory

After successfully authenticating, I listed the contents of the current directory.

```bash
ls
```

The directory contained a file named:

```text
flag.txt
```

Since the file was accessible through the anonymous FTP session, I proceeded to download it to my local machine. 

<img width="1100" height="223" alt="image" src="https://github.com/user-attachments/assets/9a76d164-e342-4dfa-8f3f-a441df3fcb81" />

---

## 6. Downloading the Flag

I used the FTP `get` command to retrieve the file:

```bash
get flag.txt
```

The transfer completed successfully, confirming that the file was readable and downloadable through the anonymous FTP account. 

I then exited the FTP session and checked the downloaded file locally:

```bash
cat flag.txt
```

The flag was:

```text
035db21c881520061c53e0536e44f815
```

<img width="379" height="95" alt="image" src="https://github.com/user-attachments/assets/0f28aeec-becb-4c2e-b9b5-f639117c11aa" />


---

## 7. Conclusion

The machine was compromised by taking advantage of **anonymous FTP access**.

The attack path was straightforward:

```text
Host Discovery
      ↓
Nmap Enumeration
      ↓
FTP on Port 21
      ↓
Anonymous Login Enabled
      ↓
Directory Enumeration
      ↓
flag.txt
      ↓
Flag Retrieved
```

The key takeaway from this machine is that **FTP should not expose anonymous access unless it is explicitly required and properly restricted**. Anonymous access can unintentionally expose sensitive files to unauthenticated users.

Fawn was successfully completed after retrieving the flag through the misconfigured FTP service. 

### Tools Used

* **Nmap** — Port and service enumeration
* **FTP client** — Service interaction and file retrieval
* **Linux CLI** — Local file inspection
