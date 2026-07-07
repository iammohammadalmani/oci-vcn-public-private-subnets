# OCI VCN with Public & Private Subnets

A hands-on cloud networking project built on Oracle Cloud Infrastructure (OCI) Always Free Tier. This project demonstrates how to design and deploy a secure, production-style network architecture — with a web server exposed to the internet and a database server completely isolated from it.

---

## What This Project Is About

One of the most fundamental concepts in cloud networking is **subnet segmentation** — the idea that not everything in your infrastructure should be equally accessible. Your web server needs to talk to the world. Your database absolutely should not.

This project puts that concept into practice. I built a complete Virtual Cloud Network from scratch on OCI, deployed two Linux servers, and configured the networking so that:

- Anyone on the internet can reach the web server
- Nobody on the internet can reach the database directly
- The web server and database can talk to each other privately
- The database can only be accessed through the web server — acting as a jump host

Everything was built manually through the OCI console to develop a deep understanding of how each component works before moving on to infrastructure-as-code tools like Terraform.

---

## Architecture

```
Internet
    |
Internet Gateway
    |
┌─────────────────────────────────────────┐
│              VCN (10.0.0.0/16)         │
│                                         │
│  ┌───────────────────────────────────┐  │
│  │       Public Subnet (10.0.1.0/24) │  │
│  │                                   │  │
│  │   Web Server (Nginx)              │  │
│  │   Private IP: 10.0.1.54          │  │
│  │   Public IP:  assigned            │  │
│  └────────────────┬──────────────────┘  │
│                   │ (internal traffic)  │
│  ┌────────────────▼──────────────────┐  │
│  │      Private Subnet (10.0.2.0/24) │  │
│  │                                   │  │
│  │   DB Server (MySQL 8.0)           │  │
│  │   Private IP: 10.0.2.106         │  │
│  │   No public IP                    │  │
│  └───────────────────────────────────┘  │
└─────────────────────────────────────────┘
```

---

## Tech Stack

- **Cloud Platform:** Oracle Cloud Infrastructure (OCI) — Always Free Tier
- **Region:** Germany Central (Frankfurt) — eu-frankfurt-1
- **OS:** Oracle Linux 8
- **Web Server:** Nginx
- **Database:** MySQL 8.0
- **Networking:** VCN, Internet Gateway, Route Tables, Security Lists

---

## What I Built

### Networking Layer
- **Virtual Cloud Network (VCN)** with CIDR block `10.0.0.0/16`
- **Internet Gateway** attached to the public route table
- **Public Route Table** routing all internet traffic (`0.0.0.0/0`) through the Internet Gateway
- **Private Route Table** with no internet route — completely isolated
- **Public Security List** allowing inbound SSH (22), HTTP (80), and HTTPS (443)
- **Private Security List** allowing MySQL (3306) only from the public subnet (`10.0.1.0/24`)

### Compute Layer
- **Web Server** deployed in the public subnet with a public IP and Nginx installed
- **DB Server** deployed in the private subnet with no public IP and MySQL 8.0 installed

### Security Decisions
| Decision | Reason |
|---|---|
| DB server has no public IP | Can never be reached directly from the internet |
| Port 3306 only open to 10.0.1.0/24 | Only the web server subnet can query the database |
| SSH key authentication only | No password login allowed on either server |
| Web server acts as jump host | Only way to access the DB server is through the web server first |

---

## Challenges I Ran Into (And How I Solved Them)

This project didn't go perfectly — and that's the point. Real cloud engineering is about troubleshooting, not just following steps.

### Challenge 1: Private Server Couldn't Install Packages
The DB server is on a private subnet with no internet access by design. But that also means it can't reach package repositories to install MySQL. A NAT Gateway would solve this in production but costs money on OCI.

**Solution:** I installed Squid proxy on the web server and routed the DB server's package manager through it. The web server already had internet access — Squid turned it into a proxy for the private subnet.

```bash
# On the web server
sudo dnf install squid -y
sudo systemctl enable squid --now

# On the DB server — install through the proxy
sudo dnf --setopt=proxy=http://10.0.1.54:3128 install mysql-server -y
```

### Challenge 2: MySQL Install Getting Killed
The DB server only has 1GB of RAM (Always Free Tier). Resolving MySQL's dependency tree was consuming too much memory and the OS was killing the process.

**Solution:** Added 2GB of swap space before retrying the install.

```bash
sudo fallocate -l 2G /swapfile
sudo chmod 600 /swapfile
sudo mkswap /swapfile
sudo swapon /swapfile
```

### Challenge 3: MySQL Hostname Resolution
When connecting from the web server to the database, MySQL was resolving the web server's hostname (`webserver.publicsubnet.myvcn.oraclevcn.com`) instead of its IP address, which caused permission mismatches.

**Solution:** Created the MySQL user with a wildcard host (`'%'`) to bypass hostname resolution entirely, then tightened permissions after confirming connectivity.

---

## How to Reproduce This Project

### Prerequisites
- OCI Free Tier account
- SSH key pair

### Step 1: Create the VCN
```
Name:       my-vcn
CIDR:       10.0.0.0/16
DNS Label:  myvcn
```

### Step 2: Create Networking Components
- Internet Gateway → attach to public route table (`0.0.0.0/0`)
- Private Route Table → no routes (isolated)
- Public Security List → allow ports 22, 80, 443 from `0.0.0.0/0`
- Private Security List → allow port 3306 from `10.0.1.0/24` only

### Step 3: Create Subnets
```
Public Subnet:   10.0.1.0/24  → public route table  → public security list
Private Subnet:  10.0.2.0/24  → private route table → private security list
```

### Step 4: Launch Instances
```
Web Server: public subnet  | assign public IP  | install Nginx
DB Server:  private subnet | no public IP      | install MySQL
```

### Step 5: Install Nginx on Web Server
```bash
sudo dnf install nginx -y
sudo systemctl enable nginx --now
sudo firewall-cmd --permanent --add-service=http
sudo firewall-cmd --reload
```

### Step 6: Install MySQL on DB Server (via Squid proxy)
```bash
# On web server first
sudo dnf install squid -y
sudo systemctl enable squid --now
sudo firewall-cmd --permanent --add-port=3128/tcp
sudo firewall-cmd --reload

# On DB server
sudo fallocate -l 2G /swapfile
sudo chmod 600 /swapfile
sudo mkswap /swapfile
sudo swapon /swapfile
sudo dnf --setopt=proxy=http://<WEBSERVER_PRIVATE_IP>:3128 install mysql-server -y
sudo systemctl enable mysqld --now
sudo mysql_secure_installation
```

### Step 7: Verify Connectivity
```bash
# From web server
mysql -h <DB_SERVER_PRIVATE_IP> -u admin -p
# You should get the mysql> prompt
```

---

## Key Concepts Demonstrated

- **Subnet segmentation** — separating public-facing and private resources
- **Security list rules** — port-level traffic control between subnets
- **Route table design** — controlling what traffic can go where
- **Bastion/jump host pattern** — accessing private resources through a public intermediary
- **OS-level firewall management** — firewalld on Oracle Linux
- **Proxy networking** — routing traffic through an intermediary server
- **Linux system administration** — swap space, service management, package installation

---

## What's Next

This project was built manually to understand every component. The next version will use:

- **Terraform** to provision the entire infrastructure as code
- **Docker** to containerise Nginx and MySQL
- **CI/CD pipeline** to automate deployments
- **NAT Gateway** for proper private subnet internet access

---

## Author

**Mohammad Almani**
Cloud Computing Intern @ Extreme Cloud Systems
AWS Cloud Practitioner | OCI Foundations Associate | OCI AI Foundations | Oracle Data Platform Foundations

[LinkedIn](https://www.linkedin.com/in/mohammad-almani-7b898935b) · [GitHub](https://github.com/iammohammadalmani)

