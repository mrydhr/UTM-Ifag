## 1. Installation pfSense basique
vidéo tuto d'installation suivi : https://youtu.be/o4LHO5hYCxQ?si=4vAv6CU-Hf6sAs6M

![[Pasted image 20260322103301.png]]

**Matériel VM (VirtualBox/VMware) :**

- 2 CPU, 4GB RAM, 32GB disque
- **Interfaces :** WAN (NAT/Bridged), LAN (Internal), DMZ (Internal2)​

**Étapes :**

1. Télécharge ISO pfSense CE 2.8+ : pfsense.org/download
2. Création de la nouvelle machine avec l'iso pfsense + FreeBSD64 
3. Affectation des 3 interface réseaux pour la machine
	1. WAN  : NAT (adapter 1) | |Internet|
	2. LAN : Réseau interne (adapter 2) | Utilisateurs
	3. DMZ : Réseau interne (adapter 3) | Services exposés
4. Installation et configuration de pfsense
	1. ![[Pasted image 20260322114114.png]]

**Test :**
Machine de test : WIN10
 LAN : Réseau interne (adapter 1)

![[Pasted image 20260322120615.png]]

on a verifier la connection entre pfsence et notre machine client on peux maintenant acceder a l'interface locale de pfsense a travers son ip

![[Pasted image 20260322121026.png]]

apres s'etre connecté en tant qu'admin on change le fuseau horaire et modifie le mdp : 12345 pour l'instant on veux juste tester la connectivité des machines entre elles 
**Configuration initiale web :**

- **Fuseau** : Africa/Algiers
- **Mot de passe admin** : 12345 (test)
- **DNS** : 8.8.8.8 / 1.1.1.1

![[Pasted image 20260322123830.png]]

test ping internet

![[Pasted image 20260322124042.png|684]]

test :
	Si on coupe Pfsense alors le client n'a plus acces a internet, le relancer reactive l'acces vers internet

**Résultats :**

- **Ping pfSense** : `ping 192.168.1.1` → OK (DHCP auto 192.168.1.x)
- **Ping Internet** : `ping 8.8.8.8` / `ping google.com` → OK via WAN pfSense
- **Test Firewall** : Arrêt pfSense → Client perd Internet → Relance → Récupéré

**Preuve fonctionnalité** : pfSense agit bien comme **passerelle unique** WAN→LAN (comportement UTM attendu).​

**Note mémoire Ch3.1 :**  
_"L'installation réussie valide la topologie WAN(Internet)-LAN(IFAG). Captures : interfaces assignées, accès web, test ping cut/restart. Prochaine étape : Portail Captif."_