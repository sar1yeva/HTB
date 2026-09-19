# HTB — Redeemer
<img width="828" height="480" alt="image" src="https://github.com/user-attachments/assets/84ceb1c1-bc0f-4519-b594-d3415715f1b9" />

## Introduction

**Redeemer** is an easy-level Hack The Box machine focused on the enumeration and exploitation of a publicly accessible **Redis** service. The machine initially appears to have no services in the default Nmap scan, so a full TCP port scan is required to identify the Redis service running on port `6379`. After connecting to Redis without authentication, the database can be enumerated and the flag retrieved directly.

---

## 1. Initial Connectivity Test

Before performing service enumeration, I verified that the target was reachable:

```bash
ping 10.129.140.210
```

### Command breakdown

* `ping` — sends ICMP Echo Request packets to the target.
* `10.129.140.210` — the target IP address.

The target responded successfully:

```text
3 packets transmitted, 3 received, 0% packet loss
```

This confirmed that the host was reachable and suitable for further enumeration. 

<img width="469" height="172" alt="image" src="https://github.com/user-attachments/assets/1e1342d6-6a39-4a11-be76-897a5cbc9fd1" />


---

## 2. Initial Nmap Scan

After confirming connectivity, I performed a standard Nmap scan:

```bash
nmap -A 10.129.140.210
```

### Command breakdown

* `nmap` — network discovery and service enumeration tool.
* `-A` — enables several advanced detection features, including:

  * OS detection
  * service/version detection
  * default NSE scripts
  * traceroute
* `10.129.140.210` — target IP.

The important result was:

```text
All 1000 scanned ports on 10.129.140.210 are in ignored states.
Not shown: 1000 closed tcp ports
```

At this point, no open TCP service was identified.

This is an important enumeration lesson: **a scan showing no open ports does not necessarily mean that the host has no accessible services.** The default Nmap scan checks the most common 1,000 TCP ports, so services running on less common ports can be missed. 

<img width="743" height="263" alt="image" src="https://github.com/user-attachments/assets/6096c6ba-8b15-441c-9b38-cc118216a33e" />


---

## 3. Full TCP Port Scan

Because the initial scan did not reveal any open ports, I expanded the enumeration to all TCP ports:

```bash
nmap -p- --min-rate 1000 -Pn 10.129.140.210
```

### Command breakdown

```text
-p-
```

This tells Nmap to scan **all 65,535 TCP ports**, rather than only the default top 1,000 ports.

The port range is effectively:

```text
1–65535
```

This is particularly useful when a service is running on a non-standard port.

```text
--min-rate 1000
```

Requests that Nmap send at least approximately 1,000 packets per second, helping speed up the scan.

This should be used carefully because aggressive scan rates can increase network traffic and potentially affect reliability.

```text
-Pn
```

Tells Nmap to treat the host as online and skip host discovery.

This is useful in environments where ICMP or other discovery probes may be filtered.

#### Target

```text
10.129.140.210
```

The target machine.

The full scan identified:

```text
PORT     STATE SERVICE
6379/tcp open  redis
```

This was the key discovery.

Port **6379/tcp** is commonly associated with **Redis**, an in-memory key-value data store. 

The enumeration path had therefore changed from:

```text
No open ports in top 1000
        ↓
Full TCP scan
        ↓
6379/tcp
        ↓
Redis
```

<img width="597" height="198" alt="image" src="https://github.com/user-attachments/assets/9f599604-a33b-4c32-912d-a59ed3c2a0a1" />


---

# 4. Connecting to Redis

With port `6379` identified, I connected to the Redis service using the Redis CLI:

```bash
redis-cli -h 10.129.140.210 -p 6379
```

### Command breakdown

* `redis-cli` — command-line client for interacting with Redis.
* `-h 10.129.140.210` — specifies the Redis server's IP address.
* `-p 6379` — specifies the Redis service port.

The general syntax is:

```bash
redis-cli -h <TARGET_IP> -p <PORT>
```

For this machine:

```bash
redis-cli -h 10.129.140.210 -p 6379
```

Once connected, Redis provides an interactive prompt:

```text
10.129.140.210:6379>
```

No password was required to interact with the service. 

---

# 5. Enumerating the Redis Server

Once connected, I started by requesting server information:

```text
info
```

The command returns information about the Redis instance, including its version, operating system, configuration, memory usage, clients, and other runtime details.

The output revealed:

```text
redis_version:5.0.7
redis_mode:standalone
os:Linux 5.4.0-77-generic x86_64
tcp_port:6379
config_file:/etc/redis/redis.conf
```

### Why `INFO` is useful

The Redis `INFO` command is valuable during enumeration because it can reveal:

* Redis version
* Operating system information
* Architecture
* Running mode
* Process information
* Port configuration
* Memory statistics
* Connected clients
* Server uptime

In this case, it confirmed that the target was running **Redis 5.0.7** on Linux and provided additional configuration information. 

<img width="361" height="521" alt="image" src="https://github.com/user-attachments/assets/95b6ded5-6c0e-4610-b440-c2d1694a76d4" />


---

# 6. Checking the Number of Redis Databases

Next, I checked the available Redis database information:

```text
info keyspace
```

Redis can contain multiple logical databases. The `INFO keyspace` section provides information about databases that currently contain keys.

This is useful because before searching for sensitive information, we need to determine **which database contains data**.

---

# 7. Selecting the Redis Database

Redis databases are selected using:

```text
select <database_number>
```

The default Redis database is normally database `0`.

The command:

```text
SELECT 0
```

switches the current Redis CLI session to database `0`.

A successful selection returns:

```text
OK
```

---

# 8. Enumerating Redis Keys

After selecting the database, I enumerated the available keys:

```text
keys *
```

* `KEYS` — searches for keys matching a pattern.
* `*` — wildcard matching all keys.

The server returned:

```text
1) "temp"
2) "flag"
3) "stor"
4) "numb"
```

This was the most important discovery during Redis enumeration because a key named:

```text
flag
```

was directly exposed. 

---

# 9. Retrieving the Flag

After identifying the `flag` key, I queried its value using:

```text
get flag
```

* `GET` — retrieves the value associated with a Redis string key.
* `flag` — the key identified during the previous `KEYS *` enumeration.

The Redis server returned the flag value.

<img width="272" height="166" alt="image" src="https://github.com/user-attachments/assets/1e6a95ef-0134-49bf-8c2b-26aba9e3e101" />


---

# Defensive Perspective

The machine demonstrates the security impact of exposing a Redis instance without appropriate access controls.

A production Redis deployment should generally be protected through appropriate network segmentation, firewall rules, authentication/access controls, and secure configuration. Redis should not be unnecessarily exposed to untrusted networks.

The key security issue demonstrated here is:

```text
Internet/Untrusted Network
          ↓
      TCP/6379
          ↓
       Redis
          ↓
 No authentication required
          ↓
Database enumeration
          ↓
Sensitive data retrieval
```

---

# Conclusion

**Redeemer** demonstrates an important enumeration principle: **scan beyond the default port range when the initial results do not explain the target's attack surface.**

The complete attack chain was:

```text
Ping
  ↓
Nmap -A
  ↓
No useful ports identified
  ↓
Full TCP scan (-p-)
  ↓
6379/tcp — Redis
  ↓
redis-cli
  ↓
INFO
  ↓
KEYS *
  ↓
flag
  ↓
GET flag
  ↓
Flag Retrieved
```

The machine is particularly useful for learning the relationship between **network enumeration, service identification, service-specific enumeration, and data extraction**. 
