# Vultr Private VPC Lab with WireGuard VPN, Nginx, and MariaDB

## Project Overview

This project demonstrates how to build a secure private cloud environment using:

* Vultr VPS Instances
* Vultr VPC Networking
* WireGuard VPN
* Nginx Web Server
* MariaDB Database Server
* Linux Routing
* NAT (Network Address Translation)
* UFW Firewall Management

The objective was to create a secure environment where private servers remain inaccessible from the public Internet while remaining accessible through a VPN connection.

---

# Real World Use Case

Imagine a company hosting:

* A web server
* A database server

The database should never be publicly accessible.

Instead, employees connect through a VPN and gain access to private resources.

```text
Employee Laptop
      │
      ▼
VPN Gateway
      │
      ▼
Private Network
      │
 ┌────┴─────┐
 ▼          ▼

Web      Database
Server    Server
```

This lab recreates that architecture.

---

# Final Network Architecture

```text
                           Internet
                               │
                               ▼

                   Gateway VPS
                   Public IP: 45.76.5.134
                   VPC IP:    10.1.96.3
                   VPN IP:    10.10.10.1
                          │
                 WireGuard VPN
                          │
                          ▼

                 Laptop VPN Client
                    10.10.10.2

                          │
                          ▼

                  Vultr VPC Network
                    10.1.96.0/20

              ┌───────────┴───────────┐
              │                       │

              ▼                       ▼

        Nginx Server            MariaDB Server
          10.1.96.4              10.1.96.5
```

---

# Understanding Every IP Address

## Gateway Public IP

```text
45.76.5.134
```

Purpose:

This is the public-facing address used by WireGuard clients to establish VPN connections.

Think of it as the **front door** of the environment.

---

## Gateway VPC IP

```text
10.1.96.3
```

Purpose:

This is the internal address used by the Gateway VPS to communicate with resources inside the Vultr VPC.

Think of it as the **internal hallway** connecting all private servers.

---

## WireGuard VPN Network

```text
10.10.10.0/24
```

Purpose:

A private network dedicated to VPN traffic.

Addresses used:

```text
10.10.10.1 = Gateway WireGuard Interface

10.10.10.2 = Laptop VPN Client
```

Diagram:

```text
Laptop
10.10.10.2
      │
      │ WireGuard Tunnel
      │
10.10.10.1
Gateway
```

---

## Vultr Private VPC Network

```text
10.1.96.0/20
```

Purpose:

Private cloud network used for server-to-server communication.

Addresses used:

```text
10.1.96.3 = Gateway

10.1.96.4 = Nginx Web Server

10.1.96.5 = MariaDB Database Server
```

Diagram:

```text
VPC Network

10.1.96.0/20

├── 10.1.96.3 Gateway
├── 10.1.96.4 Nginx
└── 10.1.96.5 MariaDB
```

---

# Step 1 - Create the VPC

A Vultr VPC was created to allow private communication between cloud resources.

Without a VPC:

```text
Server A
   │
Internet
   │
Server B
```

With a VPC:

```text
Server A
   │
Private Network
   │
Server B
```

Traffic remains inside the cloud provider's internal network.

---

# Step 2 - Deploy the Gateway VPS

Purpose:

* VPN Entry Point
* Router
* NAT Device
* Access Point into the VPC

Diagram:

```text
Internet
    │
    ▼

Gateway VPS

45.76.5.134
```

---

# Step 3 - Deploy the Nginx Server

Purpose:

Host web applications inside the private VPC.

Install:

```bash
apt update
apt install nginx -y
```

Verify:

```bash
systemctl status nginx
```

Check listening ports:

```bash
ss -tulpn | grep :80
```

Expected:

```text
0.0.0.0:80
```

Meaning:

Nginx is listening for HTTP traffic.

---

# Step 4 - Deploy the MariaDB Server

Purpose:

Host databases inside the private VPC.

Install:

```bash
apt update
apt install mariadb-server -y
```

Verify:

```bash
ss -tulpn | grep 3306
```

Expected:

```text
3306
```

Meaning:

MariaDB is listening for database connections.

---

# Step 5 - Configure Firewall Groups

Three Vultr firewall groups were created.

## Gateway Firewall

Allowed:

```text
22/tcp
51820/udp
```

Explanation:

```text
22     = SSH Administration
51820  = WireGuard VPN
```

---

## Nginx Firewall

Allowed:

```text
22/tcp
80/tcp
```

Explanation:

```text
22 = SSH Administration
80 = HTTP Website Access
```

---

## MariaDB Firewall

Allowed:

```text
22/tcp
3306/tcp
```

Explanation:

```text
22   = SSH Administration
3306 = MariaDB Database Service
```

---

# Step 6 - Install WireGuard

Gateway VPS:

```bash
apt install wireguard -y
```

Generate keys:

```bash
wg genkey | tee privatekey | wg pubkey > publickey
```

Purpose:

Create cryptographic identity for the VPN.

---

# Step 7 - Configure WireGuard

Gateway Configuration:

```ini
[Interface]
PrivateKey=<gateway-private-key>
Address=10.10.10.1/24
ListenPort=51820
```

Explanation:

## Address

```text
10.10.10.1/24
```

Creates the VPN interface.

## ListenPort

```text
51820
```

WireGuard listens on UDP port 51820.

Diagram:

```text
Laptop
10.10.10.2
      │
      ▼

WireGuard VPN

      │
      ▼

Gateway
10.10.10.1
```

---

# Step 8 - Enable IP Forwarding

Command:

```bash
sysctl -w net.ipv4.ip_forward=1
```

Explanation:

Linux normally behaves like a workstation.

Enabling IP forwarding allows Linux to behave like a router.

Without forwarding:

```text
Laptop
   │
Gateway
   X
```

Traffic stops.

With forwarding:

```text
Laptop
   │
Gateway
   │
VPC
```

Traffic continues.

---

# Step 9 - Configure NAT

Command:

```bash
iptables -t nat -A POSTROUTING \
-s 10.10.10.0/24 \
-d 10.1.96.0/20 \
-o enp8s0 \
-j MASQUERADE
```

Purpose:

Translate VPN client traffic before entering the VPC.

Command Breakdown:

### iptables

Linux firewall framework.

### -t nat

Use NAT table.

### POSTROUTING

Modify packets after routing decisions.

### -s 10.10.10.0/24

Traffic coming from VPN clients.

### -d 10.1.96.0/20

Traffic heading toward VPC resources.

### -o enp8s0

Outgoing network interface.

### MASQUERADE

Replace source address with gateway address.

---

# Understanding Packet Flow

User opens:

```text
http://10.1.96.4
```

Packet path:

```text
Laptop
10.10.10.2

     │
     ▼

WireGuard Tunnel

     │
     ▼

Gateway
10.10.10.1

     │
IP Forwarding

     ▼

NAT

     ▼

Gateway
10.1.96.3

     │
     ▼

Nginx
10.1.96.4
```

Response path:

```text
Nginx
   │
Gateway
   │
WireGuard
   │
Laptop
```

---

# Validation Testing

## Verify WireGuard

```bash
wg
```

Expected:

```text
latest handshake
```

Meaning:

VPN tunnel established.

---

## Test Nginx Connectivity

```powershell
ping 10.1.96.4
```

Result:

```text
Success
```

---

## Test Database Connectivity

```powershell
ping 10.1.96.5
```

Result:

```text
Success
```

---

## Test Website Access

Browser:

```text
http://10.1.96.4
```

Expected:

```text
Welcome to nginx!
```

---

## Test Database Port

```powershell
Test-NetConnection 10.1.96.5 -Port 3306
```

Expected:

```text
TcpTestSucceeded : True
```

Meaning:

Database port accessible through VPN.

---

# Troubleshooting

## Issue #1 - Nginx Not Reachable

### Symptom

```bash
curl http://10.1.96.4
```

Timed out.

### Root Cause

Port 80 blocked by firewall.

### Resolution

```bash
ufw allow 80/tcp
```

### Validation

```text
Welcome to nginx!
```

Displayed successfully.

Diagram:

```text
Before

Laptop ──X──> Nginx

After

Laptop ─────> Nginx
```

---

## Issue #2 - WireGuard Key Error

### Error

```text
Key is not the correct length or format
```

### Root Cause

Incorrect public key pasted into configuration.

### Resolution

Generated new keys and corrected peer configuration.

Commands:

```bash
wg genkey
wg pubkey
```

---

## Issue #3 - WireGuard Syntax Error

### Error

```text
Line unrecognized
```

### Root Cause

Incorrect formatting:

```ini
PostUp=sysctl ...
```

### Resolution

Correct syntax:

```ini
PostUp = sysctl -w net.ipv4.ip_forward=1
PostDown = sysctl -w net.ipv4.ip_forward=0
```

---

## Issue #4 - VPN Connected but VPC Unreachable

### Symptom

Laptop reached:

```text
10.10.10.1
```

but could not reach:

```text
10.1.96.4
10.1.96.5
```

### Root Cause

Gateway forwarding policy blocked routed traffic.

### Resolution

```bash
ufw route allow in on wg0 out on enp8s0
```

### Validation

Successfully reached both VPC servers.

Diagram:

```text
Laptop
10.10.10.2

      │
      ▼

WireGuard

      ▼

Gateway

      ▼

VPC

      ├── 10.1.96.4
      └── 10.1.96.5
```

---

## Issue #5 - MariaDB Not Reachable

### Symptom

```bash
nc -zv 10.1.96.5 3306
```

Failed.

### Root Cause

MariaDB only listened on localhost.

Configuration:

```ini
bind-address = 127.0.0.1
```

### Resolution

Changed to:

```ini
bind-address = 10.1.96.5
```

Restarted service:

```bash
systemctl restart mariadb
```

Opened firewall:

```bash
ufw allow 3306/tcp
```

### Validation

```bash
nc -zv 10.1.96.5 3306
```

Succeeded.

---

# Final Validation Results

✅ WireGuard Handshake Successful

✅ Laptop Connected to VPN

✅ Gateway Routing Functional

✅ NAT Operational

✅ Nginx Reachable

✅ HTTP Access Successful

✅ MariaDB Reachable

✅ TCP Port 3306 Accessible

✅ End-to-End VPN to VPC Connectivity Verified

---

# Screenshots

Place the merged screenshot inside:

```text
images/Vultr_VPC_Lab_Merged_v2.png
```

Then reference it:

```markdown
![Lab Overview](images/Vultr_VPC_Lab_Merged_v2.png)
```

---

# Skills Demonstrated

* Linux Administration
* Networking Fundamentals
* Routing
* NAT
* VPN Technologies
* WireGuard
* UFW Firewall Management
* Vultr VPC Networking
* Nginx Administration
* MariaDB Administration
* Connectivity Troubleshooting
* Infrastructure Documentation
* End-to-End Network Validation
