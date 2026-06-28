## Traçabilité — Conformité à la loi algérienne

### Qu'est-ce que la traçabilité ?

Capacité à enregistrer et conserver toutes les actions des utilisateurs sur le réseau pour les retracer en cas d'incident ou d'enquête légale.

**Pour le projet UTM IFAG :** Savoir qui (utilisateur) → s'est connecté quand (horodatage) → depuis où (IP/MAC) → a fait quoi (sites visités, volume) → a été déconnecté quand.

### Pourquoi la faire ?

| Raison | Explication |
|--------|-------------|
| **Loi algérienne** | Obligation légale de conservation des logs |
| **Sécurité** | Détecter les intrusions après coup |
| **Conformité PFE** | Critère clé d'évaluation du projet |
| **Administration** | Savoir qui fait quoi sur le réseau IFAG |

### Cadre légal algérien (Loi 18-07 + Cybersécurité)

Exigences pour un établissement comme l'IFAG :

1. **Conservation des logs** — minimum 12 mois (connexions, déconnexions, volume)
2. **Identification des utilisateurs** — chaque connexion liée à une identité unique (Samba AD)
3. **Journalisation des accès** — qui, quand, combien de temps, quel volume
4. **Protection des logs** — journaux intègres et non modifiables
5. **Mise à disposition des autorités** — en cas de réquisition légale

---

### Ce qu'on peut logger sur le réseau IFAG

| Donnée                  | Source                  | Exemple                                 |
| ----------------------- | ----------------------- | --------------------------------------- |
| Connexions utilisateurs | FreeRADIUS (Accounting) | pfeuser → 192.168.1.102 → 30min → 150MB |
| Sites visités           | Squid (proxy logs)      | pfeuser → facebook.com → 10:32          |
| Traffic IP              | pfSense (filter.log)    | 192.168.1.102 → 8.8.8.8:53              |
| Alertes sécurité        | Suricata (IDS logs)     | Scan Nmap détecté depuis 192.168.1.102  |
| Sessions VPN            | WireGuard               | client VPN connecté 45min               |
| Authentifications       | FreeRADIUS (auth.log)   | Access-Accept / Access-Reject           |

---

### Vérification des logs existants sur pfSense

**Logs disponibles dans Status → System Logs :**

| Log | Utile pour traçabilité ? | Ce qu'il contient |
|-----|--------------------------|-------------------|
| **System** | ✅ | Événements système, démarrage, erreurs |
| **Firewall** | ✅ **Oui** | Tout le trafic bloqué/autorisé (IP src/dst, ports) |
| **DHCP** | ✅ **Oui** | Bail IP → Quelle IP attribuée à quelle machine |
| **Authentication** | ✅ **Oui** | Tentatives de connexion admin pfSense |
| **IPsec** | ❌ | VPN IPsec (pas utilisé) |
| **PPP** | ❌ | Connexions PPP (pas utilisé) |
| **PPPoE/L2TP** | ❌ | Pas utilisé |
| **OpenVPN** | ❌ | Pas utilisé (on a WireGuard) |
| **NTP** | ❌ | Synchronisation horaire |
| **Packages** | ❌ | Installation/suppression de paquets |
| **Settings** | ❌ | Changements de configuration |
| **General** | ✅ | Logs généraux (multi-services) |
| **Gateways** | ✅ | État des passerelles |
| **Routing** | ❌ | Tables de routage |
| **DNS Resolver** | ✅ | Requêtes DNS |

**Les plus importants pour la loi algérienne :**
1. **Firewall** → toute connexion réseau tracée
2. **DHCP** → lien IP ⇔ machine
3. **Authentication** → accès admin

**1. RADIUS Accounting (déjà activé ✅)**
- **Services → FreeRADIUS → Accounting** → vérifier que les logs sont stockés
- Logs visibles dans : **Status → System Logs → FreeRADIUS**

**2. Logs firewall**
- **Status → System Logs → Firewall** → voir le trafic filtré
- Activer le logging sur les règles si nécessaire

**3. Sessions actives du portail captif**
- **Status → Captive Portal** → voir les utilisateurs connectés

**4. FreeRADIUS détaillé**
- SSH pfSense : `tail -f /var/log/radiusd.log`

---

### Configuration de l'envoi des logs vers Ubuntu DMZ (192.168.2.20)

Pour une vraie traçabilité, on centralise les logs sur le serveur Samba/Ubuntu (console uniquement, pas d'interface graphique).

#### Pourquoi envoyer les logs vers l'Ubuntu DMZ ?

| Raison | Explication |
|--------|-------------|
| **Conservation (loi algérienne)** | Logs pfSense temporaires → Ubuntu = stockage longue durée (12+ mois) |
| **Intégrité** | Logs sur serveur séparé = preuve légale inaltérable (même si pfSense est compromis) |
| **Centralisation** | Tous les logs au même endroit (pfSense + RADIUS + Squid + Suricata) |
| **Préparation Grafana** | Les dashboards ont besoin de logs centralisés |
| **Valeur PFE** | Architecture professionnelle de type SIEM |

#### Étape 1 — Sur l'Ubuntu DMZ (console SSH, 192.168.2.20)

Se connecter en SSH ou console :
```bash
ssh admin-samba@192.168.2.20
# ou directement sur la console de la VM
```

Installer rsyslog (serveur syslog) :
```bash
sudo apt update && sudo apt install rsyslog -y
```

Configurer rsyslog pour écouter les logs distants :
```bash
sudo sed -i 's/^#module(load="imudp")/module(load="imudp")/' /etc/rsyslog.conf
sudo sed -i 's/^#input(type="imudp" port="514")/input(type="imudp" port="514")/' /etc/rsyslog.conf
```

Redémarrer le service :
```bash
sudo systemctl restart rsyslog
```

Ouvrir le port firewall (ufw) :
```bash
sudo ufw allow 514/udp
# ou si pas d'ufw : sudo iptables -A INPUT -p udp --dport 514 -j ACCEPT
```

#### Étape 2 — Sur pfSense (interface web)

**Status → System Logs → Settings :**
- ✅ **Enable Remote Syslog Logging**
- **Remote Log Server 1 :** `192.168.2.20:514`
- **Syslog Contents :** tout cocher (Firewall, Captive Portal, FreeRADIUS, Squid, Suricata, DHCP)
- **Save → Apply**

#### Étape 3 — Vérification sur Ubuntu

```bash
# Voir les logs pfSense arriver en temps réel
sudo tail -f /var/log/syslog | grep -E "192.168.1|pfSense"

# Voir les logs spécifiques FreeRADIUS
sudo tail -f /var/log/syslog | grep -i radius

# Voir les logs du portail captif
sudo tail -f /var/log/syslog | grep -i captive
```

#### Étape 4 — Créer un fichier de log dédié

Pour séparer les logs pfSense des logs Ubuntu (optionnel mais recommandé).

**Attention :** la syntaxe `if ... == ...` ne fonctionne pas sur toutes les versions de rsyslog (v8.2312.0 et +). Utiliser la syntaxe legacy :

**Option A — Avec nano (éditeur) :**
```bash
sudo nano /etc/rsyslog.d/10-pfsense.conf
```
Coller ce contenu :
```
:fromhost-ip, isequal, "192.168.1.1"  /var/log/pfsense.log
:fromhost-ip, isequal, "192.168.1.1"  stop
```
Ctrl+X → Y → Enter

**Option B — Commande directe (copier-coller) :**
```bash
sudo tee /etc/rsyslog.d/10-pfsense.conf > /dev/null <<'EOF'
:fromhost-ip, isequal, "192.168.1.1"  /var/log/pfsense.log
:fromhost-ip, isequal, "192.168.1.1"  stop
EOF
```

**Note :** Le `'EOF'` (avec guillemets) empêche l'interprétation des variables bash.

Tester la configuration avant de redémarrer :
```bash
sudo rsyslogd -N1
```
(Doit retourner "config validation run" sans erreur)

Redémarrer rsyslog :
```bash
sudo systemctl restart rsyslog
```

Vérifier le statut :
```bash
sudo systemctl status rsyslog | head -10
```
(Pas d'erreur "error during parsing file")

Voir les logs pfSense isolés :
```bash
sudo tail -f /var/log/pfsense.log
```

#### Debug si le fichier n'apparaît pas

Si `tail -f /var/log/pfsense.log` retourne "No such file or directory" :

**1. Vérifier que imudp est bien activé :**
```bash
sudo grep -r "imudp" /etc/rsyslog.conf /etc/rsyslog.d/
```
Si les lignes sont commentées (`#`), les activer :
```bash
sudo sed -i 's/^#module(load="imudp")/module(load="imudp")/' /etc/rsyslog.conf
sudo sed -i 's/^#input(type="imudp" port="514")/input(type="imudp" port="514")/' /etc/rsyslog.conf
sudo systemctl restart rsyslog
```

**2. Vérifier que le port 514 écoute :**
```bash
sudo ss -tulpn | grep 514
```
Normalement : `udp   UNCONN  0  0  0.0.0.0:514  0.0.0.0:*   users:(("rsyslogd",pid=...,fd=...))`

**3. Tester la réception avec tcpdump :**
```bash
sudo tcpdump -i any port 514 -c 5
```
Si des paquets UDP 514 apparaissent → les logs arrivent bien du réseau.

**4. Voir les logs dans syslog (même si fichier dédié pas créé) :**
```bash
sudo grep -r "192.168.1.1" /var/log/syslog | tail -5
```

**5. Tester l'envoi depuis pfSense (SSH) :**
```bash
logger -h 192.168.2.20 -P 514 "test log pfsense"
```
(FreeBSD utilise `-h` pas `-n`)

**6. Si la règle rsyslog ne crée pas le fichier :**
Remplacer par une règle plus large :
```bash
sudo tee /etc/rsyslog.d/10-pfsense.conf > /dev/null <<'EOF'
if $fromhost startswith "192.168." then /var/log/pfsense.log
& stop
EOF
```

**7. Forcer la création du fichier :**
```bash
sudo touch /var/log/pfsense.log
sudo chown syslog:adm /var/log/pfsense.log
sudo systemctl restart rsyslog
```

### Problème : le fichier pfsense.log existe mais reste vide

**Cause :** L'IP source des logs reconnue par rsyslog n'est pas `192.168.1.1` mais `_gateway` (résolution DNS inverse).

**Petite astuce :** Voir l'IP source réelle des logs reçus :
```bash
sudo head -50 /var/log/syslog | grep -i "gateway"
```

**Solutions :**

**A — Filtrer par hostname au lieu d'IP :**
```bash
sudo tee /etc/rsyslog.d/10-pfsense.conf > /dev/null <<'EOF'
if $fromhost == "_gateway" then /var/log/pfsense.log
& stop
EOF
```

**B — Sans filtre (tous les logs syslog dans le fichier) — pour debug :**
```bash
sudo tee /etc/rsyslog.d/10-pfsense.conf > /dev/null <<'EOF'
*.* /var/log/pfsense.log
EOF
```

**C — Désactiver la résolution DNS inverse sur rsyslog (pour retrouver l'IP) :**
```bash
echo 'global(localHostname="admin-samba")' | sudo tee -a /etc/rsyslog.d/10-pfsense.conf
sudo systemctl restart rsyslog
```

Après chaque changement, redémarrer et tester :
```bash
sudo systemctl restart rsyslog
sudo tail -f /var/log/pfsense.log
```
Sur pfSense (SSH) :
```bash
logger -h 192.168.2.20 -P 514 "test pfsense"
```
Quand le fichier se remplit, ajuster le filtre pour ne garder que les logs souhaités.

### Logs disponibles directement sur pfSense

Les logs sont accessibles dans l'interface web pfSense → **Status → System Logs** :

| Log | Chemin sur pfSense | Utilisation |
|-----|-------------------|-------------|
| Firewall | `/var/log/filter.log` | Trafic réseau |
| DHCP | `/var/log/dhcpd.log` | Baux IP |
| Authentication | `/var/log/auth.log` | Connexions admin |
| FreeRADIUS | `/var/log/radius.log` + `/var/log/radacct/` | Sessions utilisateurs |
| Captive Portal | `/var/log/portalauth.log` | Connexions portail |
| Squid | `/var/squid/logs/access.log` | Sites visités |
| Suricata | `/var/log/suricata/` | Alertes IDS/IPS |
| Resolver | `/var/log/resolver.log` | Requêtes DNS |

**Autres logs utiles :**
- **Services → FreeRADIUS → Accounting** → historique des sessions avec login, IP, durée, volume
- **Status → Captive Portal** → utilisateurs connectés en temps réel
- **Diagnostics → Command** → `tail -f /var/log/filter.log`

---

### Grafana (supervision des logs)

Sur l'Ubuntu DMZ, après installation Grafana + InfluxDB ou Prometheus :

```bash
# Installation rapide
sudo apt install grafana prometheus -y

# Dashboard : connexions, trafic, alertes
```

Visualiser :
- Nombre d'utilisateurs connectés
- Volume de données par utilisateur
- Sites les plus visités
- Alertes Suricata

---

### Supervision Graphique — Grafana + InfluxDB

**État : INACHEVÉ (27/06/2026)**
- Grafana + InfluxDB installés et tournent sur Ubuntu DMZ ✓
- SNMP activé sur pfSense et répond ✓
- Problèmes rencontrés : auth InfluxDB v1 non configurée, Telegraf/MIBs non résolus, script Python SNMP non finalisé
- **À reprendre :** configurer l'auth V1 dans InfluxDB, connecter une source de données SNMP, créer le dashboard

**Pourquoi cette stack ?**
- **Grafana** : interface web de dashboards (graphiques, jauges, alertes). Permet de visualiser CPU, RAM, débit réseau, connexions actives en temps réel.
- **InfluxDB** : base de données time-series qui stocke les métriques dans le temps. Grafana lit dedans.
- **Telegraf** : agent installé sur pfSense qui collecte les stats (CPU, RAM, disque, interfaces réseau) et les envoie dans InfluxDB.

**Ce que ça apporte au projet IFAG :**
- Répond aux exigences Loi 18-07 algérienne : « mise en place d'un système de supervision et d'analyse des journaux »
- Visualisation en temps réel sans se connecter à pfSense
- Dashboard pro pour la soutenance (graphiques dynamiques)
- Détection d'anomalies (pic de trafic, CPU à 100%, etc.)

**Architecture :**
```
pfSense ──telegraf──▶ InfluxDB ──▶ Grafana
   (métriques)      (192.168.2.20:8086)  (192.168.2.20:3000)
```

**Étapes d'installation :**

1. **Installer Docker sur Ubuntu DMZ** (conteneur pour Grafana + InfluxDB)
   ```bash
   sudo apt update && sudo apt install -y docker.io docker-compose-v2
   sudo usermod -aG docker $USER
   # déconnexion/reconnexion
   ```

2. **Créer le docker-compose.yml** dans `/opt/monitoring/` :
   ```yaml
   version: '3'
   services:
     influxdb:
       image: influxdb:2
       container_name: influxdb
       restart: unless-stopped
       ports:
         - "8086:8086"
       volumes:
         - influxdb-data:/var/lib/influxdb2

     grafana:
       image: grafana/grafana
       container_name: grafana
       restart: unless-stopped
       ports:
         - "3000:3000"
       volumes:
         - grafana-data:/var/lib/grafana
       depends_on:
         - influxdb

   volumes:
     influxdb-data:
     grafana-data:
   ```

3. **Lancer les conteneurs :**
   ```bash
   cd /opt/monitoring && sudo docker compose up -d
   ```

4. **Configurer InfluxDB (interface web + CLI) :**
   - Accéder à http://192.168.2.20:8086
   - Créer un compte admin (username: admin, password: admin123@, org: ifag, bucket: pfsense)
   - **Important :** le package Telegraf de pfSense utilise l'API **InfluxDB v1**, donc il faut créer un token V1.
   - **Option A :** via l'interface web → Data → Tokens → Generate → All Access Token, copier le token
   - **Option B :** via CLI (à lancer sur Ubuntu DMZ) :
     ```bash
     # Créer un token V1
     sudo docker exec influxdb influx v1 auth create \
       --org ifag \
       --username admin \
       --password admin123@ \
       --read-bucket pfsense \
       --write-bucket pfsense
     ```
   - Le token V1 généré ressemble à : `admin:admin123@` (ou une chaîne longue). Copier ce token, on en aura besoin.

   **Test rapide que InfluxDB répond :**
   ```bash
   curl -XPOST http://192.168.2.20:8086/write?db=pfsense -u admin:admin123@ -d 'test,host=test value=1'
   ```

5. **Installer Telegraf sur pfSense :**
   - System → Package Manager → Available Packages → chercher `telegraf` → Install
   - Configurer dans Services → Telegraf :
     - **Update Interval :** `5` secondes
     - **InfluxDB Output :** coché
     - **InfluxDB Server :** `http://192.168.2.20:8086`
     - **InfluxDB Database :** `pfsense`
     - **InfluxDB Username :** `admin`
     - **InfluxDB Password :** (token V1 généré à l'étape 4 — coller ici)
     - **Short Hostname :** coché
     - **Inputs :** Enable CPU Monitor, Enable Disk Monitor, Enable Memory Monitor, Enable Netstat Monitor (cocher ceux voulus)
   - Save & Apply

6. **Configurer Grafana :**
   - Accéder à http://192.168.2.20:3000 (admin/admin, changer au premier login)
   - Configuration → Data Sources → Add → InfluxDB
   - **Query Language :** `InfluxQL` (important, pas Flux — car on utilise l'API V1)
   - **URL :** http://192.168.2.20:8086
   - **Database :** `pfsense`
   - **User :** `admin`
   - **Password :** (token V1 ou admin123@)
   - **HTTP Method :** POST
   - Save & Test

7. **Importer un dashboard :**
   - Create → Import
   - ID 15761 (dashboard Telegraf simple)
   - Sélectionner la datasource InfluxDB

**Temps estimé : ~1h**
- Installation Docker : 5 min
- Déploiement Grafana + InfluxDB : 5 min
- Configuration InfluxDB : 5 min
- Installation Telegraf pfSense : 20 min (téléchargement package)
- Configuration Telegraf : 10 min
- Configuration Grafana + dashboard : 10 min
- Debug éventuel : variable

### ~~~ Approche finale : Telegraf Docker (recommandée) ~~~

**Pourquoi cette approche ?** Le package Telegraf pfSense ne se lance pas → on installe Telegraf **en Docker** sur l'Ubuntu DMZ. Il interroge pfSense via SNMP et envoie direct dans InfluxDB. Plus simple que Prometheus + snmp-exporter.

**Architecture :**
```
pfSense (SNMP) ──▶ Telegraf (Docker) ──▶ InfluxDB ──▶ Grafana
                    Ubuntu 2.20          Ubuntu 2.20   Ubuntu 2.20
```

**Prérequis :** SNMP doit être activé sur pfSense (étape 1 ci-dessous).

**Étapes :**

**1. Activer SNMP sur pfSense (si pas déjà fait) :**
- System → Package Manager → Available Packages → chercher `snmp` → Install (intégré)
- Services → SNMP → Enable SNMP Server :
  - Location : `IFAG`
  - Contact : `admin@ifag.local`
  - Community : `public`
  - Interface : décocher tout (bind all) pour que la DMZ puisse requêter
- Save & Apply
- Status → Services → `bsnmpd` doit être démarré ✅

**2. Sur Ubuntu DMZ, remplacer Prometheus/snmp-exporter par Telegraf :**
```bash
cd /opt/monitoring

# Mettre à jour docker-compose.yml (on remplace prometheus + snmp-exporter par telegraf)
sudo tee docker-compose.yml <<'EOF'
services:
  influxdb:
    image: influxdb:2
    container_name: influxdb
    restart: unless-stopped
    ports:
      - "8086:8086"
    volumes:
      - influxdb-data:/var/lib/influxdb2

  grafana:
    image: grafana/grafana
    container_name: grafana
    restart: unless-stopped
    ports:
      - "3000:3000"
    volumes:
      - grafana-data:/var/lib/grafana

  telegraf:
    image: telegraf
    container_name: telegraf
    restart: unless-stopped
    volumes:
      - ./telegraf.conf:/etc/telegraf/telegraf.conf:ro
    depends_on:
      - influxdb

volumes:
  influxdb-data:
  grafana-data:
EOF

# Créer telegraf.conf (SNMP input → InfluxDB output)
sudo tee telegraf.conf <<'EOF'
[agent]
  interval = "30s"
  round_interval = true
  metric_batch_size = 1000
  metric_buffer_limit = 10000
  collection_jitter = "0s"
  flush_interval = "10s"
  flush_jitter = "0s"
  precision = ""
  debug = false
  quiet = false
  logfile = ""
  hostname = ""
  omit_hostname = false

[[outputs.influxdb_v2]]
  urls = ["http://influxdb:8086"]
  token = "admin123@"
  organization = "ifag"
  bucket = "pfsense"

[[inputs.snmp]]
  agents = ["192.168.2.1"]
  version = 2
  community = "public"
  timeout = "5s"
  retries = 3

  # Interfaces (traffic réseau)
  [[inputs.snmp.field]]
    name = "hostname"
    oid = ".1.0.0.1.5"
    is_tag = true

  [[inputs.snmp.table]]
    name = "interface"
    inherit_tags = ["hostname"]
    oid = ".1.3.6.1.2.1.2.2"
    [[inputs.snmp.table.field]]
      name = "ifIndex"
      oid = ".1.3.6.1.2.1.2.2.1.1"
      is_tag = true
    [[inputs.snmp.table.field]]
      name = "ifName"
      oid = ".1.3.6.1.2.1.2.2.1.2"
      is_tag = true
    [[inputs.snmp.table.field]]
      name = "ifDescr"
      oid = ".1.3.6.1.2.1.2.2.1.2"
    [[inputs.snmp.table.field]]
      name = "ifInOctets"
      oid = ".1.3.6.1.2.1.2.2.1.10"
    [[inputs.snmp.table.field]]
      name = "ifInUcastPkts"
      oid = ".1.3.6.1.2.1.2.2.1.11"
    [[inputs.snmp.table.field]]
      name = "ifInErrors"
      oid = ".1.3.6.1.2.1.2.2.1.14"
    [[inputs.snmp.table.field]]
      name = "ifOutOctets"
      oid = ".1.3.6.1.2.1.2.2.1.16"
    [[inputs.snmp.table.field]]
      name = "ifOutUcastPkts"
      oid = ".1.3.6.1.2.1.2.2.1.17"
    [[inputs.snmp.table.field]]
      name = "ifOutErrors"
      oid = ".1.3.6.1.2.1.2.2.1.20"
    [[inputs.snmp.table.field]]
      name = "ifAdminStatus"
      oid = ".1.3.6.1.2.1.2.2.1.7"
    [[inputs.snmp.table.field]]
      name = "ifOperStatus"
      oid = ".1.3.6.1.2.1.2.2.1.8"
EOF
```

**3. Relancer Docker :**
```bash
cd /opt/monitoring && sudo docker compose up -d
sudo docker ps
```

**4. Vérifier que Telegraf envoie bien les données dans InfluxDB :**
```bash
# Voir les logs Telegraf
sudo docker logs telegraf --tail 20

# Tester si les données arrivent dans InfluxDB
curl -s -G "http://192.168.2.20:8086/query?db=pfsense" -u admin:admin123@ --data-urlencode "q=SHOW MEASUREMENTS" | head -20
```

**5. Configurer Grafana :**
- http://192.168.2.20:3000 (admin/admin)
- Configuration → Data Sources → Add → InfluxDB
  - Query Language : `InfluxQL`
  - URL : `http://192.168.2.20:8086`
  - Database : `pfsense`
  - User : `admin`
  - Password : `admin123@`
  - HTTP Method : `POST`
- Save & Test ✅

**6. Importer un dashboard :**
- Create → Import → ID `15761` (dashboard Telegraf simple) ou `12259` (Interface Traffic by SNMP)
- Sélectionner la datasource InfluxDB

**Temps estimé : ~20 min**

---

### Ce qu'il faut retenir pour le PFE

**Dans le mémoire :**
- Section dédiée à "Conformité légale (Loi 18-07)"
- Expliquer que le système enregistre :
  - Identité (login Samba AD)
  - Horodatage (connexion/déconnexion)
  - Volume (RADIUS Accounting)
  - Sites visités (Squid logs)
- Durée de conservation : configurable (12 mois recommandé)

**Pour la soutenance :**
- Montrer **Status → Captive Portal** (sessions actives)
- Montrer **Status → System Logs** (logs firewall)
- Montrer la page FreeRADIUS Accounting (historique connexions)

---

### Commandes utiles

```bash
# Voir les logs en temps réel sur pfSense
tail -f /var/log/radiusd.log

# Voir les logs RADIUS Accounting
tail -f /var/log/radacct/*

# Voir les logs firewall
tail -f /var/log/filter.log

# Voir les logs Squid (sites visités)
tail -f /var/squid/logs/access.log

# Voir les logs portail captif
tail -f /var/log/captiveportal.log

# Voir les logs sur Ubuntu DMZ (si centralisé)
sudo tail -f /var/log/syslog | grep pfeuser
```

---

### Commandes supervision Grafana

```bash
# Tester SNMP (depuis Ubuntu DMZ)
sudo apt install -y snmp
snmpwalk -v 2c -c public 192.168.2.1 1.3.6.1.2.1.1 | head -10

# Mettre à jour docker-compose + telegraf.conf
cd /opt/monitoring

# Écraser docker-compose.yml (influxdb + grafana + telegraf)
sudo tee docker-compose.yml <<'EOF'
services:
  influxdb:
    image: influxdb:2
    container_name: influxdb
    restart: unless-stopped
    ports:
      - "8086:8086"
    volumes:
      - influxdb-data:/var/lib/influxdb2

  grafana:
    image: grafana/grafana
    container_name: grafana
    restart: unless-stopped
    ports:
      - "3000:3000"
    volumes:
      - grafana-data:/var/lib/grafana

  telegraf:
    image: telegraf
    container_name: telegraf
    restart: unless-stopped
    volumes:
      - ./telegraf.conf:/etc/telegraf/telegraf.conf:ro
    depends_on:
      - influxdb

volumes:
  influxdb-data:
  grafana-data:
EOF

# Créer telegraf.conf
sudo tee telegraf.conf <<'EOF'
[agent]
  interval = "30s"
  round_interval = true
  metric_batch_size = 1000
  metric_buffer_limit = 10000
  collection_jitter = "0s"
  flush_interval = "10s"
  flush_jitter = "0s"
  precision = ""
  debug = false
  quiet = false
  logfile = ""
  hostname = ""
  omit_hostname = false

[[outputs.influxdb_v2]]
  urls = ["http://influxdb:8086"]
  token = "admin123@"
  organization = "ifag"
  bucket = "pfsense"

[[inputs.snmp]]
  agents = ["192.168.2.1"]
  version = 2
  community = "public"
  timeout = "5s"
  retries = 3

  [[inputs.snmp.field]]
    name = "hostname"
    oid = ".1.0.0.1.5"
    is_tag = true

  [[inputs.snmp.table]]
    name = "interface"
    inherit_tags = ["hostname"]
    oid = ".1.3.6.1.2.1.2.2"
    [[inputs.snmp.table.field]]
      name = "ifIndex"
      oid = ".1.3.6.1.2.1.2.2.1.1"
      is_tag = true
    [[inputs.snmp.table.field]]
      name = "ifName"
      oid = ".1.3.6.1.2.1.2.2.1.2"
      is_tag = true
    [[inputs.snmp.table.field]]
      name = "ifDescr"
      oid = ".1.3.6.1.2.1.2.2.1.2"
    [[inputs.snmp.table.field]]
      name = "ifInOctets"
      oid = ".1.3.6.1.2.1.2.2.1.10"
    [[inputs.snmp.table.field]]
      name = "ifInUcastPkts"
      oid = ".1.3.6.1.2.1.2.2.1.11"
    [[inputs.snmp.table.field]]
      name = "ifInErrors"
      oid = ".1.3.6.1.2.1.2.2.1.14"
    [[inputs.snmp.table.field]]
      name = "ifOutOctets"
      oid = ".1.3.6.1.2.1.2.2.1.16"
    [[inputs.snmp.table.field]]
      name = "ifOutUcastPkts"
      oid = ".1.3.6.1.2.1.2.2.1.17"
    [[inputs.snmp.table.field]]
      name = "ifOutErrors"
      oid = ".1.3.6.1.2.1.2.2.1.20"
    [[inputs.snmp.table.field]]
      name = "ifAdminStatus"
      oid = ".1.3.6.1.2.1.2.2.1.7"
    [[inputs.snmp.table.field]]
      name = "ifOperStatus"
      oid = ".1.3.6.1.2.1.2.2.1.8"
EOF

# Relancer Docker
sudo docker compose up -d

# Voir les logs Telegraf
sudo docker logs telegraf --tail 20

# Installer les MIBs SNMP dans le conteneur Telegraf (si erreur "Unknown Object Identifier")
sudo docker exec -u root telegraf bash -c "apt-get update -qq && apt-get install -y -qq snmp-mibs-downloader 2>/dev/null && download-mibs 2>/dev/null"
sudo docker compose restart telegraf
sudo docker logs telegraf --tail 10

# Si le conteneur crash en boucle → créer une image custom avec MIBs
sudo docker compose stop telegraf
sudo docker rm -f telegraf 2>/dev/null
sudo tee /opt/monitoring/Dockerfile <<'EOF'
FROM telegraf:latest
RUN apt-get update && apt-get install -y snmp-mibs-downloader && download-mibs || true
EOF
sudo docker build -t telegraf-mibs /opt/monitoring/
sudo sed -i 's|image: telegraf|image: telegraf-mibs|' /opt/monitoring/docker-compose.yml
sudo docker compose up -d telegraf
sudo docker logs telegraf --tail 15

# Vérifier si les MIBs sont dans l'image custom
sudo docker run --rm telegraf-mibs ls /usr/share/snmp/mibs/ 2>/dev/null
sudo docker run --rm telegraf-mibs cat /etc/snmp/snmp.conf 2>/dev/null || echo "NO CONF"

# Reconstruire l'image custom avec MIBs forcées + snmp.conf corrigé
sudo tee /opt/monitoring/Dockerfile <<'EOF'
FROM telegraf:latest
RUN apt-get update && apt-get install -y -qq curl libsnmp-dev && \
    curl -sfL -o /usr/share/snmp/mibs/IF-MIB.txt \
      "https://raw.githubusercontent.com/vincentbernat/snmp-mibs-downloader/refs/heads/master/MIBs/IF-MIB.txt" && \
    curl -sfL -o /usr/share/snmp/mibs/RFC1213-MIB.txt \
      "https://raw.githubusercontent.com/vincentbernat/snmp-mibs-downloader/refs/heads/master/MIBs/RFC1213-MIB.txt" && \
    sed -i 's/mibs :/mibs ALL/' /etc/snmp/snmp.conf && \
    echo "mibdirs /usr/share/snmp/mibs" >> /etc/snmp/snmp.conf
EOF
sudo docker build -t telegraf-mibs /opt/monitoring/
sudo docker compose up -d telegraf
sudo docker logs telegraf --tail 15

# Alternative : utiliser des champs individuels au lieu de table (si MIBs ne marchent pas)

# Voir les mesures dans InfluxDB
---

### Alternative rapide : Script Python SNMP → InfluxDB

**Quand l'utiliser :** si Telegraf Docker refuse de coopérer avec les MIBs. Script maison qui snmpwalk pfSense et écrit direct dans InfluxDB. Pas de MIBs, pas de conteneurs supplémentaires.

**Installation :**
```bash
sudo apt install -y python3-requests
```

**Créer le script :**
```bash
sudo tee /opt/monitoring/pfsense-snmp.py <<'PYEOF'
#!/usr/bin/env python3
import subprocess, time, requests
from base64 import b64encode

AUTH = b64encode(b"admin:admin123@").decode()

def get_interface_names():
    r = subprocess.run(["snmpwalk", "-v2c", "-c", "public", "-On", "-Oqv",
                        "192.168.2.1", ".1.3.6.1.2.1.2.2.1.2"],
                       capture_output=True, text=True, timeout=15)
    ifaces = {}
    for i, name in enumerate(r.stdout.strip().split('\n'), 1):
        if name:
            ifaces[i] = name.strip('"')
    return ifaces

def get_stats(oid):
    r = subprocess.run(["snmpwalk", "-v2c", "-c", "public", "-On", "-Oqv",
                        "192.168.2.1", oid],
                       capture_output=True, text=True, timeout=15)
    vals = {}
    for i, v in enumerate(r.stdout.strip().split('\n'), 1):
        if v and v.strip():
            try:
                vals[i] = int(v.strip().split()[0])
            except:
                pass
    return vals

ifaces = get_interface_names()
print(f"Found interfaces: {ifaces}")

while True:
    ts = int(time.time() * 1e9)
    lines = []
    rx = get_stats(".1.3.6.1.2.1.2.2.1.10")
    tx = get_stats(".1.3.6.1.2.1.2.2.1.16")
    for idx, name in ifaces.items():
        if idx in rx and idx in tx:
            lines.append(f"interface,ifName={name},host=pfsense "
                         f"ifInOctets={rx[idx]},ifOutOctets={tx[idx]} {ts}")
    if lines:
        data = '\n'.join(lines)
        try:
            r = requests.post("http://192.168.2.20:8086/write?db=pfsense",
                            data=data,
                            headers={"Authorization": f"Basic {AUTH}",
                                    "Content-Type": "text/plain"},
                            timeout=5)
            print(f"Wrote {len(lines)} metrics ({r.status_code})")
        except Exception as e:
            print(f"Error: {e}")
    time.sleep(30)
PYEOF
```

**Lancer le script :**
```bash
cd /opt/monitoring && python3 pfsense-snmp.py &
```

**Vérifier les données :**
```bash
curl -s -G "http://192.168.2.20:8086/query?db=pfsense" -u admin:admin123@ --data-urlencode "q=SELECT * FROM interface LIMIT 5"
```

**Debug SNMP (si interface vide) :**
```bash
# Tester le snmpwalk des noms d'interfaces
snmpwalk -v2c -c public -On -Oqv 192.168.2.1 .1.3.6.1.2.1.2.2.1.2

# Tester l'auth InfluxDB
curl -s -G "http://192.168.2.20:8086/query?db=pfsense" -u admin:admin123@ --data-urlencode "q=SHOW DATABASES"

**Dashboard Grafana (http://192.168.2.20:3000)** :
- Create → Dashboard → Add panel
- Query: `FROM interface` → `SELECT field(ifInOctets), field(ifOutOctets)` → `GROUP BY tag(ifName)`
- Visualisation : Time series
```

![[Pasted image 20260628110839.png]]


![[Pasted image 20260627133637.png]]

Logs Firewall
![[Pasted image 20260627134210.png]]

Logs du portail captif :
![[Pasted image 20260627134448.png]]

COnfiguration du logging de FreeRaDIUS 
![[Pasted image 20260627135427.png]]

![[Pasted image 20260627141123.png]]
![[Pasted image 20260627152925.png]]
![[Pasted image 20260627154555.png]]

![[Pasted image 20260627164205.png]]

token api : ```
LWOBFvSFDZKiLHkES9m-6gK68uF957KRZFdfgcoLDEQSFldJvu7TKumIgE_ht-tdeUkV4afNepU4Q9_NEJT3LQ==

