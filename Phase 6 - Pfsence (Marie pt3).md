### 1. La Logique des Groupes (Côté Samba AD)
Gestion de profils d'accès différenciés (ex: prof, Etudiants, Administrateurs),
- **Groupe "Etudiants"** : Accès limité, sites récréatifs bloqués, quota de bande passante. (etudiant1)
- **Groupe "Professeurs"** : Accès complet, pas de limite. (prof1)
- **Groupe "Administrateurs"** : Accès complet, pas de limite, acces logs. (admin1)

```
# Profil pour les Étudiants : 2 Mbps Down / 512 Kbps Up / 8h de session
DEFAULT Ldap-Group == "Etudiants"
    WISPr-Bandwidth-Max-Down := 2048000,
    WISPr-Bandwidth-Max-Up := 512000,
    Session-Timeout := 10800,
    Fall-Through = No

# Profil pour les Professeurs : 10 Mbps Down / Pas de limite de temps
DEFAULT Ldap-Group == "Professeurs"
    WISPr-Bandwidth-Max-Down := 10240000,
    WISPr-Bandwidth-Max-Up := 2048000,
    Session-Timeout := 28800,
    Fall-Through = No

# Profil pour les Administrateurs : Accès total
DEFAULT Ldap-Group == "Administrateurs"
    Fall-Through = No
```

marche pas on reprend la config manuelle avec (pour une connection directe faut reapliquer tt ce qu'on a fait précédemment et pour la phase de test flm)

```
# On ne demande plus "Est-ce qu'il est dans le groupe ?"
# On dit juste "Si c'est cet utilisateur, applique ça"

etudtest Cleartext-Password := "test123@"
	WISPr-Bandwidth-Max-Down := 500000,
	WISPr-Bandwidth-Max-Up := 250000,
	Session-Timeout := 3600

etudiant1 Cleartext-Password := "etud123@"
	WISPr-Bandwidth-Max-Down := 2048000,
	WISPr-Bandwidth-Max-Up := 512000

prof1 Cleartext-Password := "chikh123@"
	WISPr-Bandwidth-Max-Down := 10000000
```
### 2. Le filtrage des sites (DNS Filter / Squid)

Le portail captif lui-même ne sait pas filtrer les sites (il gère juste l'accès). Pour bloquer des sites spécifiques selon le rôle, on a deux options sur pfSense :

1. **Squid + SquidGuard** : C'est le "Proxy" classique. C'est plus lourd mais ça permet de dire : _"Si l'utilisateur appartient au groupe Etudiant, alors applique le filtre 'Strict'"_.
    
Pour ce faire on installe le proxy Squid 
(def proxy : Le proxy est le point de passage obligé pour tout le trafic web (HTTP/HTTPS).) 

ensuite j'installe le package SquidGuard
SquidGuard est le "cerveau" qui contient les listes noires.

- **Action :** Installer le paquet **SquidGuard**.
    
- **Base de données :** On va télécharger la "Shallalist" ou la liste de l'Université de Toulouse. Ce sont des milliers de sites classés par catégories (pornographie, gambling, social_networks, etc.).

**Services > Squid Proxy Server** :

- **Enable Squid Proxy**.
- **Proxy Interface(s)** sur **LAN**.
- **Transparent HTTP Proxy** (pour que les PC n'aient rien à configurer).

**Services > SquidGuard Proxy Filter** :

- **General Settings** :  **Enable**.
    
- **Blacklist** : Coche **Blacklist** et entre l'URL d'une liste (on prend celle de toulouse c'est la plus complete que je connaisse  : `https://dsi.ut-capitole.fr/blacklists/download/blacklists.tar.gz`). Clique sur **Download** dans l'onglet "Blacklist".



![[Pasted image 20260502203757.png]]

### 3. Appliquer le blocage global (Common ACL)

Puisque tu ne veux pas gérer les groupes Samba ici, on va tout configurer dans **Common ACL** (la règle qui s'applique à tout le monde sur le LAN).

1. Va dans l'onglet **Common ACL**.
    
2. Clique sur la petite flèche devant **Target Categories**.
    
3. Ici, tu verras toutes les catégories de la liste (porn, ads, social_networks, games...).
    
4. Pour chaque catégorie que tu veux interdire, passe de `whitelist` à **deny**.
    
5. **Le point crucial pour ton "Bloquer tout"** : En bas de la liste, à la ligne **Default Access [all]**, mets **deny**.
    
6. Dans **Redirect info**, tape une adresse (ex: `http://192.168.2.1/block.php`) ou un message pour que l'étudiant sache pourquoi c'est bloqué.
    
7. **Sauvegarde** et n'oublie pas de cliquer sur **Apply** dans l'onglet General Settings.
    
![[Pasted image 20260502210352.png]]
### 4. Pourquoi c'est stratégique pour ton PFE ?

Même si tu n'utilises pas les groupes Samba dans SquidGuard, tu justifies ton choix ainsi :

- _"Pour garantir une sécurité maximale dès le déploiement, j'ai opté pour une politique de filtrage globale (Common ACL). Cela permet d'assurer qu'aucun utilisateur, quel que soit son groupe, ne puisse accéder à des contenus malveillants ou non productifs, simplifiant ainsi l'administration tout en durcissant la posture de sécurité de l'UTM."_
### 3. Les Quotas et Débits (Attributs RADIUS)

C'est là que ça devient technique et intéressant pour ton jury. FreeRADIUS peut envoyer des attributs spécifiques au pfSense au moment où `pfeuser` se connecte :

- **WISPr-Bandwidth-Max-Down** : Limite la vitesse de téléchargement.
    
- **Session-Timeout** : Déconnecte l'utilisateur après un certain temps (ex: 3600 secondes pour 1h).
    
- **Max-Daily-Session** : Limite le temps total cumulé par jour.

