# 🔒 W7-Stunnel — TLS 1.2 Proxy for Legacy Windows 7 SQL Server Clients

> A lightweight Docker-based TLS bridging proxy that allows **Windows 7 machines** to connect to a modern **SQL Server requiring TLS 1.2** — without touching the Windows 7 OS or application.

---

## 🧩 The Problem

Windows 7 was released in 2009. TLS 1.2 became the industry standard years later. The result: legacy Win7 applications and ODBC drivers default to **TLS 1.0**, which modern SQL Server instances reject or flag as insecure.

You have three options:
- ❌ Upgrade Windows 7 — not always possible in industrial or production environments
- ❌ Downgrade SQL Server TLS requirements — a security risk
- ✅ **Put a TLS-upgrading proxy in between** — this repo

---

## 💡 How It Works

This solution uses **[stunnel](https://www.stunnel.org/)** — a battle-tested open source TLS proxy — running inside a **Docker container** on an Ubuntu VM. It sits transparently between your Win7 clients and SQL Server:

```
Win7 Client (plain/TLS1.0)     Ubuntu VM - stunnel (Docker)       SQL Server (TLS 1.2)
          │                                  │                               │
          │                                  │                               │
          │  plain TCP ─── port 1433 ───────>│                               │
          │  TLS 1.0   ─── port 1434 ───────>│                               │
          │                                  │                               │
          │                                  │  upgrade to TLS 1.2 ─────────>│
          │                                  │  port 1433                    │
          │                                  │                               │
          │──── SELECT / INSERT ────────────>│                               │
          │                                  │──── query (TLS 1.2) ─────────>│
          │                                  │◀─── resultset ────────────────│
          │◀─── resultset ───────────────────│                               │
          │                                  │                               │
```

**stunnel accepts** whatever the Win7 client sends (plain TCP or TLS 1.0) and **always forwards** the connection to SQL Server using TLS 1.2. The Win7 app has no idea — it just works.

### 🔄 Traffic flow for a SELECT query

```
Win7 App          Ubuntu/stunnel          SQL Server
   │                    │                     │
   │──── SELECT ───────▶│                     │
   │                    │──── (TLS 1.2) ─────▶│
   │                    │◀─── resultset ───────│
   │◀─── resultset ─────│                     │
   │                    │                     │
   │──── INSERT ───────▶│                     │
   │                    │──── (TLS 1.2) ─────▶│
   │                    │◀─── OK / rowcount ───│
   │◀─── OK / rowcount ─│                     │
```

All traffic is **fully bidirectional** — SELECT, INSERT, UPDATE, stored procedures, error messages — everything passes through transparently.

---

## 📁 Project Structure

```
W7-Stunnel/
├── 🐳 docker-compose.yml       # Container definition, ports, healthcheck
├── 📂 stunnel/
│   └── ⚙️  mssql-proxy.conf    # stunnel TLS bridge configuration
└── 📖 README.md
```

---

## ⚙️ Configuration

### Two ports, two use cases

| Port | Accepts from Win7 | Forwards to SQL Server |
|------|-------------------|------------------------|
| **1433** | Plain TCP (no encryption) | TLS 1.2 |
| **1434** | TLS 1.0 | TLS 1.2 (re-wrapped) |

Use port **1433** if your Win7 app connects without encryption.
Use port **1434** if your Win7 app already negotiates TLS 1.0.

### SQL Server authentication

> ⚠️ stunnel handles **TLS only**. SQL Server credentials (username/password) are **not** configured here — they go in the Win7 connection string.

### Certificate trust

Since SQL Server typically uses a self-signed or internal certificate, chain validation is disabled:
```ini
verifyChain = no   ; don't require a CA-signed certificate
checkHost  = no    ; don't validate hostname in the certificate
```

---

## 🚀 Deployment

### Prerequisites

- 🐧 Ubuntu VM (20.04 or newer) with network access to both Win7 clients and SQL Server
- 🐳 Docker and Docker Compose
- 🔥 Firewall rule: Ubuntu VM → SQL Server port 1433 must be open
- 🌐 Routing: Ubuntu VM must be able to reach the SQL Server subnet

---

### Step 1 — Install Docker on Ubuntu

```bash
sudo apt update
sudo apt install docker.io docker-compose -y
sudo systemctl enable docker
sudo systemctl start docker
```

---

### Step 2 — Set your SQL Server IP

Edit `stunnel/mssql-proxy.conf` and update **both** `connect` lines:

```ini
connect = 192.168.2.50:1433   ; ← replace with your actual SQL Server IP
```

---

### Step 3 — Open firewall ports on Ubuntu

```bash
sudo ufw allow 1433/tcp   # plain TCP connections from Win7
sudo ufw allow 1434/tcp   # TLS 1.0 connections from Win7
sudo ufw reload
```

---

### Step 4 — Add static route (if SQL Server is on a different VLAN/subnet)

```bash
# Temporary — lost on reboot
sudo ip route add 192.168.2.0/24 via 192.168.1.1   # replace with your gateway

# Permanent — edit netplan
sudo nano /etc/netplan/00-installer-config.yaml
```

Add under your network interface:
```yaml
routes:
  - to: 192.168.2.0/24
    via: 192.168.1.1
```

```bash
sudo netplan apply
```

---

### Step 5 — Deploy the container

```bash
git clone https://github.com/daimik/W7-Stunnel.git
cd W7-Stunnel
docker compose up -d
```

---

### Step 6 — Verify it's running

```bash
# Check container status and health
docker compose ps

# Watch live logs
docker compose logs -f

# Test ports are open and listening
nc -zv localhost 1433
nc -zv localhost 1434
```

Expected output:
```
NAME               STATUS                   PORTS
mssql-tls-proxy    Up X minutes (healthy)   0.0.0.0:1433->1433/tcp, 0.0.0.0:1434->1434/tcp
```

---

### Step 7 — Configure Win7 clients

Update the ODBC DSN or connection string on each Win7 machine to point to the **Ubuntu VM IP** instead of SQL Server directly.

| Win7 app type | Old connection | New connection |
|---|---|---|
| Plain TCP | `Server=192.168.2.50,1433` | `Server=<ubuntu-ip>,1433` |
| TLS 1.0 | `Server=192.168.2.50,1433` | `Server=<ubuntu-ip>,1434` |

**Example ODBC connection string:**
```
Driver={SQL Server};
Server=192.168.1.100,1433;
Database=mydb;
Uid=myuser;
Pwd=mypassword;
Encrypt=optional;
TrustServerCertificate=yes;
```

> 💡 `Encrypt=optional` tells the driver that encryption is not mandatory — matching the SQL Server setting. `TrustServerCertificate=yes` prevents the driver from rejecting the SQL Server cert.

---

## 🛠️ Common Commands

```bash
# Start the proxy
docker compose up -d

# Stop the proxy
docker compose down

# Restart after config changes
docker compose restart

# Follow live logs
docker compose logs -f

# Pull latest stunnel image and redeploy
docker compose pull && docker compose up -d
```

---

## 🔍 Troubleshooting

### 🔴 Container won't start
```bash
docker compose logs mssql-proxy
```
Check the stunnel config for syntax errors. Every section must have both `accept` and `connect` lines.

### 🔴 Win7 connects to proxy but gets no data from SQL Server
- Verify the SQL Server IP in `mssql-proxy.conf` is correct
- Test routing from Ubuntu: `ping 192.168.2.50`
- Check SQL Server firewall — it must allow inbound connections from the Ubuntu VM IP
- Check SQL Server is configured to accept TCP/IP connections (SQL Server Configuration Manager → Protocols)

### 🔴 TLS handshake errors in logs
```
SSL_connect: error in SSLv3/TLS write client hello
```
- SQL Server may require a specific TLS cipher — try adding `ciphers = HIGH` to the stunnel section
- If SQL Server requires certificate validation, export the cert and add `CAfile = /path/to/cert.pem` and set `verifyChain = yes`

### 🔴 Port 1433 already in use on Ubuntu
```bash
ss -tlnp | grep 1433
```
If another service is using port 1433, change the host port in `docker-compose.yml`:
```yaml
ports:
  - "14330:1433"   # Win7 connects to port 14330 instead
  - "14340:1434"
```
Then update Win7 connection strings to use the new port number.

### 🔴 Connections drop after idle period
Add keepalive settings to `mssql-proxy.conf`:
```ini
socket = l:TCP_KEEPIDLE=60
socket = l:TCP_KEEPCNT=3
socket = l:TCP_KEEPINTVL=10
```

---

## 🔐 Security Notes

- This proxy is intended for **internal networks only** — do not expose ports 1433/1434 to the internet
- `verifyChain = no` disables certificate validation — acceptable on trusted internal VLANs, but consider adding the SQL Server cert for stricter environments
- SQL Server credentials never pass through this config — they remain in the Win7 connection string
- Consider restricting `accept` to only Win7 client IPs or subnets: `accept = 192.168.1.0/24:1433`

---

## 📦 Built With

- [stunnel](https://www.stunnel.org/) — TLS proxy engine
- [dweomer/stunnel](https://hub.docker.com/r/dweomer/stunnel) — lightweight Docker image
- [Docker Compose](https://docs.docker.com/compose/) — container orchestration
