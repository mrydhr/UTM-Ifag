## Phase 8 — Tests et Validation

### 1. Samba AD

**Objectif :** Vérifier que le contrôleur de domaine Samba AD fonctionne et que les utilisateurs peuvent s'authentifier.

**Commandes (sur Ubuntu DMZ 192.168.2.20) :**
```bash
# Lister tous les utilisateurs
samba-tool user list

# Détails d'un utilisateur
samba-tool user show pfeuser

# Voir les groupes
samba-tool group list

# Tester une connexion SMB
smbclient -L 192.168.2.20 -U pfeuser
```
**Résultat attendu :**
- `pfeuser` listé dans les utilisateurs
- Connexion SMB acceptée

**Capture :** 
![[Pasted image 20260628105259.png]]
![[Pasted image 20260628105359.png]]
![[Pasted image 20260628105455.png]]
![[Pasted image 20260628105923.png]]
![]()

---

### 2. FreeRADIUS

**Objectif :** Vérifier que l'authentification RADIUS fonctionne via Samba AD.

**Commandes (SSH pfSense) :**
```bash
# Test d'authentification
radtest pfeuser "P@ssword123" 127.0.0.1 0 "admin123@"
```

**Résultat attendu :**
```
Access-Accept
```

**Capture :** ![]()

---

### 3. Portail Captif

**Objectif :** Vérifier que le portail captif s'affiche et que l'authentification RADIUS fonctionne.

**Procédure :**
1. Ouvrir un navigateur depuis un client LAN (192.168.1.x)
2. Aller sur http://192.168.1.1:8000 (portail captif)
3. La page de connexion IFAG doit s'afficher
4. Saisir identifiant : `pfeuser` / mot de passe : `P@ssword123`
5. Cliquer sur Login

**Vérifications :**
- La page d'accueil personnalisée IFAG s'affiche après connexion
- **Status → Captive Portal → Active Sessions** : la session apparaît
- **Status → FreeRADIUS → Accounting** : l'historique s'affiche

**Capture :** 
![[Pasted image 20260427194445.png]]

---

### 4. Squid + SquidGuard (Filtrage Web)

**Objectif :** Vérifier que le proxy bloque les sites interdits.
**Capture :** ![]()
![[Pasted image 20260628155510.png]]
---

### 5. VPN WireGuard

**Objectif :** Vérifier que le client VPN peut se connecter au réseau IFAG.

**Procédure :**
1. Lancer le client WireGuard sur Windows
2. Activer le tunnel VPN_IFAG
3. Handshake doit s'établir (Status → Handshake OK)

**Tests :**
```powershell
# Depuis le client Windows connecté
ping 192.168.1.1
ping 192.168.2.20
```

**Vérifications :**
- Ping vers pfSense (192.168.1.1) ✅
- Ping vers Ubuntu DMZ (192.168.2.20) ✅
- Sur pfSense : **VPN → WireGuard** → statut du peer

**Capture :** 
![[Pasted image 20260626191839.png]]![]
![[Pasted image 20260626190536.png]]
---

### 6. Suricata (IDS/IPS)

**Objectif :** Vérifier que Suricata détecte les intrusions.

**Procédure :**
1. Depuis une machine Kali ou équivalent
2. Lancer un scan nmap vers l'IP WAN de pfSense
```bash
nmap -sS -A <IP_WAN_pfSense>
```

**Vérifications :**
- **Services → Suricata → WAN → Alerts** : des alertes apparaissent
- ```bash
  # Voir les logs Suricata
  cat /var/log/suricata/suricata.log | tail
  ```

**Résultat attendu :** Alertes type " ET SCAN Nmap Scripting Engine User-Agent" ou similaire.

**Capture :** ![]()
![[Pasted image 20260628161432.png]]
---

### 7. Firewall

**Objectif :** Vérifier les règles firewall.

**Vérifications :**

| Interface | Règles | Statut |
|-----------|--------|--------|
| **LAN** | DNS (53), DHCP (67-68), Web Admin (80,443), ICMP | ✅ |
| **DMZ** | Internet depuis DMZ (Any) | ✅ |
| **WAN** | WireGuard (51820 UDP), Anti-lockout (80,443) | ✅ |
| **WireGuard** | Pass Any Any | ✅ |

**Pas de règle "Pass Any Any" sur le LAN** (contournerait le portail captif).

**Capture :** 
![[Pasted image 20260626113824.png]]


---

### 8. Traçabilité (Logs centralisés)

**Objectif :** Vérifier que les logs pfSense arrivent sur Ubuntu DMZ.

**Vérifications (sur Ubuntu DMZ) :**
```bash
# Voir les logs pfSense en temps réel
sudo tail -f /var/log/pfsense.log

# Chercher une connexion spécifique
sudo grep pfeuser /var/log/pfsense.log

# Voir les logs syslog
sudo tail -f /var/log/syslog | grep pfSense
```

**Test depuis pfSense (SSH) :**
```bash
logger -h 192.168.2.20 -P 514 "test log pfsense"
```

**Résultat attendu :** Les logs apparaissent dans `/var/log/pfsense.log` sur Ubuntu DMZ.

**Capture :** ![]()
![[Pasted image 20260627134210.png]]
---

### 9. DNS Forwarder

**Objectif :** Vérifier que la résolution DNS fonctionne.

**Test depuis un client LAN :**
```bash
# Résolution DNS externe
nslookup google.com 192.168.1.1

# Résolution DNS locale (si configurée)
nslookup pfsense.ifag.local 192.168.1.1
```

**Résultat :** Réponse DNS valide.

---
### Récapitulatif des tests

| #   | Composant          | Statut |
| --- | ------------------ | ------ |
| 1   | Samba AD           | ✅      |
| 2   | FreeRADIUS         | ✅      |
| 3   | Portail Captif     | ✅      |
| 4   | Squid + SquidGuard | ✅      |
| 5   | VPN WireGuard      | ✅      |
| 6   | Suricata IDS/IPS   | ✅      |
| 7   | Firewall           | ✅      |
| 8   | Traçabilité        | ✅      |
| 9   | DNS Forwarder      | ✅      |
| 10  | Dashboard          | Avorté |
