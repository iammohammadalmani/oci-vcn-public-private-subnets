# Complete Setup Guide

This guide walks you through building this exact project from scratch. Every command, every click, every mistake I ran into — it's all here. I wrote this the way I wish someone had written it for me when I was starting out.

Fair warning: this is long. That's intentional. I'd rather over-explain than leave you stuck on something for two hours because I skipped a step.

---

## Before You Start

You'll need:
- An OCI Free Tier account (sign up at cloud.oracle.com — it's free)
- An SSH key pair (OCI can generate one for you during instance creation)
- About 2–3 hours of time
- Patience — cloud networking has a lot of moving parts

One important mindset note: **don't rush the networking section**. It's tempting to click through it quickly, but the VCN, subnets, route tables, and security lists are the foundation of everything. If something is misconfigured there, nothing else will work and you'll spend hours debugging the wrong thing.

---

## Part 1 — Building the Network

### Step 1: Create the VCN

The VCN (Virtual Cloud Network) is your private network in the cloud. Think of it as the building — everything else we create will live inside it.

1. Log into your OCI console at cloud.oracle.com
2. Click the hamburger menu (☰) in the top left
3. Go to **Networking → Virtual Cloud Networks**
4. Click **"Create VCN"**
5. Fill in the details:

```
Name:       my-vcn
CIDR Block: 10.0.0.0/16
DNS Label:  myvcn
```

The CIDR block `10.0.0.0/16` gives you 65,536 private IP addresses to work with. That's way more than we need, but it's the standard starting point.

6. Click **Create VCN**

---

### Step 2: Create the Internet Gateway

The Internet Gateway is what connects your VCN to the public internet. Without it, nothing inside your VCN can reach the outside world — and nothing outside can reach in.

1. Inside your new VCN, click **"Internet Gateways"** on the left sidebar
2. Click **"Create Internet Gateway"**
3. Name it `my-internet-gateway`
4. Click Create

At this point the gateway exists but nothing is using it yet. That comes next.

---

### Step 3: Set Up the Public Route Table

A route table is basically a set of directions — it tells traffic where to go. We need to tell the public subnet: "if traffic is headed for the internet, send it through the Internet Gateway."

1. Still inside your VCN, click **"Route Tables"** on the left
2. You'll see a **Default Route Table** already created — click into it
3. Click **"Add Route Rules"** and fill in:

```
Target Type:        Internet Gateway
Destination CIDR:   0.0.0.0/0
Target:             my-internet-gateway
```

`0.0.0.0/0` means "all traffic" — so anything headed outside the VCN goes through the gateway.

4. Click Save

---

### Step 4: Create the Private Route Table

Your private subnet (where the database lives) should have no internet access at all. We create a separate, empty route table for it — no routes means no internet.

1. Go back to Route Tables → Click **"Create Route Table"**
2. Name it `private-route-table`
3. Don't add any rules
4. Click Create

---

### Step 5: Create Security Lists

Security lists are your firewall rules. They control what traffic is allowed in (ingress) and out (egress) of each subnet.

#### Public Security List

This controls traffic to your web server. It needs to allow:
- SSH (port 22) so you can connect and manage it
- HTTP (port 80) so people can visit your website
- HTTPS (port 443) for secure connections

1. Click **"Security Lists"** on the left
2. Click the **Default Security List** (edit this one)
3. It already has SSH (port 22) allowed — good
4. Click **"Add Ingress Rules"** and add HTTP:

```
Source CIDR:        0.0.0.0/0
Protocol:           TCP
Destination Port:   80
```

5. Add another rule for HTTPS:

```
Source CIDR:        0.0.0.0/0
Protocol:           TCP
Destination Port:   443
```

6. Click Save

#### Private Security List

This controls traffic to your database server. It should only allow MySQL connections (port 3306) — and only from your web server subnet, not from the whole internet.

1. Click **"Create Security List"**
2. Name it `private-security-list`
3. Add this Ingress Rule:

```
Source CIDR:        10.0.1.0/24
Protocol:           TCP
Destination Port:   3306
```

`10.0.1.0/24` is your public subnet's IP range. This means only servers on that subnet (your web server) can connect to MySQL. Nobody from the internet can reach it directly.

4. Add this Egress Rule:

```
Destination CIDR:   0.0.0.0/0
Protocol:           All
```

5. Click Create

---

### Step 6: Create the Public Subnet

Now we create the actual subnets. The public subnet is where your web server will live.

1. Click **"Subnets"** on the left → **"Create Subnet"**
2. Fill in:

```
Name:               public-subnet
Subnet Type:        Regional
CIDR Block:         10.0.1.0/24
Route Table:        Default Route Table
Security List:      Default Security List
DNS Label:          publicsubnet
```

3. Click Create

---

### Step 7: Create the Private Subnet

The private subnet is where your database will live — completely isolated from the internet.

1. Click **"Create Subnet"** again
2. Fill in:

```
Name:               private-subnet
Subnet Type:        Regional
CIDR Block:         10.0.2.0/24
Route Table:        private-route-table
Security List:      private-security-list
DNS Label:          privatesubnet
```

3. Click Create

At this point your entire network is built. Before moving on, take a second to appreciate what you just created — most beginners skip straight to launching servers without understanding the network underneath them. You did it the right way.

---

## Part 2 — Launching the Servers

### Step 8: Create the Web Server

1. Go to **Compute → Instances → Create Instance**
2. Fill in:

```
Name:           webserver
Image:          Oracle Linux 9
Shape:          VM.Standard.E2.1.Micro  (Always Free)
Subnet:         public-subnet
Assign Public IP: Yes
```

3. Under **"Add SSH Keys"**: let OCI generate a key pair for you, then **download both the private and public key files** — save them somewhere safe, you'll need them
4. Click **Create**

Wait for the status to show **Running** before moving on.

---

### Step 9: Create the DB Server

1. Click **"Create Instance"** again
2. Fill in:

```
Name:           dbserver
Image:          Oracle Linux 9
Shape:          VM.Standard.E2.1.Micro  (Always Free)
Subnet:         private-subnet
Assign Public IP: No
```

**Important:** Make sure "Assign Public IP" is set to No. This is what keeps your database invisible to the internet.

3. Use the same SSH key you downloaded for the web server
4. Click **Create**

---

## Part 3 — Connecting to Your Servers

### Step 10: Connect to the Web Server

We'll use OCI Cloud Shell — it's a terminal built into the OCI console that already knows how to connect to your instances. No local setup needed.

1. Click the Cloud Shell icon (`>_`) in the top right of the OCI console
2. Upload your private key file to Cloud Shell using the gear icon ⚙️ → Upload
3. Set the correct permissions on your key:

```bash
chmod 400 ~/ssh-key-2026-07-06.key
```

4. Connect to your web server (replace with your actual public IP):

```bash
ssh -o StrictHostKeyChecking=no -i ~/ssh-key-2026-07-06.key opc@<WEBSERVER_PUBLIC_IP>
```

You'll know it worked when your terminal prompt changes to:
```
[opc@webserver ~]$
```

---

### Step 11: Connect to the DB Server (Through the Web Server)

Your DB server has no public IP, so you can't connect to it directly. You go through the web server first — this is called a **jump host** or **bastion host** pattern.

First, copy your private key to the web server:

```bash
# Run this from Cloud Shell (not from inside the web server)
scp -i ~/ssh-key-2026-07-06.key ~/ssh-key-2026-07-06.key opc@<WEBSERVER_PUBLIC_IP>:~/.ssh/
```

Then SSH into the web server, and from there SSH into the DB server:

```bash
# Connect to web server first
ssh -i ~/ssh-key-2026-07-06.key opc@<WEBSERVER_PUBLIC_IP>

# Then from inside the web server, connect to the DB server
ssh -i ~/.ssh/ssh-key-2026-07-06.key opc@<DBSERVER_PRIVATE_IP>
```

The DB server's private IP looks like `10.0.2.x` — find it under:
**OCI Console → Compute → Instances → dbserver → Networking tab**

---

## Part 4 — Installing and Configuring Software

### Step 12: Install Nginx on the Web Server

Make sure you're connected to the **web server** terminal, then run:

```bash
sudo dnf install nginx -y
sudo systemctl start nginx
sudo systemctl enable nginx
sudo firewall-cmd --permanent --add-service=http
sudo firewall-cmd --permanent --add-service=https
sudo firewall-cmd --reload
```

Test it by opening your browser and going to `http://<WEBSERVER_PUBLIC_IP>` — you should see the Nginx welcome page.

If it doesn't load, check that port 80 is open in your security list (we did this in Step 5) and that the OS firewall allowed it (the `firewall-cmd` lines above handle that).

---

### Step 13: Set Up Squid Proxy on the Web Server

Here's where things get interesting. Your DB server is on a private subnet — which means it can't reach the internet to download MySQL. A NAT Gateway would solve this in production, but it costs money on OCI Free Tier.

The solution: install Squid on the web server and use it as a proxy. The DB server sends its download requests to the web server, which fetches them from the internet and hands them back.

Still on the **web server**, run:

```bash
sudo dnf install squid -y
sudo systemctl start squid
sudo systemctl enable squid
sudo firewall-cmd --permanent --add-port=3128/tcp
sudo firewall-cmd --reload
```

Then open port 3128 in your **public subnet security list** so the DB server can reach it:

**OCI Console → Networking → VCN → Public Subnet → Security List → Add Ingress Rule:**

```
Source CIDR:        10.0.2.0/24
Protocol:           TCP
Destination Port:   3128
```

Note down your web server's **private IP** — you'll need it in the next step. Find it under:
**OCI Console → Compute → Instances → webserver → Networking tab → Private IP**

It'll look like `10.0.1.x`

---

### Step 14: Add Swap Space on the DB Server

Now switch to your **DB server** terminal.

The Always Free Tier instance only has 1GB of RAM. Installing MySQL pulls in a lot of dependencies and will exhaust the memory, causing the OS to kill the install process mid-way. Adding swap space gives it extra breathing room.

```bash
sudo fallocate -l 2G /swapfile
sudo chmod 600 /swapfile
sudo mkswap /swapfile
sudo swapon /swapfile
```

Verify it worked:

```bash
free -h
```

You should see a Swap row showing around 2.0G. If you see it — you're good to go.

---

### Step 15: Install MySQL on the DB Server

Still on the **DB server**, install MySQL through the Squid proxy on your web server:

```bash
sudo dnf --setopt=proxy=http://<WEBSERVER_PRIVATE_IP>:3128 install mysql-server -y
```

Replace `<WEBSERVER_PRIVATE_IP>` with the actual private IP you noted down in Step 13.

This will take a few minutes. When it's done you'll see:

```
Complete!
```

Now start MySQL and run the secure installation script:

```bash
sudo systemctl start mysqld
sudo systemctl enable mysqld
sudo mysql_secure_installation
```

When it asks you questions, answer like this:

```
Set root password?              → Y  (pick a strong password and write it down)
Remove anonymous users?         → Y
Disallow root login remotely?   → Y
Remove test database?           → Y
Reload privilege tables?        → Y
```

---

### Step 16: Open Port 3306 on the DB Server OS Firewall

Even though the OCI security list allows MySQL traffic, Oracle Linux has its own internal firewall that needs to be configured separately.

On the **DB server**:

```bash
sudo firewall-cmd --permanent --add-service=mysql
sudo firewall-cmd --reload
```

Verify it's open:

```bash
sudo firewall-cmd --list-all
```

You should see `mysql` listed under services.

---

### Step 17: Create a MySQL User for Remote Access

By default, MySQL only allows the root user to connect from localhost. We need to create a user that the web server can connect with.

On the **DB server**, log into MySQL:

```bash
sudo mysql -u root -p
```

Then run:

```sql
CREATE USER 'admin'@'%' IDENTIFIED BY 'YourStrongPassword123!';
GRANT ALL PRIVILEGES ON *.* TO 'admin'@'%';
FLUSH PRIVILEGES;
EXIT;
```

The `'%'` means this user can connect from any host. In a real production environment you'd lock this down to a specific IP, but for this project it's fine.

---

## Part 5 — Testing Everything

### Step 18: Install MySQL Client on Web Server

Switch back to your **web server** terminal and install the MySQL client so you can test the connection:

```bash
sudo dnf install mysql -y
```

---

### Step 19: Test the Connection

Now the moment of truth. From the **web server**, connect to the database:

```bash
mysql -h <DBSERVER_PRIVATE_IP> -u admin -p
```

Enter the password you set in Step 17.

If everything is configured correctly, you'll see:

```
Welcome to the MySQL monitor.  Commands end with ; or \g.
Your MySQL connection id is 15
Server version: 8.0.46 Source distribution
mysql>
```

That's it. You're in. Type `exit` to leave MySQL.

---

## What You Just Proved

By getting that `mysql>` prompt, you've confirmed:

- The web server can reach the internet (Nginx is serving pages) ✅
- The DB server cannot be reached from the internet (no public IP) ✅
- The web server and DB server can communicate privately ✅
- MySQL is running and accepting authenticated connections ✅
- Security lists are correctly limiting traffic between subnets ✅

---

## Troubleshooting Reference

These are the actual issues I ran into while building this project and how I fixed them.

### SSH: Connection refused (port 22)
**Cause:** Security list missing port 22 rule, or OS firewall blocking it.
**Fix:**
```bash
sudo firewall-cmd --permanent --add-service=ssh
sudo firewall-cmd --reload
```

### SSH: Connection timed out / hangs forever
**Cause:** No SSH public key stored in the instance — happens when instance is created without properly uploading the public key.
**Fix:** Terminate the instance and recreate it, making sure to upload the `.pub` key file during creation.

### DNF install: "Failed to download metadata" or timeout
**Cause:** Private subnet has no internet access.
**Fix:** Use the Squid proxy workaround from Step 13–15.

### DNF install: "Killed"
**Cause:** Out of memory — 1GB RAM isn't enough for MySQL's dependency resolution.
**Fix:** Add swap space first (Step 14), then retry.

### MySQL: "Can't connect to MySQL server" (error 2003)
**Cause:** OS firewall blocking port 3306.
**Fix:**
```bash
sudo firewall-cmd --permanent --add-service=mysql
sudo firewall-cmd --reload
```

### MySQL: "Host is not allowed to connect" (error 1130)
**Cause:** MySQL user was created for a specific IP but MySQL is resolving the hostname instead.
**Fix:** Create the user with `'%'` wildcard host (Step 17).

### MySQL: "Access denied" (error 1045)
**Cause:** Wrong password or user doesn't exist for that host.
**Fix:** Log in locally as root and recreate the user:
```sql
DROP USER IF EXISTS 'admin'@'%';
CREATE USER 'admin'@'%' IDENTIFIED BY 'YourNewPassword!';
GRANT ALL PRIVILEGES ON *.* TO 'admin'@'%';
FLUSH PRIVILEGES;
```

---

## Author

**Mohammad Almani**
Cloud Computing Intern @ Extreme Cloud Systems
AWS Cloud Practitioner | OCI Foundations Associate | OCI AI Foundations | Oracle Data Platform Foundations

[LinkedIn](https://www.linkedin.com/in/mohammad-almani-7b898935b) · [GitHub](https://github.com/iammohammadalmani)
