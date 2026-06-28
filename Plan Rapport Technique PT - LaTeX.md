## Plan de rédaction — Rapport Technique PT (LaTeX)

**Projet :** Déploiement d'une passerelle UTM open source (pfSense) pour l'IFAG  
**Modèle :** Rapport Technique PT (20–30 pages)  
**Outil :** LaTeX (overleaf.com ou distribution locale)  
**Préambule :** Utiliser `report` ou `article`, package `graphicx`, `hyperref`, `listings`, `booktabs`, `geometry`, `fancyhdr`

---

### Préambule LaTeX

---
## Structure du rapport

### 1. Page de garde (1 page)

- Logos : Université d'Évry, CFA
- Titre : **Déploiement d'une passerelle UTM open source (pfSense) pour l'IFAG**
- Membres du groupe
- Commanditaire : IFAG
- Encadrant : Benaissa Adnane
- Date soutenance : Juin 2026

---

### 2. Introduction (2–3 pages)

**Sources :** Phase 1 — Initialisation du projet, Phase 2 — Analyse des besoins

**Contenu :**
- **Contexte :** L'IFAG, établissement de formation, dispose d'un réseau existant mais la sécurité est insuffisante face aux menaces modernes (malware, phishing, attaques réseau). Absence de gestion centralisée des utilisateurs, pas de traçabilité, pas de filtrage de contenu.
- **Problématique :** Comment déployer une solution de sécurité open source complète (UTM) capable d'assurer l'authentification centralisée, le filtrage web, la détection d'intrusion, la gestion de bande passante et la traçabilité des accès conformément à la Loi 18-07 algérienne ?
- **Objectifs :**
  - Déployer une passerelle firewall (pfSense) segmentant le réseau en WAN/LAN/DMZ
  - Mettre en place un contrôleur de domaine (Samba AD) pour la gestion centralisée des utilisateurs
  - Intégrer FreeRADIUS pour l'authentification réseau (portail captif, VPN)
  - Assurer le filtrage de contenu web (Squid + SquidGuard)
  - Déployer un système de détection d'intrusion (Suricata)
  - Mettre en place un VPN sécurisé (WireGuard)
  - Centraliser les logs sur un serveur distant (traçabilité légale)
- **Plan du rapport :** Annonce des sections

---

### 3. Cahier des charges (3–4 pages)

**Sources :** Phase 1 — Initialisation, Phase 2 — Analyse des besoins, Phase 7 Pt.1 (évolution du CDC)

```latex
\chapter{Cahier des charges}
```

#### 3.1 Phase initiale (Février 2026)
- Firewall + ACLs
- Authentification LDAP directe
- DMZ
- Portail captif
- QoS

#### 3.2 Rencontres avec le commanditaire
Le client a exprimé ses besoins lors de plusieurs entretiens :
- Visioconférence initiale (février 2026)
- Réunions de suivi (mars, avril, mai 2026)
- Évolution des exigences suite aux retours

#### 3.3 Phase d'évolution (Mai 2026)
Nouvelles exigences suite à la réunion :
- **Traçabilité** (Loi 18-07 algérienne)
- **IDS/IPS** (Suricata)
- **VPN** (WireGuard)
- Supervision (abandonnée — 3 approches tentées sans succès)
- 2FA, Fail2ban, Ansible, Zero Trust (perspectives)

#### 3.4 Spécifications fonctionnelles détaillées
- Segmentation réseau : WAN (Internet), LAN (utilisateurs), DMZ (services)
- Authentification : Samba AD + FreeRADIUS
- Filtrage web : Squid + SquidGuard (blacklist shallalist)
- Détection intrusion : Suricata (mode IDS)
- VPN : WireGuard (accès distant)
- Traçabilité : logs centralisés sur serveur Ubuntu DMZ

---

### 4. Stratégie retenue (6–7 pages)

**Sources :** Phase 4 — Planification, Phase 3 — Conception
#### 4.1 Choix technologiques

**Tableau comparatif des solutions :**

| Fonction         | Solution retenue          | Alternatives       | Justification                                             |
| ---------------- | ------------------------- | ------------------ | --------------------------------------------------------- |
| Firewall/UTM     | **pfSense**               | OPNsense           | Plus répandu, communauté large, documentation riche       |
| IDS/IPS          | **Suricata**              | Snort, Zeek        | Multi-thread, meilleures performances, natif pfSense      |
| Authentification | **FreeRADIUS + Samba AD** | LDAP direct        | Architecture 3-tiers professionnelle, comptabilité RADIUS |
| VPN              | **WireGuard**             | OpenVPN, IPsec     | Simple, rapide, moderne                                   |
| Proxy            | **Squid + SquidGuard**    | pfBlockerNG        | Filtrage catégories, groupes utilisateurs                 |
| Virtualisation   | **VirtualBox**            | VMware Workstation | Gratuit, maîtrisé par l'équipe    |

**Justifications détaillées pour chaque choix :**

- **pfSense :** Distribution FreeBSD mature, n°1 des pare-feu open source (Peerspot 2024), plugins intégrés (Suricata, WireGuard, Squid)
- **FreeRADIUS :** Serveur RADIUS le plus déployé au monde, architecture modulaire, support LDAP natif
- **Suricata :** Moteur multi-thread, règles ET Open gratuites, intégration pfSense native
- **WireGuard :** Protocole moderne intégré au noyau Linux/FreeBSD, configuration minimale

#### 4.2 Architecture de la solution

```
                    ┌──────────┐
  Internet ─────────┤   WAN    │
                    │  ────────┤
                    │ pfSense  │
                    │ ──────── │
                    │   LAN    │───── Client Windows (192.168.1.x)
                    │ ──────── │
                    │   DMZ    │───── Ubuntu Samba AD (192.168.2.20)
                    └──────────┘
```

**Plan d'adressage :**
- WAN : DHCP (NAT VirtualBox)
- LAN : 192.168.1.1/24
- DMZ : 192.168.2.1/24
- VPN : 10.0.0.0/24

#### 4.3 Répartition des rôles

#### 4.4 Planification (Diagramme de Gantt)

#### 4.5 Budget

Solution 100 % open source → coût nul (licences).
- Infrastructure : VirtualBox (gratuit)
- Firewall : pfSense CE (gratuit)
- AD : Samba 4 (gratuit)
- IDS/IPS : Suricata (gratuit)
- Coût humain : ~4 mois × 5 étudiants

---

### 5. Synthèse technique (8–10 pages)

**Sources :** Phases 5, 6, 7 (tous les fichiers), AI steps.md
#### 5.1 Installation et configuration de pfSense

**Source :** Phase 5 pt1, pt2, pt3

- **Création VM :** 2 CPU, 4 Go RAM, 32 Go disque, FreeBSD 64-bit
- **Interfaces :** WAN (NAT), LAN (Réseau interne 192.168.1.1/24), DMZ (Réseau privé hôte 192.168.2.1/24)
- **Configuration initiale :** Fuseau Africa/Algiers, DNS 8.8.8.8, 1.1.1.1
- **DMZ :** Interface OPT1 renommée, IP 192.168.2.1/24
- **Règles firewall :**
  - LAN : DNS (53), DHCP (67-68), Web Admin (80,443), ICMP uniquement
  - DMZ : passage vers Internet
  - WAN : WireGuard (51820 UDP), anti-lockout
  - **Aucune règle "Pass Any Any" sur le LAN** (contournerait le portail captif)
- **QoS :** Traffic Shaper pour priorisation bande passante

#### 5.2 Contrôleur de domaine Samba AD

**Source :** Phase 6 pt1

- **VM Ubuntu Server** (192.168.2.20), console-only, DMZ uniquement
- **Installation Samba 4** comme contrôleur de domaine (AD DC)
- **Domaine :** `ifag.local`
- **Utilisateurs créés :** `pfeuser`, `etudiant1`, `prof1`, `admin1`
- **Groupes :** `Etudiants`, `Professeurs`, `Administrateurs`
- **Vérification :** `samba-tool user list`, `smbclient -L localhost`

#### 5.3 Intégration FreeRADIUS → Samba AD

**Source :** Phase 6 pt1, Phase 7 pt1

- **FreeRADIUS installé sur pfSense** (package, pas VM dédiée)
- **Architecture 3-tiers :** Client → FreeRADIUS (127.0.0.1) → LDAP → Samba AD
- **Configuration LDAP** dans `mods-enabled/ldap`
- **Problème rencontré :** FreeRADIUS trouvait l'utilisateur AD mais refusait l'authentification
  - *Cause :* `Auth-Type` non défini
  - *Solution :* Création de `/usr/local/etc/raddb/users` avec `DEFAULT Auth-Type := LDAP`
- **Validation :** `radtest pfeuser "P@ssword123" 127.0.0.1 0 "admin123@"` → `Access-Accept`
- **Profil utilisateurs (bandwidth limits) :**
  - Étudiants : 2 Mbps Down / 512 Kbps Up
  - Professeurs : 10 Mbps Down / 2 Mbps Up
  - Administrateurs : illimité

#### 5.4 Portail captif

**Source :** Phase 6 pt2, Phase 7 pt1

- **Configuration :** Services → Captive Portal → zone LAN
- **Authentification :** RADIUS (FreeRADIUS 127.0.0.1)
- **HTTPS activé** (certificat auto-signé)
- **Page de connexion personnalisée :** logo IFAG intégré
- **Règles LAN strictes :** 4 règles uniquement (DNS, DHCP, Web Admin, ICMP)
- **Pas de "Pass Any Any"** pour éviter le contournement
- **Fonctionnement :** Le client LAN est redirigé vers le portail avant tout accès Internet

#### 5.5 Proxy et filtrage web (Squid + SquidGuard)

**Source :** Phase 6 pt3

- **Squid :** Proxy transparent HTTP sur interface LAN
- **SquidGuard :** Filtrage par catégories
- **Blacklist :** Shallalist (catégories : pornographie, réseaux sociaux, jeux, etc.)
- **Common ACL :** Politique de blocage globale appliquée à tous les utilisateurs
  - *Justification :* Sécurité maximale dès le déploiement, simplification administrative
- **Blocage des catégories non productives** (porn, ads, social\_networks, games)

#### 5.6 Détection d'intrusion (Suricata)

**Source :** Phase 7 pt1

- **Prérequis :** Désactivation des 3 offloadings (Hardware Checksum, TCP Segmentation, Large Receive Offloading) dans System → Advanced → Networking
- **Installation :** Package Suricata depuis l'interface pfSense
- **Interface surveillée :** WAN uniquement
- **Mode :** IDS (détection sans blocage)
- **Règles :** ET Open (téléchargées et activées)
- **Fonctionnement :** Suricata analyse le trafic entrant/sortant du WAN et génère des alertes pour toute activité suspecte (scans Nmap, malwares, tentatives d'exploitation)

#### 5.7 VPN WireGuard

**Source :** Phase 7 pt1

- **Installation :** Package WireGuard sur pfSense
- **Configuration tunnel :**
  - Interface VPN : 10.0.0.1/24
  - Port : 51820 UDP
  - Client : 10.0.0.2/32
- **Client Windows :** Application WireGuard, clé publique échangée
- **Règles firewall :**
  - WAN : UDP 51820 autorisé
  - Interface VPN : tout trafic autorisé
- **Résultat :** Handshake établi, pings OK (10.0.0.1, 192.168.1.1, 192.168.2.20)

#### 5.8 Traçabilité et conformité légale

**Source :** Phase 7 pt2

- **Cadre légal :** Loi algérienne 18-07 (cybersécurité) — conservation des logs 12 mois
- **Sources de logs :**
  - Firewall (filter.log)
  - DHCP (dhcpd.log)
  - RADIUS Accounting (identité, durée, volume)
  - Squid (sites visités)
  - Suricata (alertes)
  - Portail captif (connexions/déconnexions)
- **Centralisation :** Envoi des logs pfSense vers Ubuntu DMZ (192.168.2.20) via syslog (port 514 UDP)
- **Filtre rsyslog :** `if $fromhost == "_gateway" then /var/log/pfsense.log & stop`
- **Vérification :** `logger -h 192.168.2.20 -P 514 "test log pfsense"` → message reçu dans `/var/log/pfsense.log`

#### 5.9 Supervision — 3 approches tentées sans succès

**Source :** Phase 7 pt2

Une tentative de supervision graphique avec Grafana + InfluxDB a été menée via 3 approches successives :

1. **Telegraf natif sur pfSense** — le package refusait de se lancer
2. **Prometheus + snmp-exporter** — abandonné (problèmes de génération snmp.yml / MIBs)
3. **Telegraf en Docker (custom `telegraf-mibs`)** — échec d'initialisation SNMP (MIBs non résolus, snmp.conf non corrigé malgré une image custom avec téléchargement forcé des MIBs)

Une **4e approche** (script Python SNMP → InfluxDB) a été écrite mais pas finalisée (authentification InfluxDB v1 non configurée).

→ La supervision est laissée en perspective d'amélioration future.

#### 5.10 Tests et validation

**Source :** Phase 8

**Tableau récapitulatif des tests :**

| # | Composant | Test | Statut |
|---|-----------|------|--------|
| 1 | Samba AD | `samba-tool user list`, `smbclient` | ✅ |
| 2 | FreeRADIUS | `radtest` → Access-Accept | ✅ |
| 3 | Portail Captif | Connexion navigateur, authentification RADIUS | ✅ |
| 4 | Squid + SquidGuard | Blocage sites interdits (shallalist) | ✅ |
| 5 | VPN WireGuard | Handshake, ping ressources LAN/DMZ | ✅ |
| 6 | Suricata | Scan nmap → alertes détectées | ✅ |
| 7 | Firewall | Règles LAN/DMZ/WAN appliquées | ✅ |
| 8 | Traçabilité | Logs centralisés sur Ubuntu DMZ | ✅ |
| 9 | DNS Forwarder | Résolution DNS vers l'extérieur | ✅ |

---

### 6. Bilan (2 pages)

#### 6.1 Objectifs atteints
- ✅ Firewall pfSense segmenté (WAN/LAN/DMZ)
- ✅ Contrôleur de domaine Samba AD
- ✅ Authentification RADIUS centralisée
- ✅ Portail captif avec page personnalisée IFAG
- ✅ Filtrage web (SquidGuard + shallalist)
- ✅ IDS/IPS Suricata
- ✅ VPN WireGuard
- ✅ Traçabilité des logs (conformité Loi 18-07)

#### 6.2 Difficultés rencontrées
- **FreeRADIUS → Samba AD :** Deadlock au démarrage, correction DNS + Auth-Type LDAP
- **Suricata :** Offloading à désactiver manuellement, redémarrage pfSense nécessaire
- **Supervision :** 3 approches testées sans succès (Telegraf pfSense ne démarrait pas, Prometheus + snmp-exporter bloqué sur les MIBs, Telegraf Docker custom avec MIBs forcées échouait aussi). Script Python SNMP → InfluxDB écrit mais non finalisé (auth InfluxDB v1 absente)
- **pfSense :** Démarrages lents et parfois capricieux (deadlock FreeRADIUS bloquait l'interface web, nécessité de redémarrer plusieurs fois)
- **VirtualBox :** Espace disque hôte limité , mode réseau à configurer manuellement (Réseau interne vs Pont), VM ralenties par les ressources limitées du PC hôte
#### 6.3 Compétences développées
- Administration pfSense (firewall, QoS, packages, debugging)
- Administration Samba AD (domaine, utilisateurs, groupes, LDAP)
- Configuration FreeRADIUS (LDAP bind, virtual servers, clients, debugging avec radiusd -X)
- Déploiement Docker (Grafana, InfluxDB, Telegraf, docker-compose)
- Administration Ubuntu (rsyslog, netplan, services systemd)
- SNMP (activation bsnmpd, test snmpwalk, débogage MIBs)
- Supervision (architectures Prometheus + snmp-exporter, Telegraf SNMP, InfluxDB v1/v2)
- Scripting Python (collecte SNMP → base de données)
- Gestion de projet en équipe (méthodologie, cahier des charges)
- Conformité réglementaire (Loi 18-07 algérienne — traçabilité, conservation des logs)

---

### 7. Conclusion (1 page)

- Rappel de la problématique et de la solution
- Synthèse des réalisations
- **Limites** : Supervision non réalisée, blacklist SquidGuard à optimiser
- **Perspectives :**
  - Google Auth 2FA avec FreeRADIUS
  - Fail2ban contre les attaques brute force
  - Ansible pour automatiser la configuration
  - Architecture Zero Trust
  - Migration vers du matériel physique (appliances Netgate)
- Ouverture : Une solution UTM open source comme pfSense peut offrir un niveau de sécurité équivalent aux solutions propriétaires, à coût nul, tout en respectant les contraintes légales locales.

---

### 8. Annexes

#### Annexe A : Schéma d'architecture réseau
- Diagramme des VMs et flux réseau (WAN/LAN/DMZ/VPN)

#### Annexe B : Configurations clés
- `/usr/local/etc/raddb/users` (règle Auth-Type LDAP)
- Configuration WireGuard (clés, peers, allowed IPs)
- Règles firewall pfSense
- Configuration rsyslog (filtre de logs)

#### Annexe C : Liste des captures d'écran
- Interface pfSense (dashboard, règles, services)
- Page de connexion du portail captif (logo IFAG)
- Alertes Suricata
- Client VPN WireGuard connecté
- Logs centralisés sur Ubuntu DMZ

#### Annexe D : Commandes utiles
```bash
# Test authentification RADIUS
radtest pfeuser "P@ssword123" 127.0.0.1 0 "admin123@"

# Voir les utilisateurs Samba
samba-tool user list

# Test log syslog
logger -h 192.168.2.20 -P 514 "test log"

# Voir les alertes Suricata
cat /var/log/suricata/suricata.log | tail
```

#### Annexe E : Diagramme de Gantt
(Inclure le Gantt depuis `UTM ifag Gantt.pod`)
