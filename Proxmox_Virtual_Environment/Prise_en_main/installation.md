## Presentation Proxmox :

**Proxmox Virtual Environment (VE)** est une plateforme open source de gestion de serveurs qui permet de déployer et de gérer des machines virtuelles (KVM) et des conteneurs légers (LXC) depuis une interface web centralisée. C'est une solution tout-en-un qui intègre également le stockage défini par logiciel et la gestion du réseau, offrant une alternative gratuite et performante aux solutions d'entreprise comme VMware ESXi.
## Installation rapide et prise en main :

Le lien d'installation de l'iso du Proxmxo 9.2 :
```
https://enterprise.proxmox.com/iso/proxmox-ve_9.2-1-arm64.iso
```
Utilisez un outil comme BalenaEtcher ou Rufus pour flasher cette image sur votre clé USB afin de la rendre bootable.

Démarrez votre serveur sur la clé USB. L'installateur graphique vous guidera à travers plusieurs étapes, Spécifiez : l'adresse IP, le Nom, Le Serveur DNS, et le Mot de passe root.

Enfin d'installation , connectez vous à :
https://ip_proxmox:8006
Avec les identifiants :
<u>root : Votre Mot de passe </u>
