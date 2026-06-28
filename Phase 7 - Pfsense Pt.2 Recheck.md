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

| Donnée | Source | Exemple |
|--------|--------|---------|
| Connexions utilisateurs | FreeRADIUS (Accounting) | pfeuser → 192.168.1.102 → 30min → 150MB |
| Sites visités | Squid (proxy logs) | pfeuser → facebook.com → 10:32 |
| Traffic IP | pfSense (filter.log) | 192.168.1.102 → 8.8.8.8:53 |
| Alertes sécurité | Suricata (IDS logs) | Scan Nmap détecté depuis 192.168.1.102 |
| Sessions VPN | WireGuard | client VPN connecté 45min |
| Authentifications | FreeRADIUS (auth.log) | Access-Accept / Access-Reject |

---

### Vérification des logs existants sur pfSense

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

### Configuration de l'envoi des logs vers Ubuntu DMZ (recommandé)

Pour une vraie traçabilité, on centralise les logs sur le serveur Ubuntu (192.168.2.20).

#### Sur l'Ubuntu DMZ (192.168.2.20) — serveur syslog

```bash
sudo apt update && sudo apt install rsyslog -y
```

#### Configurer rsyslog pour recevoir les logs pfSense

Éditer `/etc/rsyslog.conf` :
```bash
sudo nano /etc/rsyslog.conf
```

Dé-commenter :
```
module(load="imudp")
input(type="imudp" port="514")
```

Redémarrer :
```bash
sudo systemctl restart rsyslog
sudo ufw allow 514/udp
```

#### Sur pfSense — envoyer les logs

- **Status → System Logs → Settings**
- **Enable Remote Syslog Logging** ✅
- **Remote Log Server 1 :** `192.168.2.20:514`
- **Syslog Contents :** tout cocher (Firewall, Captive Portal, FreeRADIUS, etc.)
- **Save → Apply**

#### Vérification sur Ubuntu

```bash
sudo tail -f /var/log/syslog | grep 192.168.1.
```

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

