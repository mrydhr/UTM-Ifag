## Étape 1 | Recherches

###  Définir les technologies à utiliser
#####  1 - Firewall / UTM : 
###### PFsence :
(source : https://www.pfsense.org/about-pfsense/index.html)

pfSense est une distribution open source basée sur FreeBSD, conçue comme pare-feu et routeur entièrement administrable via une interface web. Développé par Rubicon Communications (Netgate), il s’impose comme l’une des solutions de sécurité réseau les plus utilisées dans les environnements professionnels et domestiques.
Le logiciel est proposé en deux éditions : **Community Edition**, libre et gratuite, et **pfSense Plus**, optimisée et supportée commercialement par Netgate. Il peut être installé sur du matériel tiers, sur des appliances Netgate (ex. 6100, 8200) ou déployé dans le cloud sur Amazon Web Services et Microsoft Azure
Avec plusieurs millions d’installations actives dans le monde, pfSense est reconnu pour sa fiabilité, sa flexibilité et sa rentabilité. En 2024, il a été classé **n°1 des solutions de pare-feu** sur le site d’évaluations Peerspot, salué pour sa convivialité et ses performances élevées

###### Alternative - OPNSence: 
(source : https://opnsense.org/)

OPNsense est une distribution libre et open source de pare-feu et de routage, basée sur FreeBSD et développée par la société néerlandaise Deciso B.V. Lancée officiellement en 2015, elle est conçue pour offrir des fonctions de sécurité réseau de niveau professionnel dans un cadre transparent, flexible et communautaire.
OPNsense comprend un pare-feu à états pour IPv4/IPv6, un système de détection et prévention d’intrusion (IDS/IPS : Suricata ou Snort), la gestion VPN (IPsec, OpenVPN, WireGuard), un proxy filtrant, le façonnage de trafic et la gestion multi-WAN. Son interface web moderne et son architecture modulaire (framework MVC avec API) facilitent l’extension via des plugins comme Zenarmor, NGINX ou HAProxy.
Apprécié pour sa stabilité, sa souplesse et son coût nul de licence, OPNsense figure parmi les meilleures solutions de pare-feu open source, utilisée tant dans les PME que dans les laboratoires ou administrations. Les évaluations d’experts soulignent sa convivialité, sa transparence et sa fréquence d’évolution, tout en recommandant des améliorations sur l’interface et la documentation.

###### Choix final :
###### PfSense

PfSense est l’une des solutions firewall open source **les plus utilisées dans les entreprises et les laboratoires réseau**.
Ce qui la rend :
- largement testée
- fiable
- bien documentée
Ce qui faciliteras :
- l’apprentissage
- la résolution des problèmes

##### Conclusion :
Nous avons choisi pfSense car c’est une solution open source très répandue, stable et largement documentée. Elle dispose d’une grande communauté et permet d’intégrer facilement plusieurs fonctionnalités de sécurité comme l’IDS/IPS avec Suricata, le VPN ou le filtrage réseau. Cela en fait une solution adaptée pour déployer une architecture UTM dans notre projet.

#####  2 - Firewall / UTM : (test phase)

######  Suricata: 
(source : https://suricata.io/our-story/suricata/)

Suricata est un moteur libre et open source de détection et de prévention d’intrusions réseau (IDS/IPS). Développé par la **Open Information Security Foundation**, il offre une analyse en profondeur du trafic (deep packet inspection) et une surveillance de la sécurité réseau en temps réel pour identifier et bloquer les menaces. Il est largement utilisé dans les secteurs public et privé pour la détection d’anomalies et l’intégration dans les systèmes SIEM.

Suricata prend en charge le multi-threading, ce qui permet une inspection simultanée sur plusieurs processeurs sans perte de paquets. Son langage de règles et signatures avancé facilite la détection d’activités suspectes au niveau applicatif : HTTP, TLS, DNS, ou encore SSH. Les résultats sont exportables au format EVE JSON pour une intégration fluide dans des solutions et l’ajout de scripts permet de personnaliser la détection de comportements malveillants. www.itrion.eu/es/tech5-suricata

###### Alternative - Snort:
(Source : www.snort.org/document/)

Snort est un logiciel libre d’analyse de paquets réseau et un système de détection et de prévention d’intrusions (IDS/IPS) développé à l’origine par Martin Roesch et maintenu par Cisco Systems via le groupe Cisco Talos. Il identifie les menaces en temps réel en comparant le trafic réseau à des signatures d’attaques connues.

Snort capture et analyse les paquets transitant sur un réseau pour détecter des modèles d’attaques. Il s’appuie sur un moteur d’analyse modulaire, une base de signatures (règles Snort) et des modules de préprocesseurs qui normalisent le trafic. Il peut fonctionner en mode détection passif (IDS) ou en mode blocage actif (IPS).

###### Alternative - Zeek:
(Source : https://zeek.org/about/)

Zeek est une plateforme logicielle open source d’analyse du trafic réseau et de surveillance de la sécurité. Conçu pour convertir le trafic en journaux exploitables, Zeek est largement utilisé par les équipes de cybersécurité, les chercheurs et les opérateurs réseau pour comprendre l’activité, détecter les menaces et soutenir les enquêtes numériques.

Zeek fonctionne de manière passive : il observe le trafic réseau sans interférer. Son moteur d’événements transforme les paquets en événements de plus haut niveau (ex. requêtes HTTP, échanges DNS, certificats SSL). Ces événements sont interprétés par des scripts dans un langage spécifique à Zeek, permettant de consigner, corréler ou alerter selon des politiques définies. Il peut être déployé sur matériel standard ou en cluster pour surveiller des liens à très haut débit.

###### Choix final :
###### Suricata

Il analyse le trafic réseau pour :

- détecter les attaques
- identifier des comportements suspects
- bloquer certaines intrusions

pfSense permet d’installer Suricata directement comme **plugin**.

- installation rapide
- gestion via l’interface pfSense

Suricata gère le traitement **multi-thread** ce qui lui permet d'analyser :

- plusieurs flux réseau simultanément
- trafic à haut débit

Suricata utilise des **bases de signatures** pour détecter les attaques :

- malware
- scans réseau
- attaques web
- brute force

##### Conclusion :
Parmi les différentes solutions IDS/IPS étudiées, Suricata a été retenu car il offre de hautes performances grâce au traitement multi-thread, une bonne intégration avec pfSense et la possibilité de fonctionner à la fois comme système de détection et de prévention d’intrusion.

#####  3 - Solution d’authentification :
###### Active Directory :
(source : https://learn.microsoft.com/fr-fr/windows/win32/ad/active-directory-domain-services
https://www.manageengine.com/products/ad-manager/windows-active-directory-administration-tool.html)

**Active Directory (AD)** est un service d’annuaire développé par **Microsoft** pour les environnements Windows Server. Il centralise l’authentification, la gestion des identités et le contrôle des accès au sein d’un réseau d’entreprise. AD est essentiel dans les infrastructures informatiques car il permet d’administrer efficacement utilisateurs, groupes, ordinateurs et ressources partagées.

Présent dans la majorité des entreprises utilisant Windows, AD permet de :

- Centraliser la création et la suppression de comptes.
- Appliquer des **stratégies de groupe (Group Policy)** pour sécuriser et configurer les postes.
- Faciliter la conformité aux politiques de sécurité et réglementations (GDPR, SOX, HIPAA).
- Intégrer des environnements hybrides avec **Microsoft Entra ID** pour gérer les identités cloud.

###### FreeRADIUS :
(source : https://www.freeradius.org/documentation/freeradius-server/3.2.9/concepts/freeradius.html)

FreeRADIUS est un serveur d’authentification open source implémentant le protocole RADIUS (Remote Authentication Dial-In User Service) pour la gestion centralisée des fonctions AAA : Authentification, Autorisation et Comptabilité. Logiciel libre sous licence , il est le serveur RADIUS le plus utilisé au monde, déployé par des FAI, des entreprises et des universités dans le cadre de réseaux filaires, Wi-Fi et VPN

FreeRADIUS s’appuie sur une architecture modulaire : chaque module assure une fonction précise (authentification ; journalisation ; accès SQL ou LDAP). Cette conception permet d’ajouter ou retirer des composants sans modifier le cœur du serveur. Le système gère aisément des millions de requêtes d’authentification grâce à un traitement multithread et à la prise en charge du clustering pour la redondance et la montée en charge

###### Architecture :
- L’utilisateur tente d’accéder au réseau  
- Le firewall (pfSense) demande une authentification  
- La demande est envoyée au serveur RADIUS  
- RADIUS vérifie l’utilisateur dans Active Directory  
- Accès autorisé ou refusé
##### Conclusion :
La solution retenue repose sur Active Directory pour la gestion centralisée des utilisateurs et FreeRADIUS pour l’authentification réseau. Cette combinaison permet de contrôler l’accès au réseau tout en assurant une gestion centralisée des comptes et une meilleure traçabilité des activités des utilisateurs.

#####  4 - L’environnement de virtualisation : 

###### VirtualBox :
source : (https://www.virtualbox.org/manual/topics/Introduction.html#ct_about-virtualbox)

VirtualBox est un logiciel de virtualisation libre et open source qui permet d’exécuter plusieurs systèmes d’exploitation sur un même ordinateur physique. Développé initialement par **Innotek** puis acquis par **Oracle Corporation**, il est aujourd’hui connu sous le nom complet **Oracle VM VirtualBox**. Il est largement utilisé pour le test, le développement et l’apprentissage de divers environnements logiciels.

###### Altrnative - VMware Workstation:
source : (https://www.vmware.com/products/desktop-hypervisor/workstation-and-fusion#overview)

VMware Workstation est un logiciel de virtualisation de bureau développé par **VMware**. Il permet d’exécuter plusieurs systèmes d’exploitation invités sur un seul ordinateur physique via des machines virtuelles. Outil populaire chez les développeurs, testeurs et administrateurs systèmes, il offre une isolation complète entre environnements virtuels et physiques.

##### Conclusion :
Après avoir étudié VirtualBox et VMware Workstation, nous avons choisi VirtualBox pour notre projet. Ce logiciel est plus simple à utiliser et suffisant pour la mise en place de notre laboratoire virtuel. De plus, notre encadrant le maîtrise mieux, ce qui facilitera l’accompagnement. Bien que VMware soit plus répandu en entreprise, VirtualBox reste une solution adaptée pour un projet académique.

