# Introduction
Vous trouverez ici mes notes sur l'utilisation du système de fichiers `ZFS` sur Proxmox

# Pratique

## Supprimer un nœud PVE de l'architecture ZFS
L'objectif ici est de supprimer un nœud PVE de mon architecture ZFS car son seul disque disponible pour le ZFS n'est pas adapté à ce système de fichiers.  

> [!warning]
> En effet, les SSD SATA grand public ne possèdent pas assez de cache pour ZFS. On préférera des NVMe de type entreprise (même si les modèles grand public fonctionnent) et des HDD de type "CRM

1. Retirez tous les jobs de réplication ZFS vers ce nœud dans `Datacenter` --> `Replication`
<img width="1342" height="567" alt="poseidon_zfs_replication_job" src="https://github.com/user-attachments/assets/dd41ebee-acb8-4f05-9dc5-7066b0863a9f" />
Dans mon cas, j'ai supprimé tous mes jobs à destination du nœud Poseidon
<br><br>

2. Une fois cela fait, retirez votre PVE de la liste des nœuds disponibles pour le ZFS dans `Datacenter` --> `Storage` --> cliquez sur votre stockage ZFS puis retirez-le de la liste
<img width="2115" height="644" alt="delete_zfs_node" src="https://github.com/user-attachments/assets/88063e8c-1de6-451c-8062-6815660e9309" />
<br>

3. Nous devons maintenant supprimer le pool ZFS du nœud dans son intégralité car celui-ci n'est constitué que du seul disque que nous allons réinitialiser

   a. Rendez-vous donc ensuite dans la gestion ZFS de votre nœud --> `Disks` --> `ZFS`
   
   b. Cliquez sur le pool ZFS, puis cliquez sur `More` en haut à droite et sélectionnez `destroy` pour supprimer le pool
<img width="9920" height="3240" alt="zsf_disk_destroy" src="https://github.com/user-attachments/assets/33569325-b6c1-4c73-8df3-f340058b28e6" />

Vous pouvez ici choisir de réinitialiser le(s) disque(s) après le(s) avoir sortis du pool. Nous choisissons de le faire ici afin de pouvoir le(s) réutiliser immédiatement

4. Vérifiez ensuite que vos/votre disque(s) ne contiennent plus aucune trace de ZFS dans l'onglet `Disks` de votre PVE
<br>

**Vous avez terminé !**  

Pour résumer tout ce que nous avons fait :
- vous avez sortis un nœud PVE de votre architecture ZFS
- supprimé son pool ZFS
- nettoyé son/ses disque(s) afin de pouvoir les réutiliser pour autre chose
