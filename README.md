# 🌐 AWS EC2 + VPC — Launching a Web App Inside a Custom Network
 
**Lab:** Lab 2 — Creating a VPC and Launching an EC2 Instance  
**Console:** AWS Management Console  — Region: Asia Pacific (Sydney) `ap-southeast-2`

---

## 📋 Objective

Build and explore the networking layer of a cloud application by working inside a custom Amazon VPC, then launch an EC2 instance into that network with a User Data bootstrap script that automatically installs Node.js and deploys a web application from S3 on first boot — without any manual configuration after launch.

This lab connects everything: the VPC provides the isolated network, subnets divide it across Availability Zones, the Internet Gateway enables public access, Security Groups control traffic at the instance level, and the EC2 User Data script wires the compute to the storage to produce a running web app — all automatically.

---

## 🛠️ Tools Used

| Tool | Purpose |
|------|---------|
| **AWS Management Console** | Browser-based interface for all AWS service configuration |
| **Amazon VPC** | Inspecting the custom virtual network: CIDR block, subnets, route tables, internet gateway |
| **VPC Resource Map** | Visual diagram showing how subnets, route tables, and gateways connect |
| **Security Groups** | Reviewing inbound and outbound traffic rules for the web app EC2 instance |
| **Amazon EC2** | Launching a `t3.micro` instance using Amazon Linux 2023 AMI |
| **EC2 User Data** | Bootstrap shell script that auto-installs Node.js and deploys the app from S3 on first boot |
| **Amazon S3** | Source for the app code and npm cache pulled down by the User Data script |
| **Coursera / AWS Labs** | Guided lab environment providing the pre-built AWS account and VPC |

---

## ✅ Skills Learned

- Reading and interpreting **VPC configuration details**: IPv4 CIDR block (`10.10.0.0/16`), DNS resolution, tenancy, Network ACL, and route table assignments
- Understanding **CIDR notation** — what `10.10.0.0/16` means in terms of available IP address space
- Navigating the **VPC Resource Map** — a visual topology diagram showing subnets, route tables, internet gateway, and S3 endpoint connected together
- Understanding multi-AZ design: subnets spread across `ap-southeast-2a` and `ap-southeast-2b` for high availability
- Identifying the role of an **Internet Gateway (IGW)** in enabling inbound/outbound internet traffic for public subnets
- Understanding **VPC Endpoints** (Lab S3 Endpoint) — private routing to S3 without traversing the public internet
- Reading **Security Group inbound rules** — HTTPS (TCP 443) and HTTP (TCP 80) open to `0.0.0.0/0`
- Reading **Security Group outbound rules** — DNS (UDP 53), all traffic to the VPC CIDR, and all traffic to the S3 VPC Endpoint prefix list
- Understanding the difference between **inbound rules** (what traffic can reach the instance) and **outbound rules** (what traffic the instance can send)
- Recognizing the **terminated instance state** and understanding EC2 instance lifecycle (pending → running → stopping → terminated)
- Selecting an **AMI (Amazon Machine Image)** — Amazon Linux 2023 — and understanding what an AMI contains (OS, config, base software)
- Choosing an **instance type** (`t3.micro`) and understanding the naming convention (family, generation, size)
- Writing and applying an **EC2 User Data bootstrap script** in bash that runs automatically on first boot
- Understanding how User Data connects EC2 to S3 — the script uses the AWS CLI (`aws s3 cp`) to pull app code from a bucket at launch
- Tracing the full automated deployment pipeline: **EC2 boots → User Data runs → Node.js installs → app downloads from S3 → web server starts**

---

## 📸 Step-by-Step Walkthrough

---

### Step 1 — Inspecting the Lab VPC Configuration

**Service:** VPC → Your VPCs → `vpc-0c3743a00e4836eaa` / Lab VPC  
**Region:** Asia Pacific (Sydney) — `ap-southeast-2`

Explored the **Lab VPC** details. Key configuration values:

| Setting | Value |
|---------|-------|
| **VPC ID** | `vpc-0c3743a00e4836eaa` |
| **IPv4 CIDR** | `10.10.0.0/16` — 65,536 possible IP addresses |
| **DNS Resolution** | Enabled — instances get DNS hostnames automatically |
| **Default VPC** | No — this is a custom VPC built for the lab |
| **Tenancy** | Default — shared hardware (vs. Dedicated for compliance requirements) |
| **Network ACL** | `acl-076f4dddc47fb1027` |
| **Main Route Table** | `rtb-0eaf30b118aed8cda` |
| **Block Public Access** | Off — public subnets can receive internet traffic |

The `/16` prefix means the first 16 bits are fixed (`10.10`), leaving 16 bits for subnets and hosts — a standard VPC size that gives plenty of room to carve out multiple subnets.

<img width="1366" height="768" alt="EC2 VPC (1)" src="https://github.com/user-attachments/assets/06ac9fee-f12e-4e4b-a3b0-3062b57a56e5" />

---

### Step 2 — Reading the VPC Resource Map (Network Topology)

**Service:** VPC → Your VPCs → Lab VPC → Resource Map tab

The VPC Resource Map provides a visual topology diagram of how all networking resources connect inside the Lab VPC. The diagram shows three columns representing the flow of network traffic:

**Subnets (left)** — two public subnets across two Availability Zones for high availability:
- `lab-2-public-subnet-1` in `ap-southeast-2a` — CIDR `10.10.1.0/24`
- `lab-2-public-subnet-2` in `ap-southeast-2b`

**Route Tables (center)** — two route tables controlling traffic flow:
- `lab-2-rtb-public` — the custom route table for public subnets (routes internet traffic to the IGW)
- `rtb-0eaf30b118aed8cda` — the main/default route table

**Connections to other networks (right)** — two exit points:
- `lab-2-igw` — the Internet Gateway, enabling public internet access in/out of the VPC
- `Lab S3 Endpoint` — a VPC Endpoint for private, direct routing to S3 without going through the public internet

This diagram represents a real-world public subnet architecture used by the majority of AWS web applications.

<img width="1366" height="768" alt="EC2 VPC (2)" src="https://github.com/user-attachments/assets/29fcdf15-a408-4021-b21d-3b08ff5d0a20" />

---

### Step 3 — Reviewing Security Group Inbound Rules (WebAppSG)

**Service:** VPC → Security Groups → `WebAppSG`

Navigated to the Security Groups section and inspected the `WebAppSG` security group attached to the Lab VPC. Three security groups exist in this VPC — `WebAppSG`, `Lab VPCEndpointsSG`, and a default group.

The **Inbound Rules** for `WebAppSG` (2 rules):

| Type | Protocol | Port | Source | Description |
|------|----------|------|--------|-------------|
| HTTPS | TCP | **443** | `0.0.0.0/0` | Allow HTTPS from VPC |
| HTTP | TCP | **80** | `0.0.0.0/0` | Allow HTTP from VPC |

Both rules allow traffic from *any* IP address (`0.0.0.0/0`) — meaning this web server is publicly accessible on both HTTP and HTTPS. This is the standard inbound rule configuration for a public-facing web application. The source `0.0.0.0/0` means the internet — any browser, anywhere, can reach this instance on ports 80 and 443.

<img width="1366" height="768" alt="EC2 VPC (3)" src="https://github.com/user-attachments/assets/8647ed39-af09-4cb2-86b9-ed3248675b5d" />

---

### Step 4 — Reviewing Security Group Outbound Rules (WebAppSG)

**Service:** VPC → Security Groups → `WebAppSG` → Outbound Rules

Scrolled to the Outbound Rules for `WebAppSG` (3 rules):

| Type | Protocol | Port | Destination | Description |
|------|----------|------|-------------|-------------|
| DNS (UDP) | UDP | **53** | `0.0.0.0/0` | Allow DNS resolution |
| All traffic | All | All | `sg-082287dbb4c06f349` (Lab VPCEndpointsSG) | Allow all traffic to VPC... |
| All traffic | All | All | `pl-6ca54005` (com.amaz...) | Allow all traffic to S3 re... |

These outbound rules are deliberately scoped — the instance can resolve DNS, communicate with the VPC Endpoints Security Group, and send traffic to S3 (via the VPC Endpoint prefix list) — but it is **not** open to all outbound internet traffic. This is a security best practice: only allow outbound traffic that the application actually needs, and route S3 traffic through the private VPC Endpoint rather than the public internet.

<img width="1366" height="768" alt="EC2 VPC (4)" src="https://github.com/user-attachments/assets/0a34961c-5c81-4947-be04-6ee23ddc3ef1" />

---

### Step 5 — Viewing a Terminated EC2 Instance

**Service:** EC2 → Instances → `i-03230451be2340082`

Viewed a previously launched EC2 instance in the Terminated state — `t3.micro` instance in `ap-southeast-2`. Key observations from the instance details:

- **Instance state: Terminated** — the instance has been shut down and its resources released; terminated instances cannot be restarted
- **Public IPv4 address: —** (blank) — terminated instances lose their public IP
- **Private IPv4 address: —** (blank) — the private IP has been released back to the subnet pool

This step demonstrates understanding the EC2 instance lifecycle: instances move through states — pending → running → stopping → stopped → terminating → terminated. Understanding the difference between *stopped* (paused, can be restarted, EBS volume retained) and *terminated* (permanently deleted) is a critical operational concept, especially for managing cloud costs.

<img width="1366" height="768" alt="EC2 VPC (5)" src="https://github.com/user-attachments/assets/9bbfa05c-13bd-4c75-9ec0-37d841dd4ad2" />

---

### Step 6 — Selecting an AMI for the New EC2 Instance

**Service:** EC2 → Launch Instance → Application and OS Images (AMI)

Started launching a new EC2 instance and selected the Amazon Machine Image. Chose Amazon Linux 2023 AMI (`ami-0ac4101c751eae35f`) with the following specs:

- **Architecture:** 64-bit (x86), UEFI-preferred
- **Virtualization:** hvm (Hardware Virtual Machine)
- **Root device type:** ebs (Elastic Block Store)
- **ENA enabled:** true (Enhanced Networking Adapter — high throughput, low latency)
- **Instance type selected:** `t3.micro` (2 vCPUs, 1 GB RAM — free tier eligible)
- **Security group assigned:** `WebAppSG` (the group inspected in Steps 3 & 4)

An AMI is a pre-configured template containing the OS, configuration, and optionally pre-installed software. Amazon Linux 2023 is AWS's own Linux distribution — optimized for performance on EC2, with long-term support and tight AWS CLI/SDK integration. The AMI selection is the first major decision when launching any EC2 instance.

<img width="1366" height="768" alt="EC2 VPC (6)" src="https://github.com/user-attachments/assets/f91500ae-e4de-435f-a00f-8dfa1971defb" />

---

### Step 7 — EC2 User Data Bootstrap Script

**Service:** EC2 → Launch Instance → Advanced Details → User Data

Entering a User Data shell script that runs automatically when the EC2 instance boots for the first time. This script fully automates the deployment of the web application without any manual SSH or configuration after launch:

```bash
# Installs Node.js
dnf install nodejs20 nodejs20-npm -y

# Downloads an NPM cache from S3 to aid package installation
aws s3 cp s3://S3_BUCKET_NAME/npm-cache.tar.gz \
/var/cache/npm-cache.tar.gz

# Extracts the cache to a directory
mkdir -p /root/.npm
tar xzf /var/cache/npm-cache.tar.gz -C /root/.npm/

# Downloads the web app code as a zip file
mkdir -p /var/app/
aws s3 cp s3://S3_BUCKET_NAME/app.zip \
/var/app/app.zip
```

**What this script does step by step:**
1. Uses `dnf` (Amazon Linux's package manager) to install Node.js 20 and npm
2. Uses the **AWS CLI** (`aws s3 cp`) to copy a pre-cached npm dependency archive from S3 — avoiding slow package downloads at boot
3. Extracts the npm cache to `/root/.npm` so the app can install packages offline
4. Creates the app directory `/var/app/` and downloads the application zip from the same S3 bucket

**Why this is significant:** This is Infrastructure as Code thinking — the server configures itself from a script at launch rather than requiring a human to log in and set things up. This pattern is the foundation of auto-scaling groups, where new instances need to self-configure automatically when demand spikes.

<img width="1366" height="768" alt="EC2 VPC (8)" src="https://github.com/user-attachments/assets/17b41eb2-ba81-4a02-af3c-51dcd28beb37" />
