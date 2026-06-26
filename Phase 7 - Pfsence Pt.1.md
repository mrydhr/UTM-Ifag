## Phase 7 — Évolution du projet UTM IFAG

Après avoir fait le point en réunion, le cahier des charges a évolué : un membre en moins, plus de charges, et de nouvelles fonctionnalités à ajouter.

---

## Nouvelles fonctionnalités demandées

- **Traçabilité** — requise par la loi algérienne
- **Sécurité (IDS/IPS)** — Suricata
- **Automatisation (Ansible)**
- **Supervision (Grafana)**
- **Zero Trust**
- **VPN (WireGuard)**
- **Google Auth 2FA** — jointure avec FreeRADIUS
- **Fail2ban**

---

## Problème rencontré au lancement des machines

Lors de la restauration de l'environnement FreeRADIUS/Samba AD, les utilisateurs Active Directory étaient trouvés via LDAP, mais l'authentification échouait.

**Log FreeRADIUS :**
```
ldap: User object found
ldap: WARNING: PAP authentication to Active Directory MUST set 'Auth-Type := LDAP'
ERROR: No Auth-Type found
```

### Analyse
- ✅ connectivité réseau pfSense → Samba AD
- ✅ authentification du compte de service LDAP
- ✅ recherche correcte des utilisateurs (filtre `sAMAccountName`)
- ✅ configuration LDAP valide dans `mods-enabled/ldap`
- ❌ Absence d'une règle définissant explicitement le type d'authentification LDAP

### Correction appliquée

Fichier créé : `/usr/local/etc/raddb/users`
```
DEFAULT Auth-Type := LDAP
```

Après redémarrage du service FreeRADIUS :
```
radtest pfeuser "P@ssword123" 127.0.0.1 0 "admin123@"
→ Access-Accept
```

### Explication technique

Samba AD (comme Microsoft AD) ne fournit pas les mots de passe en clair via LDAP. FreeRADIUS ne peut donc pas comparer localement le mot de passe reçu.

La directive `DEFAULT Auth-Type := LDAP` force FreeRADIUS à effectuer une authentification par **bind LDAP** directement sur le contrôleur de domaine pour valider le mot de passe.

---

## Portail captif

Au redémarrage, problème d'accès à l'interface admin (redirigé vers le portail général). Avec les soucis LDAP + portail actif, impossible d'accéder à l'admin.

### Solutions appliquées
1. IP de l'admin ajoutée dans les **Allowed IP Addresses** du portail captif → contourne l'authentification pour l'admin
![[Pasted image 20260625231715.png]]



### Configuration finale du portail

**Services → Captive Portal → Edit (zone LAN) :**
- ✅ Enable
- Interface : LAN uniquement
- Authentication : RADIUS Authentication
- Serveur RADIUS : 127.0.0.1 / secret : admin123@
- ✅ Send RADIUS accounting packets → traçabilité
- ✅ Enable HTTPS login → protection du formulaire de login
- HTTPS server name : portal.ifag.local
- Certificat : GUI default
- ❌ Disable HTTPS Forwards : PAS COCHÉ
- Save → Apply

Logo IFAG intégré dans la page de login du portail captif.
Redirection vers le portail → fonctionnelle.

![[Pasted image 20260626134226.png]]

![[Pasted image 20260626135404.png]]

---
## Reconfiguration du firewall

**Firewall → Rules → LAN** (4 règles, PAS de "Pass Any Any" qui contourne le portail) :

| #   | Action | Protocol | Source  | Destination   | Port   | Desc      |
| --- | ------ | -------- | ------- | ------------- | ------ | --------- |
| 1   | Pass   | TCP/UDP  | LAN net | This Firewall | 53     | DNS       |
| 2   | Pass   | UDP      | LAN net | This Firewall | 67-68  | DHCP      |
| 3   | Pass   | TCP      | LAN net | This Firewall | 80,443 | Web Admin |
| 4   | Pass   | ICMP     | LAN net | This Firewall | any    | Pings     |
| 5   | Pass   | Any      | LAN net | This Firewall | any    | Intrnet   |

**Firewall → Rules → DMZ :**

| # | Action | Protocol | Source | Destination | Port | Desc |
|---|--------|----------|--------|-------------|------|------|
| 1 | Pass | Any | DMZ net | Any | any | DMZ Internet |

**Save → Apply Changes**

---

## Suricata (IDS/IPS)

### Prérequis obligatoire
**System → Advanced → Networking** — décocher les 3 options :

| Option                               | État             |
| ------------------------------------ | ---------------- |
| Hardware Checksum Offloading         | Disable (coché)  |
| Hardware TCP Segmentation Offloading | Disable (coché)  |
| Hardware Large Receive Offloading    |  Disable (coché) |

*Redémarrage pfSense requis après modification.*

![[Pasted image 20260626171501.png]]
### Installation
**System → Package Manager → Available Packages → `suricata` → Install**

### Configuration de base
- **Services → Suricata → Interfaces → Add** → Interface WAN
- **Services → Suricata → Rules → Download** → **ET Open** → Download
- **Services → Suricata → WAN → Enable** ✅ → Mode : **IDS** (détection sans blocage)
- **Save → Apply**

---

## VPN WireGuard

### Installation
**System → Package Manager → Available Packages → `wireguard` → Install**

### Configuration du tunnel

**VPN → WireGuard → Tunnels → Add Tunnel :**
- Description : `VPN_IFAG`
- Listen Port : `51820` (UDP)
- Interface Keys : **Generate**

**Clé publique serveur :** `vm3fK3/wp7NVgFuZ+RUQex/6018M1m/onqO7jkqrHiE=`

![[Pasted image 20260626173745.png]]
### Client Windows

1. Installer WireGuard depuis `wireguard.com/install/`
2. **Add Tunnel → Add empty tunnel...**
3. Copier la **clé publique** du client

**Clé publique client :** `m26AnWkWkgAmdGtRg8wWBkgoyapjsCKTsI6llJYwCz4=`

![[Pasted image 20260626180840.png]]
### Ajout du Peer sur pfSense

**VPN → WireGuard → Tunnels → Edit → Peers → Add Peer :**
- Public Key : (clé publique du client Windows)
- Allowed IPs : `10.0.0.2/32`
- Save

![[Pasted image 20260626181204.png]]
### Assignation de l'interface

**Interfaces → Assignments → Add →** `tun_wg0` → Save

Puis configurer l'interface (OPTx) :
-  Enable
- IPv4 : Static → `10.0.0.1/24`
- Save → Apply

![[Pasted image 20260626183519.png]]
### Règles Firewall

**Firewall → Rules → OPT (VPN) → Add :**
- Pass / Any / Any / Any → "Allow VPN traffic"

**Firewall → Rules → WAN → Add :**
- Pass / UDP / This Firewall port 51820 → "Allow WireGuard WAN"

### Configuration client Windows

```ini
[Interface]
PrivateKey = (générée automatiquement)
Address = 10.0.0.2/32
DNS = 192.168.1.1

[Peer]
PublicKey = vm3fK3/wp7NVgFuZ+RUQex/6018M1m/onqO7jkqrHiE=
AllowedIPs = 192.168.1.0/24, 192.168.2.0/24, 10.0.0.0/24
Endpoint = 192.168.1.1:51820
```

![[Pasted image 20260626182021.png]]
### Test

**Depuis Windows (cmd) :**
```cmd
ping 10.0.0.1      → OK (ping pfSense via VPN)
ping 192.168.1.1   → OK (ping LAN pfSense)
```

![[Pasted image 20260626191839.png]]

![[Pasted image 20260626190536.png]]

**CONNEXION RÉUSSIE** — Handshake établi, pings fonctionnels.

---

## Prochaines étapes

1. **Traçabilité** — logs RADIUS + connexions
2. **Supervision (Grafana/ntopng)** — dashboards trafic
3. **Google Auth 2FA** — double authentification FreeRADIUS
4. **Fail2ban** — protection brute force
5. **Ansible** — automatisation
6. **Zero Trust** — segmentation fine
