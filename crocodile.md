# Hack The Box: Crocodile Writeup — Very Easy

> **Difficulty:** Very Easy | **Tier:** Starting Point Tier 1 | **Tags:** FTP, Anonymous Login, Web Enumeration, Gobuster, Apache

Another day, another box! Today we're combining FTP anonymous access with web directory brute forcing to grab credentials and log into an admin panel 😄 Let's dive in 🚀

---

## What is Crocodile?

Crocodile is a Very Easy difficulty machine from Hack The Box Starting Point Tier 1. It teaches you how to exploit anonymous FTP access to find leaked credentials, then use Gobuster to find hidden login pages on a web server.

**Skills covered:**
- Network reconnaissance with Nmap
- Anonymous FTP login
- Downloading files from FTP
- Directory brute forcing with Gobuster
- Web authentication

---

## Setting Up

Connect to the HTB network using your OpenVPN config file:

```bash
sudo openvpn <your vpn filename>
```

Then spawn the Crocodile machine from the HTB Starting Point page. My target IP was `10.129.1.15`.

<img width="1392" height="110" alt="machine IP" src="https://github.com/user-attachments/assets/d167fbd1-ef47-4a13-9e75-f23421a8c55c" />

---

## Step 1 — Ping the Target

Before doing anything, confirm the machine is reachable:

```bash
ping -c 4 10.129.1.15
```

All 4 packets returned with 0% packet loss. Target is alive. ✅

<img width="542" height="176" alt="ping check" src="https://github.com/user-attachments/assets/7e72950b-c3cc-4c6b-bf5c-666de47a8e03" />

---

## Step 2 — Nmap Scan

Run a full service version scan with default scripts:

```bash
nmap -sVC -T4 10.129.1.15
```

**What `-sC` does:** It runs **default Nmap scripts** during the scan.

<img width="790" height="504" alt="nmap scan" src="https://github.com/user-attachments/assets/d6438e2e-96d3-46e3-8079-b5842a6a1849" />

Two open ports — **FTP on port 21** and **HTTP on port 80**.

The nmap output already tells us:
- FTP version: `vsftpd 3.0.3` 
- FTP anonymous login is allowed — the code returned is `230` 
- Apache version: `Apache httpd 2.4.41` 
- Two interesting files sitting on the FTP server

---

## Step 3 — Anonymous FTP Login

The nmap scan revealed anonymous FTP login is allowed. Connect to FTP using the `anonymous` username:

```bash
ftp anonymous@10.129.1.15
```

Just press Enter. We're in! ✅

<img width="341" height="122" alt="connecting to ftp anonymous" src="https://github.com/user-attachments/assets/01dbc540-ec8a-4a41-8350-148a4e6954c0" />

---

## Step 4 — Download Files from FTP

To download files from the FTP server, we use the `get` command:

```bash
get allowed.userlist
get allowed.userlist.passwd
```

<img width="644" height="108" alt="downloading file" src="https://github.com/user-attachments/assets/c200e386-e862-4230-866b-3730771ca0f0" />

---

## Step 5 — Read the Downloaded Files

Check the contents of both files:

```bash
cat allowed.userlist
```
<img width="290" height="85" alt="usernames" src="https://github.com/user-attachments/assets/44772ec2-442d-49c2-929c-6ad27aff917c" />


The higher-privilege sounding username is **admin**.

```bash
cat allowed.userlist.passwd
```

<img width="339" height="85" alt="passwords" src="https://github.com/user-attachments/assets/f188ef45-43dc-4e0d-b761-3b4ecc511e25" />

We now have a list of usernames and matching passwords. Time to find the login page.

---

## Step 6 — Web Enumeration with Gobuster

Visiting `http://10.129.1.15` shows a business template website. Let's find hidden pages using Gobuster with the `-x` switch to look for specific file types:

```bash
gobuster dir -u http://10.129.1.15/ -w /usr/share/seclists/Discovery/Web-Content/DirBuster-2007_directory-list-2.3-small.txt -x php
```

The `-x php` switch tells Gobuster to search for PHP files specifically.

<img width="675" height="68" alt="task 8" src="https://github.com/user-attachments/assets/0bb7914b-df76-4f10-aa9c-b7ca94d31b25" />

<img width="1161" height="537" alt="gobuster scan" src="https://github.com/user-attachments/assets/bc3d30d6-d515-4d06-949e-e9d8e93c589d" />

---

## Step 7 — Login to the Web Panel

Navigate to `http://10.129.1.15/login.php`:

<img width="1892" height="914" alt="logi page" src="https://github.com/user-attachments/assets/97f181e5-21ce-48ff-86dd-c01d0a76de06" />

Try the credentials from our downloaded files. Using:
- **Username:** `admin`
- **Password:** `rKXM59ESxesUFHAd`
  
<img width="290" height="85" alt="usernames" src="https://github.com/user-attachments/assets/fadfa8b7-196a-4e20-ac54-e9e1b67006d7" />

<img width="339" height="85" alt="passwords" src="https://github.com/user-attachments/assets/272e8a62-b750-4782-980e-f4990ba9f2c2" />

We're in! The admin dashboard loads and shows us the flag. 🎉

<img width="1911" height="1033" alt="adminpage" src="https://github.com/user-attachments/assets/a1ad7b97-1b57-4c64-8d2f-f579b822ee37" />


---

## Task Answers

**Task 1 — What Nmap switch employs default scripts during a scan?**
`-sC`

**Task 2 — What service version is running on port 21?**
`vsftpd 3.0.3`

**Task 3 — What FTP code is returned for Anonymous FTP login allowed?**
`230`

**Task 4 — What username do we provide to log in anonymously to FTP?**
`anonymous`

**Task 5 — What command downloads files from the FTP server?**
`get`

**Task 6 — What higher-privilege username is in allowed.userlist?**
`admin`

**Task 7 — What version of Apache HTTP Server is running?**
`Apache httpd 2.4.41`

**Task 8 — What Gobuster switch specifies file types to search for?**
`-x`

**Task 9 — Which PHP file provides an opportunity to authenticate?**
`login.php`

---

## What I Learned

Crocodile shows how dangerous anonymous FTP access can be. Leaving sensitive files like credential lists on a publicly accessible FTP server is a critical mistake. Here's what to fix in a real environment:

- **Disable anonymous FTP** unless absolutely required
- **Never store credentials in plaintext files** on any server
- **Restrict FTP access** to trusted IPs only
- **Use directory protection** on web admin pages

---

*Written by [Tharindu Pathmasiri](https://github.com/tharindu-L) — IT Undergraduate | Cybersecurity Enthusiast*
