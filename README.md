# mssql-tls-proxy

A Docker-based TLS bridging proxy for legacy Windows 7 clients that need to
connect to a SQL Server requiring TLS 1.2.

## Architecture

```
Win7 App (plain or TLS 1.0)
        |
        v
Ubuntu VM - Docker (stunnel)
  port 1433  <-- plain TCP connections
  port 1434  <-- TLS 1.0 connections
        |
        v  (always TLS 1.2)
SQL Server (192.168.2.50:1433)
```

## Project Structure

```
mssql-proxy/
├── docker-compose.yml
├── stunnel/
│   └── mssql-proxy.conf
└── README.md
```

## Prerequisites

- Ubuntu VM with Docker and Docker Compose installed
- Network route from Ubuntu VM to SQL Server subnet
- Firewall rules allowing Ubuntu → SQL Server on port 1433

---

## 1. Install Docker on Ubuntu

```bash
sudo apt update
sudo apt install docker.io docker-compose -y
sudo systemctl enable docker
sudo systemctl start docker
```

---

## 2. Configure SQL Server IP

Edit `stunnel/mssql-proxy.conf` and replace the placeholder IP:

```ini
connect = 192.168.2.50:1433   # <-- replace with your SQL Server IP
```

Both `[mssql-plain]` and `[mssql-tls10]` sections need to be updated.

---

## 3. Open firewall on Ubuntu

```bash
sudo ufw allow 1433/tcp
sudo ufw allow 1434/tcp
sudo ufw reload
```

---

## 4. (If needed) Add static route to SQL Server subnet

If the SQL Server is on a different VLAN/subnet:

```bash
# Temporary (lost on reboot)
sudo ip route add 192.168.2.0/24 via 192.168.1.1   # replace with your gateway

# Permanent - edit netplan config
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

## 5. Deploy

```bash
cd mssql-proxy
docker compose up -d
```

---

## 6. Verify it's running

```bash
# Check container status
docker compose ps

# Watch live logs
docker compose logs -f

# Test port is open
nc -zv localhost 1433
nc -zv localhost 1434
```

---

## 7. Configure Win7 clients

Update the SQL Server connection string or ODBC DSN on each Win7 machine.
Point to the Ubuntu VM IP instead of the SQL Server directly.

| Win7 app type          | Connection string                          |
|------------------------|--------------------------------------------|
| Plain TCP (no TLS)     | `Server=<ubuntu-ip>,1433`                  |
| Uses TLS 1.0           | `Server=<ubuntu-ip>,1434`                  |

Example ODBC connection string:
```
Driver={SQL Server};Server=192.168.1.100,1433;Database=mydb;Uid=myuser;Pwd=mypass;
```

---

## Common Commands

```bash
# Start
docker compose up -d

# Stop
docker compose down

# Restart
docker compose restart

# View logs
docker compose logs -f

# Update stunnel image
docker compose pull && docker compose up -d
```

---

## Troubleshooting

**Container won't start**
```bash
docker compose logs mssql-proxy
```
Check the stunnel config for syntax errors.

**Win7 can connect to proxy but proxy can't reach SQL Server**
- Verify the SQL Server IP in `mssql-proxy.conf`
- Check routing: `ping 192.168.2.50` from Ubuntu
- Check SQL Server firewall allows connections from Ubuntu VM IP

**TLS handshake errors in logs**
- SQL Server may require certificate validation — set `verifyChain = yes`
  and provide the SQL Server certificate

**Port 1433 already in use on Ubuntu**
- Check if SQL Server or another service is running locally: `ss -tlnp | grep 1433`
- Change the host port in `docker-compose.yml`: `"14330:1433"`
  and update Win7 connection strings accordingly
