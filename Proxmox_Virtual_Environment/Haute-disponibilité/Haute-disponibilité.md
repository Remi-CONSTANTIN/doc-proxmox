# Introduction
Cette documentation n'est pas un tutoriel, et ne contient donc que très peu de tehnique.
La haute disponibilité d'un serveur sur proxmox se décompose en deux grande parties : 

---

### 1 - Stockage
La haute disponibilité d'un serveur passe en premier lieu par la disponibilité de ses fichiers.
Par défaut, une machine est installée dans `local-lvm`qui est un stockage local au nœud où elle se trouve.
Si le nœud, ou le disque où se trouve la machine, venait à tomber pour une quelconque raison, la VM s'arrête brutalement. Dans le meilleure des cas il y a seulement de la perte de données et dans le pire c'est la machine en entière qui est perdue. 

Pour éviter cela, nous avons plusieurs façon de rendre les données hautement disponibles.

---

#### 1.a - Réplication ZFS (asynchrone)
La réplication ZFS consiste, par exemple, à initialiser un nouveau disque avec le système de fichiers `zfs` sur plusieurs nœud d'un cluster (ou même tous) puis de répliquer les machines sur le disque `ZFS` de chaque nœuds toutes les X min/heures (min 1min).
Nous nous  inscrivons donc dans une logique d'**hyperconvergence**.

Pourquoi `ZFS` ? Tout simplement parce que c'est le seul système de fichiers qui supporte la réplication de Proxmox.

##### Technique
Pour mettre cela en place techniquement, il faut dans l'idéal : 
- Ajouter un/des nouveau(x) disque(s) sur chaque nœuds, de l'initialiser avec `ZFS`. Lors de la création, il est possible de sélectionner plusieurs disques pour y faire du RAID logiciel (Miroir, RAID10, RAIDZ, RAIDZ2, RAIDZ3, dRAID, dRAID3)
- Déplacer la/les serveur(s) sur le nouveau stockage `ZFS`
- Configurer la réplication pour chacune des machines. Et cela pour chaque nœuds vers lesquels répliquer la VM. Vous pouvez y choisir la fréquence de réplication

##### Inconvénients
Comme dit précédemment, la réplication sur les systèmes de fichiers ZFS se fait toutes les X min/heures. La perte de données est donc toujours possible en fonction de votre intervalle de réplication. De plus cela ajoute de la charge CPU et RAM sur les plus grosse infrastructures mais reste négligeable sur la plupart des Home-Lab.

---
#### 1.b - Réplication CEPH (synchrone)
La réplication via l'outils CEPH est une réplication en "live", elle est donc sensée être quasiment instantanée.
Pour configurer l'outil, cela se passe dans `votre-noeud` --> `CEPH`

##### Technique
Pour le mettre en place, cela nécessite un peu plus d'étapes que la réplication ZFS :
- Installation du paquet
- Configuration des `monitors` et des `managers`, servant à la gestion du cluster `CEPH`
- Ajoute des disques destinés au cluster
- On créé le pool CEPH
- On déplace des machines sur ce nouveau stockage qui apparaît en dessous de chaque nœuds

##### Inconvénients
De part le fait que la réplication soit quasiment instantanée (et par d'autres facteurs propre au fonctionnement de l'outil), le disque est souvent sollicité. Cela a pour conséquence d'user prématurément les disques n'ayant pas été conçus pour cet usage. Les SSD grands publiques vont donc voir leur durée de vie grandement s'écourter.
Autre problème, la réparations de l'outil peut-être assez compliqué.
Et pour finir, cela génère beaucoup de trafic réseau. Proxmox recommande donc des liens Gps en production (on peut se contenter de 2.5Gps dans un home-Lab je pense).

---
#### 1.c - Stockage partagé (SAN / NAS)
La dernière option viable pour la haute disponibilité est le stockage partagé. Cette méthode consiste à délocaliser entièrement le stockage des machines virtuelles sur un équipement distant dédié, accessible par le réseau.

Ici, on sépare la force de calcul (la RAM et le CPU fournis par les nœuds Proxmox) de la donnée (les disques virtuels stockés sur l'équipement distant). Si un nœud Proxmox tombe en panne, un autre nœud peut rallumer la VM instantanément puisqu'il a déjà accès à ses disques via le réseau.

On retrouve généralement deux grandes familles d'équipements :

- Le NAS (Network Attached Storage) : Souvent utilisé avec le protocole **NFS** (fichiers) ou **iSCSI** (blocs). Très populaire (ex: TrueNAS, Synology), c'est une solution abordable et courante pour les Home-Labs et PME.
- Le SAN (Storage Area Network) : Un réseau de stockage pur, souvent sous forme de baies très onéreuses utilisant de la fibre optique (Fibre Channel). C'est la norme dans les très grandes infrastructures pour des performances maximales.

##### Technique
Contrairement à ZFS ou Ceph, il n'y a aucune réplication à configurer côté Proxmox. Les données ne sont présentes qu'à un seul endroit : sur le stockage partagé. Pour le mettre en place, il suffit de :
- Configurer les partages (NFS, iSCSI...) sur le NAS/SAN
- Se rendre dans l'interface Proxmox (Datacenter --> Storage --> Add) et d'y déclarer ce stockage distant
- Déplacer les disques des VMs sur ce nouveau stockage.

##### Inconvénients
- Le SPOF (Single Point of Failure) : Si votre NAS/SAN tombe en panne, toutes les VMs de tout le cluster s'arrêtent net. Pour éviter cela, il faut redonder le NAS/SAN lui-même, ce qui double la facture.
- La dépendance au réseau : Tout le trafic de lecture/écriture des disques passe par le réseau. Sur une simple prise 1 Gbps, les performances seront très mauvaises. Il est fortement recommandé d'avoir un réseau dédié au stockage en 10 Gbps minimum (ou via agrégation de liens)
- Le coût : Le matériel réseau haut débit et les baies de disques dédiées représentent un investissement financier important

---
# 2 - Règles de haute-disponibilité
Une fois le stockage redondé, il faut procéder à plusieurs autres configurations.

#### 2.a - Réplication (ZFS seulement)
Là où la réplication du stockage entre les nœuds est automatique avec CEPH, ZFS, lui, n'est qu'il simple système de fichier qui supporte la réplication. Elle doit donc être mise en place manuellement pour chaque machines. La question ne se pose pas avec le SAN car la redondance des données est gérée du côté baie.

##### Technique
Nous devons donc nous rendre sur la VM, n'importe quel nœud ou sur le datacenter directement pour faire la synchronisation inter-nœuds. Rendez-vous dans l'onglet `Replication` de n'importe lequel des trois.

##### Inconvénients
Tout doit être configuré manuellement pour chaque nœuds 

---
#### 2.b - Règles 
Il est possible de dire aux VMs sur quel nœud du cluster elle doivent être obligatoirement/en priorité et dans quel état il faut essayer de maintenir la machine.
Tout se passe dans `Datacenter` --> `HA`
##### Technique
Pour ce faire, il faut :
- Ajouter la machine dans les ressources HA. On y choisis plusieurs paramètres et nettement celui de l'état dans lequel on souhaite essayer de maintenir la machine (allumée, éteinte etc...)
- Créer une règles donnant un ordre de priorité à chacun des nœuds. 0 étant la plus petite priorité.

##### Inconvénients
Il n'y a pas vraiment d'inconvénient à part le fait qu'il faille le faire pour chaque machines


# Conclusion
Proxmox nous met à disposition plusieurs moyens pour faire de la haute-disponibilité de stockage. Il est donc assez facile de trouver chaussure à son pied.

Le principal problème qui en découle est peut être la courbe d'apprentissage plutôt raide à cause de la multitude de menus pour arriver à configurer tous les aspects de la haute-disponibilité.
