## 1. Mettre en place DMZ

**Creation du serveur Ubuntu pour les test :**
- Nom : `ServIfag`
- Type : Linux
- Version : Ubuntu (64-bit)
- Adapter 2 → Réseau privé (DMZ)

Configuration de l'interface sur PfSense :
![[Pasted image 20260324190014.png]]

- Interfaces → Assignments
- Add → OPT1
- Renommer en **DMZ**
- Enable interface
- Ipv4 config --> IPv4 Static
- 192.168.2.1/24
![[Pasted image 20260324191128.png]]
![[Pasted image 20260324191307.png]]

On dit ensuite configurer notre firewall pour permettre a notre serveur de ping le routeur et l'internet

![[Pasted image 20260325100042.png]]

on repasse a la configuration du serv test on doit lui assigner une ip statique pour etablir la connection avec la DMZ

![[Pasted image 20260325101228.png]]

on peux voir que des que j'ai configuré on peux ping notre pfsense
test vers l'internet ping 8.8.8.8

![[Pasted image 20260325102519.png]]

un peu lent mais sa marche!
mtn on test vers une ip hors DMZ

![[Pasted image 20260325102908.png]]

parfait ! (thats what i though)

probleme avec ping google.com --> DNS mal configuré

config du fichier .yaml

![[Pasted image 20260326075923.png]]

apres avoir resolu le probleme on met a jour le serveur avec apt update 
ensuite on installe apache

![[Pasted image 20260326081625.png]]

notre serveur est fonctionnel mais le lan n'a encore aucun acces a la dmz donc le client ne peux pas avoir acces au site du serv on doit donc passer a la configuration de notre firewall

![[Pasted image 20260326081804.png]]

