**1. Définir le problème**
- Le réseau de l’IFAG existe déjà
- mais la sécurité actuelle est insuffisante face aux menaces modernes
- absence de gestion centralisée
- manque de traçabilité des utilisateurs

**2. Définir l’objectif**

- Déployer une passerelle de sécurité open source pour gérer et filtrer les accès internet (Firewall, ACLs,..),  
- protéger le réseau local (LAN) contre les menaces provenant d'Internet (WAN) et gérer les privilèges utilisateurs,  
- Optimisation des ressources : Gérer et prioriser la bande passante selon les besoins,  
- Interconnexion des zones WAN (Internet), LAN (Utilisateurs) et éventuellement mettre en place une DMZ,  
- Contrôle des accès : Identifier et authentifier les utilisateurs avant d'autoriser l'accès aux ressources.

**3. Identifier les acteurs**

| Rôle            | Qui                                                                                                       |
| --------------- | --------------------------------------------------------------------------------------------------------- |
| Client          | IFAG                                                                                                      |
| Chef de projet  | Djouaher Mariya                                                                                           |
| Équipe projet   | Boutekrabt Maria<br>Benmerabet Nada Fella<br>Abderraouf Derradji Kasri<br>Tabbi Mohamed Kamel Seif Eddine |
| Utilisateurs    | Étudiants / professeurs / Administration                                                                  |
| Administrateurs | Service IT                                                                                                |
| Encadrant       | Benaissa Adnane                                                                                           |

### 1. L'Interconnexion Authentification (LDAP/AD)

C'est le lien entre la page de login et ton serveur Samba.

- **Action :** Aller dans `System > User Management > Authentication Servers`.
    
- **But :** Ajouter ton serveur Samba comme source d'authentification. Ainsi, quand un utilisateur se connecte sur le Portail Captif, pfSense ira demander au serveur Samba si le mot de passe est correct.


### 2. Le Filtrage de Contenu (Le "Web Filtering")

Pour bloquer les sites malveillants ou non autorisés à l'IFAG.

- **Action :** Installer le paquet **pfBlockerNG**.
    
- **But :** Activer le filtrage DNS (DNSBL) pour bloquer des catégories de sites (publicités, phishing, réseaux sociaux si besoin). C'est une spécification majeure de ton cahier des charges.
    

### 3. La Détection d'Intrusion (IDS/IPS)

C'est ce qui transforme ton pare-feu en "bouclier intelligent".

- **Action :** Installer **Snort** ou **Suricata**.
    
- **But :** Configurer l'analyse des paquets sur l'interface WAN et DMZ pour détecter des tentatives de scan (comme Nmap) ou des exploits connus.
    

### 4. La Gestion de la Bande Passante (Traffic Shaping)

- **Action :** Aller dans `Firewall > Traffic Shaper`.
    
- **But :** Limiter la vitesse des étudiants pour éviter qu'un seul utilisateur ne sature toute la connexion de l'école (QoS).
    

### 5. La Journalisation et le Reporting (Logs)

Pour ta partie "Conformité légale".

- **Action :** Configurer `Status > System Logs`.
    
- **But :** Vérifier que pfSense garde une trace de qui s'est connecté et quand. Pour ton PFE, tu peux même installer **ntopng** pour avoir des graphiques très détaillés sur l'utilisation du réseau.
