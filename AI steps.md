## État du projet

### ✅ Terminé
- pfSense installé (Phase 5 pt1)
- VLAN DMZ configuré (Phase 5 pt2)
- Firewall rules de base (Phase 5 pt3)
- QoS/Traffic Shaping (Phase 5 pt3)
- Samba AD installé sur VM (192.168.2.20)
- FreeRADIUS installé sur pfSense (127.0.0.1)
- FreeRADIUS lié à Samba AD (LDAP + DEFAULT Auth-Type := LDAP)
- Portail captif configuré (Services → Captive Portal → zone LAN)
- Squid + SquidGuard (filtrage web, shallalist)
- DNS Forwarder activé
- DNS publics en secours : 8.8.8.8, 1.1.1.1
- **Portail captif fonctionnel** — authentification RADIUS via FreeRADIUS → Samba AD
- **Suricata (IDS/IPS)** — installé + offloading désactivé

### ❌ Firewall rules LAN (quand portail captif actif)

**ATTENTION :** NE PAS ajouter de règle "Pass Any → Any" sur le LAN — ça contourne le portail captif.

**Firewall → Rules → LAN → Add** (4 règles, dans l'ordre) :

| # | Action | Protocol | Source | Destination | Port | Desc |
|---|--------|----------|--------|-------------|------|------|
| 1 | Pass | TCP/UDP | LAN net | This Firewall | 53 | DNS |
| 2 | Pass | UDP | LAN net | This Firewall | 67-68 | DHCP |
| 3 | Pass | TCP | LAN net | This Firewall | 80,443 | Web Admin |
| 4 | Pass | ICMP | LAN net | This Firewall | any | Pings |

**Firewall → Rules → DMZ → Add** (si DMZ existe) :

| # | Action | Protocol | Source | Destination | Port | Desc |
|---|--------|----------|--------|-------------|------|------|
| 1 | Pass | Any | DMZ net | Any | any | DMZ Internet |

**Save → Apply Changes**

### Configuration portail captif avec HTTPS

**Services → Captive Portal → Edit (zone LAN) :**

- ✅ **Enable**
- **Interface :** LAN
- **Authentication → Method :** RADIUS Authentication
- **RADIUS Server 1 :** 127.0.0.1 / secret : admin123@
- **HTTPS login (Enable) :** ✅
- **HTTPS server name :** portal.ifag.local
- **SSL/TLS Certificate :** GUI default
- **HTTPS Forwards (Disable) :** ❌ PAS COCHÉ
- **Send RADIUS Accounting packets :** ✅ (pour logs/traçabilité)
- **Save → Apply**

Le portail captif écoute sur **port 8000** (HTTP) et interceptera les requêtes HTTP + HTTPS (443).

---

### Suricata (IDS/IPS)

**Prérequis obligatoire :** Désactiver les offloading dans **System → Advanced → Networking** :

| Option | État |
|--------|------|
| Hardware Checksum Offloading | ✅ **Disable** (coché) |
| Hardware TCP Segmentation Offloading | ✅ **Disable** (coché) |
| Hardware Large Receive Offloading | ✅ **Disable** (coché) |

*Redémarrage pfSense requis après modification.*

**Installation :** System → Package Manager → Available Packages → `suricata` → Install

**Configuration de base :**
- **Services → Suricata → Interfaces → Add** → Interface WAN → Save
- **Services → Suricata → Rules → Download** → cocher **ET Open** → Download
- **Services → Suricata → WAN → Enable** ✅ → Mode : **IDS** (détection seulement)
- **Save → Apply**

**Test :** Lancer un `nmap` depuis Kali → Voir l'alerte dans Suricata → Alerts

---

### En attente (Phase 7 — nouveau cahier des charges)

Ordre de priorité :

1. ✅ ~~Suricata (IDS/IPS)~~ — **Terminé**
2. ✅ ~~VPN WireGuard~~ — **Terminé**

### Qu'est-ce qu'un VPN ?

**VPN (Virtual Private Network)** = tunnel chiffré entre un client distant et le réseau IFAG.

**Utilité pour le projet :**

| Profil | Sans VPN | Avec VPN |
|--------|----------|----------|
| **Admin** (gérer le réseau à distance) | Impossible | ✅ SSH pfSense depuis chez lui |
| **Professeur** (accès fichiers école) | Impossible | ✅ Accès aux ressources |
| **Étudiant** (cours en ligne) | Impossible | ✅ Accès au réseau pédagogique |

**Sécurité :**
- Trafic **chiffré de bout en bout**
- Authentification possible via FreeRADIUS
- Traçabilité : logs de connexion (loi algérienne)

### Comparaison VPN

| Critère | **WireGuard** | **OpenVPN** | **IPsec** |
|---------|---------------|-------------|-----------|
| **Vitesse** | 🚀 Très rapide | 🐢 Plus lent | ⚡ Rapide |
| **Configuration** | ✅ Très simple | 😤 Complexe | 😤 Complexe |
| **Performance** | Excellent (kernel) | Bon (userspace) | Excellent |
| **Mobile** | ✅ Excellent | ✅ Bon | ✅ Bon |
| **Popularité** | Moderne, tendance | Très répandu | Standard entreprises |
| **Pour le PFE** | **Simple à démontrer** | Classique mais lourd | Adapté si exigé |

**Recommandation :** WireGuard — rapide, simple, moderne.

### Installation WireGuard sur pfSense

**1. Installer le package :**
System → Package Manager → Available Packages → `wireguard` → Install

**2. Créer le tunnel :**
VPN → WireGuard → Tunnels → Add Tunnel :
- Description : `VPN_IFAG`
- Listen Port : `51820` (UDP)
- Interface Keys : **Generate**
- Save

**3. Ajouter un Peer (client) :**
VPN → WireGuard → Tunnels → Edit → Peers → Add Peer :
- Public Key : (à récupérer depuis le client)
- Allowed IPs : `192.168.1.0/24, 192.168.2.0/24`
- Save

**4. Assigner l'interface WireGuard :**
Interfaces → Assignments → Add → WireGuard (tun_wg0) → Save
- Enable ✅
- IPv4 Configuration : Static
- IPv4 Address : `10.0.0.1/24`
- Save → Apply

**5. Règles Firewall :**
**Firewall → Rules → WireGuard (VPN) → Add :**
- Action : Pass / Protocol : Any / Source : Any / Dest : Any
- Desc : "Allow VPN traffic"

**Firewall → Rules → WAN → Add :**
- Action : Pass / Protocol : UDP / Dest : This Firewall (port 51820)
- Desc : "Allow WireGuard WAN"

**6. Client Windows :**
- Télécharger WireGuard sur `wireguard.com/install/`
- Créer une interface avec :
  - Private Key : (générée par le client)
  - Public Key : à copier dans le Peer sur pfSense
  - Endpoint : IP publique pfSense:51820
  - Allowed IPs : `192.168.1.0/24, 192.168.2.0/24, 10.0.0.0/24`

**7. Test :**
- Client connecté → `ping 192.168.1.1` → OK
- `ping 192.168.2.20` (Samba) → OK

3. **Traçabilité** — logs → syslog → serveur distant
4. **Supervision (Grafana/ntopng)** — Package ntopng + Grafana
5. **Google Auth 2FA** — FreeRADIUS + Google Authenticator
6. **Ansible** — Automatisation depuis Ubuntu DMZ
7. **Fail2ban** — Sécurité supplémentaire
8. **Zero Trust** — Segmentation fine + micro-authentification

---

### Notes FreeRADIUS + Samba AD

Fix appliqué pour l'authentification LDAP :

Fichier `/usr/local/etc/raddb/users` créé avec :
```
DEFAULT Auth-Type := LDAP
```

Après redémarrage du service FreeRADIUS, tests OK (`radtest` → Access-Accept).

---

### Problèmes résolus

1. **Ping Windows → pfSense** : règle ICMP dans Firewall → Rules → LAN
2. **FreeRADIUS → Samba AD** : LDAP bind fonctionnel, `DEFAULT Auth-Type := LDAP` dans `/usr/local/etc/raddb/users`
3. **DNS** : DNS Forwarder activé (Services → DNS Forwarder)
4. **Portail captif** : règles LAN = DNS, DHCP, Web Admin, ICMP seulement (PAS de Pass Any Any)
5. **Le portail captif doit être sur interface LAN uniquement** (pas WAN/DMZ)

### Commandes Samba (sur VM Samba 192.168.2.20)
```bash
# Voir tous les utilisateurs Samba AD
samba-tool user list

# Voir tous les groupes
samba-tool group list

# Détails d'un utilisateur
samba-tool user show pfeuser

# Voir les groupes d'un utilisateur
samba-tool user listgroups pfeuser

# Voir les membres d'un groupe
samba-tool group listmembers Etudiants

# Créer un utilisateur
samba-tool user create username password

# Supprimer un utilisateur
samba-tool user delete username
```

### Commandes utiles (SSH pfSense)

```bash
# Test authentification RADIUS
radtest pfeuser "P@ssword123" 127.0.0.1 0 "admin123@"

# Redémarrer FreeRADIUS en mode debug
killall -9 radiusd && radiusd -X

# Voir les logs RADIUS
tail -f /var/log/radiusd.log

# Voir le portail captif
sockstat -4l | grep 8000

# Voir les règles firewall chargées
pfctl -s rules

# Voir les processus
ps aux | grep -E "lighttpd|captive|unbound|radius"

# Suricata — logs d'alertes
tail -f /var/log/suricata/suricata.log

# Suricata — alertes visibles aussi dans
cat /var/log/suricata/stats.log | tail
```
