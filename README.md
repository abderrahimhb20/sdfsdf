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

```
# 1️⃣ Rename interfaces (optional but clean)
/interface ethernet
set [ find default-name=ether1 ] name=wan-port
set [ find default-name=ether2 ] name=lan-port

# 2️⃣ Set WAN (DHCP client for internet)
/ip dhcp-client
add interface=wan-port disabled=no

# 3️⃣ Set LAN IP
/ip address
add address=192.168.19.1/24 interface=lan-port

# 4️⃣ Create DHCP pool
/ip pool
add name=pool-lan ranges=192.168.19.10-192.168.19.254

# 5️⃣ DHCP server
/ip dhcp-server
add name=dhcp-lan interface=lan-port address-pool=pool-lan disabled=no

# 6️⃣ DHCP network config
/ip dhcp-server network
add address=192.168.19.0/24 gateway=192.168.19.1 dns-server=8.8.8.8,1.1.1.1

# 7️⃣ DNS config
/ip dns
set servers=8.8.8.8,1.1.1.1 allow-remote-requests=yes

# 8️⃣ NAT (VERY IMPORTANT)
/ip firewall nat

Je vois le problème ! Il y a des indices **internes Kibana** mais **pas de `mikrotik-syslog-*`**. Logstash n'a pas encore reçu de données.

Scrolle vers le bas pour voir le status Logstash, et lance ces commandes :

```bash
# 1. Voir si Logstash tourne
sudo systemctl status logstash --no-pager | head -20

# 2. Logstash écoute sur 514 ?
sudo ss -ulnp | grep 514

# 3. Test manuel — envoyer un faux log
echo "<14>Apr  3 12:00:00 mikrotik TEST-ELK" | nc -u -w1 127.0.0.1 514

# 4. Attendre 5 secondes puis vérifier
sleep 5
curl http://localhost:9200/_cat/indices?v | grep mikrotik
```

---

## ⚠️ Si Logstash n'est pas démarré :

```bash
# Démarrer Logstash
sudo systemctl start logstash

# Attendre 3 minutes (Logstash est lent au démarrage)
sleep 180

# Vérifier
sudo systemctl status logstash --no-pager | head -5
```

---

## ⚠️ Si port 514 nécessite des droits root :

```bash
# Autoriser Logstash à utiliser le port 514 (< 1024)
sudo setcap 'cap_net_bind_service=+ep' \
  /usr/share/logstash/jdk/bin/java

sudo systemctl restart logstash
```

Envoie le résultat du `status logstash` 👇
add chain=srcnat out-interface=wan-port action=masquerade

# 9️⃣ Enable interfaces (just in case)
/interface enable wan-port
/interface enable lan-port
```
Je vois **ERROR** dans les logs Logstash et **port 514 vide** (Logstash n'écoute pas sur 514). Voyons l'erreur exacte :

```bash
# Voir l'erreur complète Logstash
sudo journalctl -u logstash -n 30 --no-pager | grep -i "error\|ERROR\|bind\|permission\|514"
```

---

## 🔧 Fix — Port 514 nécessite root privileges

Le port 514 est un port système (< 1024). Logstash ne peut pas l'utiliser directement. **2 solutions** :

### Solution 1 — Changer le port vers 5514 (plus simple)

```bash
# Modifier la config Logstash
sudo sed -i 's/port  => 514/port  => 5514/' \
  /etc/logstash/conf.d/mikrotik-syslog.conf

# Vérifier
grep "port" /etc/logstash/conf.d/mikrotik-syslog.conf

# Redémarrer
sudo systemctl restart logstash
sleep 30

# Vérifier que le port est ouvert
sudo ss -ulnp | grep 5514
```

### Solution 2 — Rediriger 514 → 5514 avec iptables

```bash
sudo iptables -t nat -A PREROUTING -p udp --dport 514 \
  -j REDIRECT --to-port 5514
```

---

## Après le fix — tester :

```bash
# Envoyer un faux log sur le nouveau port
echo "<14>Apr  3 12:00:00 mikrotik TEST-ELK" \
  | nc -u -w1 127.0.0.1 5514

# Attendre et vérifier l'index
sleep 10
curl http://localhost:9200/_cat/indices?v | grep mikrotik
```

Lance **Solution 1** et envoie le résultat 👇
