# Introduction

L'objectif de ce document est de décortiquer la mise en place d'un stockage distribué basique et répliqué via **ZFS**, couplé au mécanisme de **Haute Disponibilité (HA)** sous Proxmox VE (PVE).

Nous allons aborder le principe théorique des composants de stockage ZFS et des mécanismes d'élection de quorum et de basculement (*failover*), puis nous passerons à la mise en pratique pas à pas illustrée à partir des captures d'écran du cluster de test (`FYC-Cluster`).

---

# La théorie

Afin de pouvoir configurer sereinement un cluster Proxmox en Haute Disponibilité sans SAN/NAS centralisé, il est essentiel d'assimilation les concepts sous-jacents de ZFS et du gestionnaire de ressources HA (*CRM/LRM*).
<br><br>

## ZFS Storage Pool (zpool)
ZFS est un système de fichiers combiné à un gestionnaire de volumes logiques. Dans un cluster Proxmox à plusieurs nœuds physiques sans stockage partagé externe, ZFS permet d'utiliser le stockage local de chaque serveur pour y créer des volumes (*zpool*) portant un nom rigoureusement identique sur chaque nœud (ex: `zfs-pool`).

### Comparatif des modes de stockage PVE
| Mode de stockage | Nécessite un réseau dédié | Tolérance à la panne nœud | Copie des données locale |
| :--- | :--- | :--- | :--- |
| **Local LVM / LVM-Thin** | Non | Aucune | Oui (interne au serveur) |
| **ZFS Pool + Réplication** | Recommandé | Oui (via réplication Asynchrone) | Oui (sur chaque nœud) |
| **Ceph Storage** | Oui (10GbE+ obligatoire) | Oui (Synchrone en temps réel) | Distribuée sur le cluster |

> [!note]
> L'association de **ZFS + Réplication Proxmox** offre un excellent compromis pour les petits clusters (2 à 3 nœuds) qui ne disposent pas d'un réseau 10GbE dédié ni de 4 à 5 disques minimum par nœud requis pour Ceph.
<br><br>

## La Réplication ZFS
La réplication ZFS repose sur l'envoi d'instantanés (*snapshots*) incrémentiels d'un nœud source vers un ou plusieurs nœuds cibles.
- **Principe** : Toutes les X minutes (ex: toutes les 30 min ou toutes les 1 min), Proxmox transfère uniquement les blocs de données modifiés du disque virtuel (`vm-100-disk-0`) vers le `zfs-pool` des autres nœuds.
- **Avantage** : Si le nœud principal s'éteint brutalement, les nœuds secondaires possèdent déjà une copie quasi identique du disque de la VM sur leur propre stockage local.

> [!caution]
> La réplication ZFS est **asynchrone**. En cas de panne soudaine du serveur maître, les données modifiées entre la dernière réplication et la panne seront perdues (définies par votre RPO / *Recovery Point Objective*).
<br><br>

## Haute Disponibilité (HA) & Quorum

La Haute Disponibilité Proxmox garantit le redémarrage automatique d'une machine virtuelle ou d'un conteneur en cas d'extinction ou de crash d'un nœud physique.

Elle s'appuie sur trois composants fondamentaux :
1. **Quorum (Corosync)** : Le cluster doit maintenir la majorité absolue des votes des nœuds (ex: 2 nœuds en ligne sur 3). Si le quorum est perdu, l'accès au HA est verrouillé par sécurité pour éviter l'effet *Split-Brain*.
2. **CRM (Cluster Resource Manager)** : Élu parmi les nœuds du cluster, il orchestre l'état des ressources HA et décide sur quel nœud basculer une VM en panne.
3. **LRM (Local Resource Manager)** : Présent sur chaque nœud, il exécute les ordres du CRM (démarrer, arrêter, migrer une VM) et gère le composant *Watchdog* de sécurité (*Fencing*).

| Composant HA | Rôle dans le Cluster |
| :--- | :--- |
| **Corosync** | Gère le vote, les battements de cœur (*heartbeats*) et le Quorum |
| **pve-ha-crm** | Décide du basculement d'une VM vers un nœud sain |
| **pve-ha-lrm** | Pilote le démarrage/arrêt local et isole le nœud défaillant (*Fencing*) |

> [!tip]
> Pour que le basculement HA d'une VM soit quasi instantané et sans erreur de stockage, le disque de la VM **doit obligatoirement** se trouver sur un storage partagé (Ceph/NFS/iSCSI) OU sur un stockage local ZFS répliqué sur tous les nœuds cibles !
<br><br>

---

# En pratique

Nous allons maintenant suivre les étapes concrètes de configuration sur notre cluster `FYC-Cluster` composé de 3 nœuds : `fyc-proxmox-1`, `fyc-proxmox-2` et `fyc-proxmox-3`.
<br><br>

## Étape 1 : Préparation et création du Pool ZFS

### 1. Initialisation des disques en GPT
Avant de créer le pool ZFS, assurez-vous que les disques secondaires sur chaque nœud sont initialisés avec une table de partition GPT.

Rendez-vous dans : `Votre-Nœud` --> `Disks` --> Sélectionnez le disque (ex: `/dev/sdb`) --> Cliquez sur **Initialize Disk with GPT**.

<img width="1200" alt="Formater disque" src="./Formater disque.png" />
<br><br>

### 2. Création du pool ZFS (`zfs-pool`)
Une fois le disque initialisé, naviguez vers `Votre-Nœud` --> `Disks` --> `ZFS` et cliquez sur **Create: ZFS**.

<img width="1200" alt="creer pool zfs" src="./creer pool zfs.png" />

Complétez le formulaire d'initialisation du ZFS :
- `Name` : `zfs-pool` (*Attention : le nom doit être strictement identique sur tous vos nœuds PVE*)
- `RAID Level` : `Single Disk` (ou RAID1/RAID10 selon votre infrastructure matérielle)
- `Compression` : `on` (fortement conseillé pour optimiser les I/O et l'espace disque)
- `ashift` : `12` (alignement standard pour disques modernes / SSD à secteurs 4K)
- `Device` : Cocher `/dev/sdb` (ou les disques destinés au stockage ZFS)

<img width="700" alt="choisir disque" src="./choisir disque.png" />
<br><br>

### 3. Vérification de la propagation du ZFS Pool
Répétez l'opération sur les nœuds `fyc-proxmox-2` et `fyc-proxmox-3`. Vérifiez ensuite dans l'arborescence globale du Datacenter que le stockage `zfs-pool` apparaît bien sous chaque nœud :

<img width="400" alt="verif zfs-pool" src="./verif zfs-pool.png" />
<br><br>

### 4. Attribution du stockage ZFS à la Machine Virtuelle
Lors de la création de votre VM (ou via l'onglet Hardware), déplacez le disque virtuel sur le pool ZFS nouvellement créé.

Dans notre cas, la VM `100 (Debian1)` a son disque principal `scsi0` configuré sur `zfs-pool:vm-100-disk-0`.

<img width="1100" alt="vm stockage zfs" src="./vm stockage zfs.png" />
<br><br>

---

## Étape 2 : Configuration de la Réplication ZFS

Pour permettre au HA de basculer la VM `100` vers un autre nœud en cas de crash, nous devons synchroniser régulièrement son disque avec `fyc-proxmox-2` et `fyc-proxmox-3`.
<br><br>

### 1. Ajout des tâches de réplication
Sélectionnez la VM `100 (Debian1)` --> Onglet `Replication` --> Cliquez sur **Add**.

<img width="1000" alt="replication" src="./replication.png" />

Renseignez les champs du job :
- `CT/VM ID` : `100`
- `Target` : Sélectionnez le nœud cible (ex: `fyc-proxmox-2`)
- `Schedule` : `*/30` (ex: toutes les 30 minutes, ou `*/5` pour toutes les 5 minutes)
- `Rate limit (MB/s)` : `unlimited` (ou réduisez si vous souhaitez préserver la bande passante réseau)
- `Enabled` : Coché

<img width="450" alt="replication vers noeud 2" src="./replication vers noeud 2.png" />

> [!note]
> Créez un deuxième job de réplication pour cibler également le troisième nœud (`fyc-proxmox-3`).
<br><br>

### 2. Validation des tâches de réplication
Vérifiez dans le tableau que les réplications affichent l'état `Status: OK` et qu'une première synchronisation a eu lieu :

<img width="1200" alt="config replication ok" src="./config replication ok.png" />
<br><br>

### 3. Inspection des disques répliqués sur les nœuds distants
Rendez-vous sur les nœuds `fyc-proxmox-2` et `fyc-proxmox-3` sous `zfs-pool` --> `VM Disks`. Vous devez constater la présence du disque virtuel répliqué `vm-100-disk-0` (30.00 GiB) en attente :

| Disque répliqué sur `fyc-proxmox-2` | Disque répliqué sur `fyc-proxmox-3` |
| :---: | :---: |
| <img width="600" alt="verif proxmox 2 replication" src="./verif proxmox 2 replication.png" /> | <img width="600" alt="verif proxmox 3 replication" src="./verif proxmox 3 replication.png" /> |

<br><br>

---

## Étape 3 : Activation et configuration de la Haute Disponibilité (HA)

La synchronisation des disques étant opérationnelle, nous pouvons armer la Haute Disponibilité au niveau de notre Datacenter.
<br><br>

### 1. Armer le service HA du Datacenter
Allez dans `Datacenter` --> `HA` --> Cliquez sur le bouton **Arm HA**.

<img width="1100" alt="ACTIVER HA" src="./FYC/ACTIVER HA.png" />

Vérifiez les indicateurs du status HA :
- `quorum` : `OK`
- `fencing` : `armed (CRM watchdog active)`
- `master` : Affiche le nœud élu maître (ex: `fyc-proxmox-2`)

<img width="1100" alt="ha activé" src="./FYC/ha activé.png" />
<br><br>

### 2. Ajouter la ressource VM au gestionnaire HA
Naviguez dans `Datacenter` --> `HA` --> Section **Resources** --> Cliquez sur **Add**.

<img width="900" alt="menu - configuration de la ressource vm 100" src="./FYC/menu - configuration de la ressource vm 100.png" />

Configurez la ressource comme suit :
- `VM` : `100` (Debian1)
- `Max. Restart` : `1` (nombre de tentatives de redémarrage sur place avant migration)
- `Max. Relocate` : `1` (nombre de basculements autorisés vers un autre nœud)
- `Request State` : `started`
- `Failback` : Coché
- `Auto-Rebalance` : Coché

<img width="600" alt="configuration de la ressource vm 100" src="./FYC/configuration de la ressource vm 100.png" />

Une fois ajoutée, la ressource apparaît dans l'état `started` assignée au nœud d'origine `fyc-proxmox-1` :

<img width="1000" alt="ressource ajouté vm 100" src="./FYC/ressource ajouté vm 100.png" />
<br><br>

---

## Étape 4 : Tests de basculement (Failover) & Simulation de pannes

Afin de valider l'efficacité de notre Haute Disponibilité ZFS + HA, nous allons simuler des pannes matérielles.
<br><br>

### Test 1 : Panne du nœud maître `fyc-proxmox-1`
Nous coupons brutalement le premier nœud (`fyc-proxmox-1`) sur lequel s'exécute la VM `100`.

1. Dans l'interface Proxmox, le nœud `fyc-proxmox-1` passe en échec (icône rouge `x`).
2. Le gestionnaire HA détecte la rupture des battements de cœur et passe le LRM du nœud 1 en statut `dead`.

<img width="1100" alt="active" src="./FYC/active.png" />

3. Le CRM isole le nœud défaillant (*Fencing*) puis réassigne automatiquement la VM `100` au nœud répliqué disponible `fyc-proxmox-3`.
4. La VM `100 (Debian1)` démarre automatiquement sur `fyc-proxmox-3` sans intervention humaine !

<img width="400" alt="bien swap" src="./FYC/bien swap.png" />
<br><br>

### Vérification par les logs CRM (`pve-ha-crm`)
En consultant les logs du service HA sur le nœud master via la commande `journalctl -u pve-ha-crm -f`, on observe la séquence exacte du basculement :

<img width="1100" alt="logs basculement" src="./FYC/logs basculement .png" />

```text
Sep 08 15:29:23 fyc-proxmox-2 pve-ha-crm[1409]: service 'vm:100': state changed from 'started' to 'fence'
Sep 08 15:29:23 fyc-proxmox-2 pve-ha-crm[1409]: node 'fyc-proxmox-1': state changed from 'unknown' => 'fence'
Sep 08 15:30:33 fyc-proxmox-2 pve-ha-crm[1409]: fencing: acknowledged - got agent lock for node 'fyc-proxmox-1'
Sep 08 15:30:33 fyc-proxmox-2 pve-ha-crm[1409]: service 'vm:100': state changed from 'fence' to 'recovery'
Sep 08 15:30:33 fyc-proxmox-2 pve-ha-crm[1409]: recover service 'vm:100' from fenced node 'fyc-proxmox-1' to node 'fyc-proxmox-3'
Sep 08 15:30:33 fyc-proxmox-2 pve-ha-crm[1409]: service 'vm:100': state changed from 'recovery' to 'started' (node = fyc-proxmox-3)
```

> [!tip]
> Le journal démontre la transition d'état claire du CRM : `started` -> `fence` -> `recovery` -> `started (sur fyc-proxmox-3)`.
<br><br>

### Test 2 : Cascade de pannes ("Coupé décalé" du nœud 3 vers le nœud 1)
Nous simulons ensuite la perte successive du nœud `fyc-proxmox-3` alors que la VM y a été basculée, pendant que `fyc-proxmox-1` est remis en ligne.

1. `fyc-proxmox-3` s'éteint et passe en statut `old timestamp - dead?`. La ressource `vm:100` passe immédiatement en statut `fence`.

<img width="1100" alt="coupé décalé part 1" src="./FYC/coupé décalé noeud 3 vers noeud 1 - part1.png" />

2. Le CRM bascule de nouveau la charge vers le nœud réhabilité `fyc-proxmox-1`.
3. La VM `100` est relancée avec succès sur `fyc-proxmox-1` !

<img width="1100" alt="coupé décalé part 2" src="./FYC/coupé décalé noeud 3 vers noeud 1 - part2.png" />

**BINGO ! La Haute Disponibilité ZFS répliquée est totalement opérationnelle et tolère les pannes en cascade !**
<br><br>

---

# Erreurs connues & Troubleshooting

### 1. La VM refuse de basculer en HA et affiche une erreur de stockage
- **Cause** : Le pool ZFS n'a pas exactement le même nom sur le nœud cible, ou la réplication n'a jamais été exécutée initialement.
- **Solution** : Vérifiez que `zfs-pool` est bien créé sur tous les nœuds avec le même nom et forcez une réplication manuelle via `Replication` --> **Schedule now**.

### 2. Le cluster perd le Quorum lors de l'arrêt d'un nœud
- **Cause** : Cluster composé de seulement 2 nœuds sans périphérique de vote externe.
- **Solution** : Ajoutez un nœud léger (ou installez un `qdevice` Corosync sur une petite VM ou un Raspberry Pi) pour garantir un nombre d'électeurs impair.

### 3. Erreur de réplication ZFS verrouillée (*Dataset is locked*)
- **Cause** : Un job de réplication s'est interrompu brutalement lors d'un crash réseau.
- **Solution** : Supprimez le snapshot temporaire bloqué via CLI sur le nœud source :
  ```bash
  zfs release -r pve-repl-lock zfs-pool/vm-100-disk-0
  ```
