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
L'erreur est confirmée : **"Permission denied - bind(2) for port 514"** — mais je vois aussi que **port 5514 est configuré** ✅ et le test `nc` a été envoyé.

Mais `curl | grep mikrotik` retourne **vide** — Logstash n'a pas encore créé l'index. C'est parce que Logstash tourne encore sur l'ancien port 514 et redémarre en boucle.

Fix complet :

```bash
# 1. Arrêter Logstash
sudo systemctl stop logstash

# 2. Vérifier que le port est bien 5514 dans la config
sudo grep "port" /etc/logstash/conf.d/mikrotik-syslog.conf

# 3. Aussi limiter la heap de Logstash (tu as peu de RAM)
sudo bash -c 'cat > /etc/logstash/jvm.options.d/heap.options << EOF
-Xms256m
-Xmx256m
EOF'

# 4. Redémarrer proprement
sudo systemctl start logstash

# 5. Attendre que Logstash démarre complètement (3 min)
sleep 180

# 6. Vérifier le port
sudo ss -ulnp | grep 5514
```

Doit afficher :
```
udp  UNCONN  0  0  0.0.0.0:5514  *  users:(("java",...))
```

Puis tester :
```bash
# Envoyer faux log
echo "<14>Apr  3 12:00:00 mikrotik TEST-ELK" \
  | nc -u -w1 127.0.0.1 5514

sleep 10

# Vérifier index créé
curl http://localhost:9200/_cat/indices?v | grep mikrotik
```

Envoie le résultat du `ss -ulnp | grep 5514` 👇
# Vérifier les nouveaux logs
curl http://localhost:9200/_cat/indices?v | grep mikrotik

# Voir les données
curl "http://localhost:9200/mikrotik-syslog-*/_search?pretty&size=3"
# Vérifier que les docs existent bien
curl "http://localhost:9200/mikrotik-syslog-*/_count"

# Rafraîchir l'index
curl -X POST "http://localhost:9200/mikrotik-syslog-*/_refresh"

Le problème est **"No fields exist in this data view"** — le Data View est mal créé. Il faut le supprimer et recréer correctement.

---

## 🔧 Fix complet — sur Ubuntu terminal

```bash
# 1. Supprimer l'ancien index et recréer avec de nouvelles données
curl -X DELETE "http://localhost:9200/mikrotik-syslog-*"

# 2. Envoyer 5 nouveaux logs de test
for i in 1 2 3 4 5; do
  echo "<14>$(date '+%b %d %H:%M:%S') mikrotik TEST-ELK-$i firewall drop" \
    | nc -u -w1 127.0.0.1 5514
  sleep 1
done

# 3. Attendre et vérifier
sleep 10
curl "http://localhost:9200/mikrotik-syslog-*/_count"
```

---

## 🔧 Puis dans Kibana — Recréer le Data View

**Dismiss** les 2 popups d'abord, puis :

```
☰ → Stack Management → Data Views
→ Supprimer "mikrotik-syslog-*" (icône poubelle 🗑️)
→ Create data view

Name          : mikrotik-syslog-*
Index pattern : mikrotik-syslog-*
Timestamp     : @timestamp
→ Save
```

---

## ⚠️ Si "No fields" persiste — forcer le refresh du mapping

```bash
# Rafraîchir le mapping Kibana
curl -X POST "http://localhost:9200/mikrotik-syslog-*/_refresh"

# Voir les champs disponibles
curl "http://localhost:9200/mikrotik-syslog-*/_mapping?pretty" \
  | grep '"type"' | head -20
```

كيفاش نبني داشبورد Kibana في Windows بناءً على بيانات MikroTik — شرح كامل خطوة بخطوة:---

## المراحل بالتفصيل

### 1. تحضير MikroTik

في الـ **Winbox** أو **Terminal** ديال MikroTik، خص تفعّل اثنين:

**Syslog** (للـ logs):
```
/system logging action
set remote=192.254.223.12 remote-port=514 name=syslog
/system logging
add topics=firewall,info action=syslog
```

**NetFlow** (للـ traffic):
```
/ip traffic-flow
set enabled=yes active-flow-timeout=1m
/ip traffic-flow target
add dst-address=192.254.223.12 port=2055 version=9
```

---

### 2. تنزيل وتثبيت ELK Stack على Windows

من [elastic.co/downloads](https://www.elastic.co/downloads/)، حمّل هاد الثلاثة:
- **Elasticsearch** → فك الضغط في `C:\elasticsearch`
- **Logstash** → `C:\logstash`
- **Kibana** → `C:\kibana`

---

### 3. تشغيل Elasticsearch

```cmd
cd C:\elasticsearch\bin
elasticsearch.bat
```
دير تيستي على `http://localhost:9200` — خاصك تشوف JSON بـ `cluster_name`.

---

### 4. Config Logstash Pipeline

دير فايل `C:\logstash\conf.d\mikrotik.conf`:

```ruby
input {
  syslog {
    port => 514
    type => "mikrotik-syslog"
  }
  udp {
    port => 2055
    codec => netflow { versions => [9] }
    type => "mikrotik-netflow"
  }
}

filter {
  if [type] == "mikrotik-syslog" {
    grok {
      match => { "message" => "%{SYSLOGBASE} %{GREEDYDATA:msg}" }
    }
  }
  if [type] == "mikrotik-netflow" {
    mutate { add_field => { "[@metadata][index]" => "netflow" } }
  }
  geoip {
    source => "src_addr"
    target => "src_geo"
  }
}

output {
  elasticsearch {
    hosts => ["http://localhost:9200"]
    index => "mikrotik-%{type}-%{+YYYY.MM.dd}"
  }
}
```

ثم شغّلو:
```cmd
cd C:\logstash\bin
logstash.bat -f C:\logstash\conf.d\mikrotik.conf
```

---

### 5. تشغيل Kibana

```cmd
cd C:\kibana\bin
kibana.bat
```
دخل على `http://localhost:5601`

---

### 6. بناء Dashboard في Kibana

1. **Stack Management → Index Patterns** → دير index `mikrotik-*`
2. **Discover** → تأكد البيانات راهي توصل
3. **Visualizations** → دير:
   - **Bar chart** لـ Top Source IPs
   - **Line chart** لـ Traffic over time
   - **Pie chart** لـ Protocols (TCP/UDP)
   - **Data table** لـ Firewall drops
   - **Maps** لـ GeoIP visualization
4. **Dashboard** → جمع كل الـ panels

---

##/system logging action
add name=logstash target=remote remote=116.202.19.149 remote-port=5140 syslog-facility=local0 bsd-syslog=yes

/system logging  
add action=logstash topics=firewall
add action=logstash topics=info
add action=logstash topics=warning
add action=logstash topics=errorصيل في جزء معين؟

# 1 - Installer socat
sudo apt install -y socat

# 2 - Lancer le relay en background
sudo socat UDP4-RECVFROM:5140,fork UDP4-SENDTO:116.202.19.149:5140 &

# 3 - Vérifier qu'il écoute
ss -ulnp | grep 5140
ip addr show  
# Voir les paquets UDP qui arrivent
sudo tcpdump -i any udp port 5140 -v
Fix 1 — Relancer socat correctement
Sur Ubuntu VM, ouvre un nouveau terminal et tape commande par commande :
bash# Tuer l'ancien socat
sudo pkill socat
bash# Relancer socat correctement
sudo socat UDP4-RECVFROM:5140,fork UDP4-SENDTO:116.202.19.149:5140 &
bash# Vérifier qu'il écoute (commande séparée)
ss -ulnp | grep 5140
bash# Voir ton IP Ubuntu VM
ip addr show
Je vois le problème ! Le socat s'arrête encore `[1]+ Stopped`. C'est parce qu'il a besoin de tourner en **foreground** ou avec `nohup`.

---

## Fix — Lancer socat correctement

```bash
# Tuer tout
sudo pkill socat

# Lancer avec nohup pour qu'il ne s'arrête pas
sudo nohup socat UDP4-RECVFROM:5140,fork UDP4-SENDTO:116.202.19.149:5140 > /tmp/socat.log 2>&1 &

# Vérifier le PID
echo "PID: $!"

# Vérifier qu'il écoute
sleep 2
ss -ulnp | grep 5140
```

---

## Vérifier que socat tourne vraiment

```bash
# Voir si le processus est actif
ps aux | grep socat

# Voir les logs socat
tail -f /tmp/socat.log
```

---

## Si socat continue à s'arrêter — Alternative avec screen

```bash
sudo apt install -y screen

sudo screen -dmS socat-relay bash -c \
  'socat UDP4-RECVFROM:5140,fork UDP4-SENDTO:116.202.19.149:5140'

# Vérifier
ss -ulnp | grep 5140
```

---

## Après que socat tourne — Changer MikroTik

Dans **Winbox → System → Logging → Actions → logstash** :

- Remote Address: **`192.168.19.128`**
- Remote Port: **`5140`**

Test dans Winbox Terminal :
```
/log info "TEST RELAY OK"
```

Envoie-moi le résultat de `ps aux | grep socat` ! 🔧

Je vois le problème. Logstash tourne en mode `--config.test_and_exit` (validation seulement), et Kibana essaie d'accéder à `localhost:5601` mais ne répond pas encore.

Voici le diagnostic et fix étape par étape :

## 1. Vérifier l'état réel des services
```bash
sudo systemctl status elasticsearch
sudo systemctl status kibana
sudo systemctl status logstash
```

## 2. Le problème probable — Kibana pas encore démarré

La capture montre que tu curl `localhost:5601` dans le terminal de Logstash. Kibana prend **1-2 minutes** à démarrer. Check les logs :

```bash
sudo journalctl -u kibana -f --no-pager | tail -30
```

## 3. Fix si Kibana fail au démarrage

```bash
# Voir l'erreur exacte
sudo journalctl -xe -u kibana | tail -50
```

**Causes fréquentes et fixes :**

```bash
# Cas 1 : Elasticsearch pas encore prêt quand Kibana démarre
curl http://localhost:9200  # teste d'abord ça

# Cas 2 : Kibana config pointe vers mauvais host
sudo nano /etc/kibana/kibana.yml
# Vérifier que ces lignes sont correctes :
# server.host: "0.0.0.0"
# elasticsearch.hosts: ["http://localhost:9200"]
```

## 4. Séquence correcte avec attente

```bash
# Elasticsearch
sudo systemctl start elasticsearch
echo "Attente Elasticsearch..."
until curl -s http://localhost:9200 > /dev/null; do sleep 5; done
echo "Elasticsearch OK ✅"

# Kibana
sudo systemctl start kibana
echo "Attente Kibana (peut prendre 60-90s)..."
until curl -s http://localhost:5601 > /dev/null; do sleep 5; done
echo "Kibana OK ✅"

# Logstash (sans --config.test_and_exit cette fois)
sudo systemctl start logstash
```

## 5. Si Kibana refuse toujours

```bash
# Vérifier le port
ss -tlnp | grep 5601

# Vérifier la version compatibility
curl http://localhost:9200
# La version Elasticsearch doit matcher Kibana (ex: tous les deux 8.x)
```

Envoie-moi le output de `sudo journalctl -xe -u kibana | tail -30` et je te dis exactement ce qui bloque.
echo "nameserver 8.8.8.8" | sudo tee /etc/resolv.conf
