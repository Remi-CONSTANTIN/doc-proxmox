# Introduction
Cette procédure fait suite à la création d'un LUN sur mon NAS Synology, que vous pouvez retrouver [ici](https://github.com/tadaron/Mon-Home-Lab/blob/4146b9909a38964018f17ecdcbde4efbe6cc987b/Infra/Services/Synology/SAN-Synology_configuration.md).  
L'objectif est d'utiliser un LUN pour créer un datastore Proxmox afin d'y mettre, par exemple, des machines virtuelles, des ISOs ou des images de conteneurs. 

# Procédure
Pouvoir utiliser un LUN comme datastore Proxmox ne se fait malheureusement pas en 2 clics et nécessite, dans un premier temps, la connexion du LUN puis son formatage.
Nous verrons ici comment faire cela.

## 0. Prérequis
Veuillez à avoir créé un LUN, à l'avoir associé à une cible iSCSI accessible depuis votre cluster PVE.  
Si besoin d'aide sur un NAS Synology, reférez vous à me documentation de [mise en place rapide](https://github.com/tadaron/Mon-Home-Lab/blob/4146b9909a38964018f17ecdcbde4efbe6cc987b/Infra/Services/Synology/SAN-Synology_configuration.md).

## 1. Connexion du LUN
Rendez-vous sur votre cluster PVE dans l'onglet `Datacenter` --> `Storage` --> cliquez sur `Add` --> sélectionnez `iSCSI`  

<img width="1100" height="344" alt="iSCSI_pve" src="https://github.com/user-attachments/assets/6eb0712e-05c1-4b99-a7e0-f65b065e5adf" />  

Complétez avec vos informations :
- `ID` : nom d'affichage de la cible `iSCSI` dans l'interface Proxmox (purement esthétique)
- `Portal` : IP/Nom DNS de votre machine qui fournit le LUN (dans mon cas, mon NAS Synology)
- `Target` : si l'IP/Nom DNS est bon, vous devriez avoir la liste des cibles iSCSI proposées par votre SAN
- `Nodes` : vous pouvez restreindre ce stockage à certains nœuds PVE
- `Use LUNs directly` : **Important !** Si vous laissez cette case cochée, Proxmox va utiliser ce disque brut pour une seule machine virtuelle géante, ou vous pourrez juste l'assigner directement à un disque/VM.  
**Ce n'est pas du tout notre objectif ici, car nous voulons l'utiliser comme datastore.**

Une fois cela validé, restez dans le même onglet et passez à l'étape suivante.

## 2. Formatage du nouveau stockage
Recliquez sur `Add`, puis, cette fois, sélectionnez `LVM`  
<img width="605" height="395" alt="lvm_format_lun-proxmox1" src="https://github.com/user-attachments/assets/6cc44384-b9d7-48a2-a4ec-3cf2269d7de8" />

Remplissez toutes les informations relatives au LUN à formater :
- `ID` : comme d'habitude, c'est le nom d'affichage de ce stockage dans Proxmox
- `Base Storage` : sélectionnez la cible iSCSI sur laquelle aller chercher le LUN (j'ai nommé la mienne `LUN-Proxmox1` par abus de langage)
- `Base Volume` : le LUN visé. Vous pouvez avoir plusieurs LUN sous la même cible iSCSI
- `Volume group` : à vous de choisir le nom du VG qui va accueillir les données
- `Content` : le type de contenu que vous voulez y mettre (`Disk image` et/ou `Container`)
- `Shared` : permet de spécifier au cluster PVE qu'il s'agit d'un emplacement partagé et non d'un stockage local. Cela permet la migration de machines inter-nœuds sans coupure
- `Wipe Removed Disk` : habituellement, lors d'une suppression de disque, seul l'index est modifié, mais pas réellement les données sur le disque. Cette option permet d'écraser les données par des zéros, cela peut etre utile dans des environnements sensibles (bancaire, médical etc...)
- `Allow Snapshots as Volume-Chain` : en Beta sur PVE 9.2.3. Permet de faire des "snapshots" sur un stockage LVM qui ne le supporte habituellement pas

Une fois cela fait, vous devriez voir un nouvel emplacement de stockage, mais cette fois ci utilisable pour y mettre des VMS/conteneurs

# Conclusion
L'utilisation d'un SAN sous Proxmox n'est pas chose particulièrement aisée ni optimisée.    
En effet, on notera l'impossibilité d'avoir accès à des snapshots fiables sur **PVE 9.2.3**, même si la fonctionnalité `Allow Snapshots as Volume-Chain` nous est proposée en Preview. On notera aussi le fait que le LVM ne fait pas de `Thin provisionning`et reserve donc la taille entière de la machine dès la création (`Thick provisionning`).   

En l'état des choses, la concurrence supporte mieux les snapshots. VMWare propose le système de fichier `VMFS`, et XCP-NG utilise lui aussi LVM mais stocke les VMs en `.VHD`.
