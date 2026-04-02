# 🎉 Kibana fonctionne parfaitement !

**"Welcome to Elastic"** sur `localhost:5601` — magnifique ! ✅

---

Maintenant tu veux refaire l'installation sur **Ubuntu** (pas Kali). Voilà le guide complet et propre, **sans les erreurs** qu'on a rencontrées :

---

# 📦 Installation ELK sur Ubuntu — Guide Propre

## Étape 1 — Préparer le système

```bash
sudo apt update && sudo apt upgrade -y
sudo apt install -y curl wget gnupg apt-transport-https
```

## Étape 2 — Ajouter le repo Elastic

```bash
wget -qO - https://artifacts.elastic.co/GPG-KEY-elasticsearch \
  | sudo gpg --dearmor -o /usr/share/keyrings/elasticsearch-keyring.gpg

echo "deb [signed-by=/usr/share/keyrings/elasticsearch-keyring.gpg] \
  https://artifacts.elastic.co/packages/8.x/apt stable main" \
  | sudo tee /etc/apt/sources.list.d/elastic-8.x.list

sudo apt update
```

## Étape 3 — Installer les 3 composants

```bash
sudo apt install -y elasticsearch kibana logstash
```

## Étape 4 — Configurer Elasticsearch ⚠️ (la partie critique)

```bash
# Limiter la heap RAM (important si < 4GB RAM)
sudo bash -c 'cat > /etc/elasticsearch/jvm.options.d/heap.options << EOF
-Xms512m
-Xmx512m
EOF'

# Config principale — propre sans doublons
sudo bash -c 'cat > /etc/elasticsearch/elasticsearch.yml << EOF
path.data: /var/lib/elasticsearch
path.logs: /var/log/elasticsearch
cluster.name: mikrotik-elk
node.name: elk-node-1
network.host: 0.0.0.0
http.port: 9200
discovery.type: single-node
xpack.security.enabled: false
xpack.security.enrollment.enabled: false
xpack.security.http.ssl.enabled: false
xpack.security.transport.ssl.enabled: false
EOF'
```

## Étape 5 — Configurer Kibana

```bash
sudo bash -c 'cat > /etc/kibana/kibana.yml << EOF
server.port: 5601
server.host: "0.0.0.0"
elasticsearch.hosts: ["http://localhost:9200"]
EOF'
```

## Étape 6 — Configurer Logstash (pipeline MikroTik)

```bash
sudo bash -c 'cat > /etc/logstash/conf.d/mikrotik-syslog.conf << EOF
input {
  udp {
    port  => 514
    type  => "syslog"
    codec => "plain"
  }
}

filter {
  grok {
    match => {
      "message" => "%{SYSLOGTIMESTAMP:log_timestamp} %{IPORHOST:mikrotik_host} %{GREEDYDATA:log_message}"
    }
    tag_on_failure => ["_grok_failure"]
  }
  mutate {
    add_field => { "device_type" => "MikroTik" }
  }
  date {
    match => ["log_timestamp", "MMM  d HH:mm:ss", "MMM dd HH:mm:ss"]
    timezone => "Africa/Casablanca"
    target => "@timestamp"
  }
}

output {
  elasticsearch {
    hosts => ["http://localhost:9200"]
    index => "mikrotik-syslog-%{+YYYY.MM.dd}"
  }
  stdout { codec => rubydebug }
}
EOF'
```

## Étape 7 — Démarrer dans l'ordre

```bash
# 1. Elasticsearch d'abord
sudo systemctl enable elasticsearch
sudo systemctl start elasticsearch
sleep 30
curl http://localhost:9200  # doit retourner JSON ✅

# 2. Kibana ensuite
sudo systemctl enable kibana
sudo systemctl start kibana

# 3. Logstash en dernier
sudo /usr/share/logstash/bin/logstash \
  --config.test_and_exit \
  -f /etc/logstash/conf.d/mikrotik-syslog.conf
# doit afficher "Configuration OK" ✅

sudo systemctl enable logstash
sudo systemctl start logstash
```

## Étape 8 — Vérifier tout fonctionne

```bash
# Status des 3 services
sudo systemctl status elasticsearch kibana logstash --no-pager

# Test Elasticsearch
curl http://localhost:9200

# Test Kibana (attendre 2 min)
curl http://localhost:5601
```

## Étape 9 — Configurer MikroTik (WinBox)

```
System > Logging > Actions > [+]
  Name   : remote-elk
  Type   : remote
  Remote : <IP Ubuntu server>
  Port   : 514

System > Logging > Rules > [+]
  Topics : firewall  → Action : remote-elk
  Topics : info      → Action : remote-elk
  Topics : warning   → Action : remote-elk
  Topics : dhcp      → Action : remote-elk
```

## Étape 10 — Kibana : créer l'index pattern

```
Kibana → Stack Management → Index Patterns
  → Create index pattern
  → Name: mikrotik-syslog-*
  → Time field: @timestamp
  → Save

Kibana → Discover → sélectionner mikrotik-syslog-*
→ Tu vois les logs MikroTik en temps réel ! 🎉
```

---

Lance l'installation et envoie-moi les screenshots étape par étape 📸
