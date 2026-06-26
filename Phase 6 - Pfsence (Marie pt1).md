Recu machine samba, mise en place de l'adressage ip 
sources : 
https://wiki.samba.org/index.php/Authenticating_Freeradius_against_Active_Directory
https://www.freeradius.org/documentation/freeradius-server/4.0.0/howto/datastores/ad/samba.html
https://www.inkbridgenetworks.com/blog/blog-10/can-you-use-freeradius-and-active-di
https://www.slideserve.com/ramosa/radius-and-freeradius-powerpoint-ppt-presentation




logins :
server-ifag login : admin-samba
password : admin123@

carte reseaux : DMZ only
addressage avec netplan ip : 192.168.2.20

==**a revoir :**==
addressage ip pk manuel - conflit d'addresses ip a test
dhcp def + addressage

### 1. L'Interconnexion Authentification (LDAP/AD)

C'est le lien entre la page de login et le serveur Samba.
## Synthèse de l'évolution de l'architecture d'authentification

Initialement, le projet prévoyait une liaison **LDAP directe** entre le portail captif **pfSense** et l'annuaire **Samba4**. Cette méthode, bien que fonctionnelle, présentait des limites en termes de souplesse de configuration et de gestion fine des sessions utilisateurs.

Pour répondre aux exigences de sécurité et de modularité d'un environnement **UTM (Unified Threat Management)**, nous avons choisi de faire évoluer cette architecture en intégrant **FreeRADIUS** sur le pfSense. Cette solution offre plusieurs avantages stratégiques pour le projet :

- **Abstraction et Standardisation :** RADIUS agit comme une passerelle universelle. pfSense interroge le service RADIUS local, qui se charge de valider les identifiants auprès du contrôleur de domaine Samba.
    
- **Sécurité Renforcée :** L'utilisation de protocoles d'authentification standardisés et d'un "Secret Partagé" sécurise les échanges entre les services.
    
- **Contrôle Avancé :** RADIUS permet de mettre en place des politiques de gestion plus strictes, comme la limitation du temps de connexion ou le suivi précis de la consommation de données par les étudiants.
    

Cette transition d'un modèle de connexion directe vers une architecture **3-tiers** (Client → RADIUS → LDAP) professionnalise l'infrastructure et garantit une gestion centralisée et évolutive de l'accès réseau au sein de l'IFAG.

On commence alors avec l'installation de FreeRadius3 sur l'interface web de Pfsence![[Pasted image 20260426200832.png]]

### Optimisation des Ressources et Rationalisation de l'Infrastructure

Le choix d'installer **FreeRADIUS** en tant que package directement intégré au système **pfSense**, plutôt que sur un serveur dédié, repose sur une stratégie d'optimisation des performances et de rentabilité de l'infrastructure virtuelle :

- **Maximisation des performances (Performance per Watt/RAM)
- **Réduction de la latence réseau
- **Rentabilité et Simplification de l'UTM**

Apres l'installation la premiere chose a faire est de configurer RADIUS pour qu'il parle avec lui meme vu qu'on le tourne en local sois dans le meme environnement que pfsense

![[Pasted image 20260426201633.png]]

**Client Shared Secret :** admin123@

Ensuite on le branchera a samba
## Journal de Bord : La Résurrection de FreeRADIUS & Samba

### 1. Le Diagnostic Initial : Le "Trou Noir"

- **Symptôme :** `radiusd -X` ne renvoyait absolument rien. L'interface pfSense tombait en erreur 504 (Gateway Timeout).
    
- **Cause réelle :** Un **Deadlock système**. FreeRADIUS tentait de résoudre l'adresse IP de Samba via DNS (Inverse Lookup) avant même de démarrer. Comme le DNS ne répondait pas, le processus restait figé, bloquant au passage le serveur Web de pfSense.
    

### 2. Nettoyage de la Configuration (Le "Debug" Chirurgical)

- **Erreur de syntaxe :** Corruption du fichier `mods-enabled/ldap` avec des accolades orphelines (`Too many closing braces`).
    
- **Fichiers fantômes :** Appels à des fichiers inexistants (`auth.conf`) qui empêchaient la lecture de la configuration globale.
    
- **Solution :** Réécriture manuelle des blocs de configuration via `sed` en ligne de commande et correction du **Distinguished Name (DN)** de l'administrateur pour correspondre exactement à l'arborescence Active Directory/Samba.
    

### 3. Optimisation du Module LDAP (Le "Fail-Fast")

Pour arrêter de "tourner dans le vide", nous avons appliqué une configuration de type **"Haute Disponibilité"** :

- **Désactivation des Referrals :** Ajout de `chase_referrals = no` pour empêcher RADIUS de se perdre dans les redirections complexes de l'AD.
    
- **Gestion du Pool :** Passage du `start_connections` à **0** pour permettre au service de démarrer instantanément sans attendre que le tunnel vers Samba soit établi.
    
- **DNS Kill :** Désactivation de `hostname_lookups` pour supprimer toute dépendance au DNS externe lors du démarrage.
    

### 4. Validation par "Force Brute"

- **Test Unitaire :** Utilisation de `ldapsearch` en ligne de commande pour prouver que le port **389** était ouvert et que les identifiants étaient valides.
    
- **Démarrage Interactif :** Passage en mode Debug (`radiusd -X`) via le **vrai Shell SSH** (et non l'interface Web) pour visualiser le flux de données en temps réel.

![[Pasted image 20260427194445.png]]

#### Samba AD refusait de donner les mots de passe en clair à RADIUS (Erreur `Failed retrieving values`).
#### Les Commandes de Débogage (Le mode "Expert")

Ouvre toujours deux fenêtres SSH.

**Fenêtre 1 (Le Cerveau) :**
```
# Arrêter le service en arrière-plan
killall -9 radiusd

# Lancer en mode debug (X = ultra détaillé)
radiusd -X
```

**Fenêtre 2 (L'Action) :**
```
# La commande de test ultime (Secret : admin123@)
radtest pfeuser P@ssword123 127.0.0.1 0 admin123@
```

- **Succès :** `Received Access-Accept`.
- **Échec :** `Received Access-Reject` ou `Shared secret is incorrect`.

---

###  Les Correctifs "Chirurgicaux" (`sed`)

Les commandes pour forcer :

| **Cible**       | **Commande (sed)**                                                                                                          | **Pourquoi ?**                                         |
| --------------- | --------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------ |
| **Performance** | `sed -i '' 's/hostname_lookups = yes/hostname_lookups = no/g' /usr/local/etc/raddb/radiusd.conf`                            | Évite que RADIUS ne freeze au démarrage.               |
| **Samba AD**    | `sed -i '' 's/chase_referrals = yes/chase_referrals = no/g' /usr/local/etc/raddb/mods-enabled/ldap`                         | Empêche RADIUS de se perdre dans les redirections AD.  |
| **Auth**        | `sed -i '' 's/^[\t ]*access_attr = "userPassword"/# access_attr = "userPassword"/g' /usr/local/etc/raddb/mods-enabled/ldap` | Force Samba à gérer le mot de passe lui-même.          |
| **Mode Bind**   | `sed -i '' "s/control:Auth-Type.*:=.*/control:Auth-Type := 'ldap'/g" /usr/local/etc/raddb/mods-enabled/ldap`                | Oblige RADIUS à utiliser la liaison LDAP pour valider. |

---

###  Vérification côté Samba

Si RADIUS est OK mais que tu as un `Reject`, vérifie l'utilisateur sur ton serveur Samba :


```
# Vérifier que l'utilisateur répond bien
smbclient -L localhost -U pfeuser%P@ssword123
```

- **La sauvegarde logique :** Exportation du fichier `config.xml` de pfSense sur flashdisk
- **La sauvegarde physique :** Archivage des fichiers du répertoire `/usr/local/etc/raddb/` qui contient la logique LDAP et les secrets clients.

1. **La problématique :** Pourquoi le `Access-Reject` ? (L'histoire du Bind LDAP et du Shared Secret).
2. **La méthodologie :** Utilisation de `radiusd -X` pour isoler le problème.
3. **La résolution :** Les commandes `sed` pour adapter FreeRADIUS à Samba AD.
4. **La validation :** Capture d'écran du succès final.


### Problème rencontré

Lors de la restauration de l'environnement FreeRADIUS/Samba AD, les utilisateurs Active Directory étaient correctement trouvés via LDAP, mais l'authentification échouait systématiquement.

Les journaux FreeRADIUS indiquaient notamment :

```
ldap: User object foundldap: WARNING: PAP authentication to Active Directory MUST set 'Auth-Type := LDAP'ERROR: No Auth-Type found
```

Cela montrait que :

- la connectivité LDAP était fonctionnelle ;
- le compte utilisateur était localisé dans Active Directory ;
- mais FreeRADIUS ne déterminait pas la méthode d'authentification à utiliser.

### Analyse

Les vérifications ont permis de confirmer :

- connectivité réseau entre pfSense et le contrôleur Samba AD ;
- authentification du compte de service LDAP ;
- recherche correcte des utilisateurs avec le filtre `sAMAccountName` ;
- configuration LDAP valide dans `mods-enabled/ldap`.

Le problème provenait finalement de l'absence d'une règle définissant explicitement le type d'authentification LDAP.

### Correction appliquée

Création du fichier :

```
/usr/local/etc/raddb/users
```

avec le contenu :

```
DEFAULT Auth-Type := LDAP
```

Après redémarrage du service FreeRADIUS, les tests d'authentification ont retourné :

```
Access-Accept
```

pour un mot de passe valide.

### Explication technique

Samba Active Directory ne fournit pas les mots de passe en clair via LDAP. FreeRADIUS ne peut donc pas comparer localement le mot de passe reçu.

La directive :

```
DEFAULT Auth-Type := LDAP
```

force FreeRADIUS à effectuer une authentification LDAP (bind LDAP) auprès d'Active Directory afin de vérifier le mot de passe directement sur le contrôleur de domaine.

Sans cette directive, FreeRADIUS identifiait correctement l'utilisateur mais ne savait pas quelle méthode utiliser pour valider ses identifiants, ce qui provoquait un rejet de l'authentification.