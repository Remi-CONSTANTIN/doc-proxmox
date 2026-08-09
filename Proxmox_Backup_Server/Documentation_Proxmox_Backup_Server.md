# Installation
PBS se monte un peu comme un PVE. C'est à dire qu'il vous faudra une clé USB bootable et un serveur physique pour l'installer. L'installeur est le même que PVE.

Dans un contexte d'entreprise les serveurs de sauvegarde sont la plupart du temps physiques. Si vous vous tentez à le virtualiser sur votre cluster et que celui-ci vient à ne plus être accessible, vous vous serez juste mis des battons dans les roues pour une éventuelle restauration de données.


# Configuration

### Mise à jour
Une fois installé il est conseillé de vérifier la configuration des dépôts.
Dans une installation fraîche, 3 dépôts sont présent :
- Les deux premiers sont relatifs à Debian, car, en effet PBS est basé sur Debian tout comme PVE
- Le troisième concerne le dépôt PBS lui même. Si vous avez souscrit à un abonnement chez proxmox alors laissez le dépôt Entreprise. Dans le cas où vous l'avez monté seulement pour un Home-Lab alors désactivez le et ajouter le `pbs-no-subscription`
Profitez-en pour mettre à jour la machine.
<img width="1726" height="427" alt="apt_repo_pbs" src="https://github.com/user-attachments/assets/eae5b5df-22d2-459a-b211-36837d8da053" />

---

### Création d'un datastore

#### 1.a Ajout d'un/des disque(s)
Après l'installation de PBS sur le disque système de votre machine, il nous faut ajouter un/des disque(s) afin d'y stocker les sauvegardes.

Il sera de votre initiative de choisir entre l'ajout de disques physiques directement sur la machine ou d'utiliser l'ISCSI pour simuler le comportement.
Les partages réseaux NFS ou SMB sont déconseillés par Proxmox car ceux-ci ne sont pas fait pour ingérer des millions de chunks générés par PBS.

Une fois le/les disque montés, rendez-vous dans `Administration` --> `Storage/disks` afin de vérifier si le nouveau support est détecté. 

#### 1.b -  Création du datastore

- Dans un contexte d'entreprise, le standard est ZFS car celui-ci intègre la correction automatique des bits corrompus (bit-rot). Il gère lui-même le RAID donc pas besoin de RAID matériel. De plus, il y est toujours extrêmement rapide de faire des snapshots des données qu'importe la volumétrie.

> [!NOTE]
> Contrairement aux sauvegardes classiques, PBS découpe les données en millions de petits blocs (chunks de 4 Mo). Les tâches de maintenance comme le Garbage Collection ou les Verify Jobs doivent lire les métadonnées de chacun de ces blocs, ce qui demande énormément d'IOPS
> Proxmox recommande d'utiliser des disques SSD ou NVMe pour encaisser cette charge
> Si vous utilisez des disques mécaniques (HDD) : En raison de leur lenteur, les tâches de maintenance peuvent prendre des dizaines d'heures. L'astuce consiste à utiliser ZFS pour y ajouter un "Special Device" (un SSD dédié). ZFS y stockera uniquement les métadonnées : lors du Garbage Collection, le système les lira instantanément sur le SSD, préservant ainsi vos disques mécaniques.

- Dans le cas d'un Home-lab, vous n'avez qu'à créer un nouveau `Directory` à partir de ce disque (dans l'onglet `Directory)
Pour ce qui est des paramètres, on se contentera de :
1. `Filesystem` : ext4 (léger et peut être réduit si besoin contrairement au ZFS qui est lourd et à l'XFS qui ne peut être diminué)
2. `Add as datastore` : Coché (Obligatoire pour qu'il apparaisse comme stockage disponible)
3. `Removable datastore` : Décoché (cela ne concerne que les supports qui ont pour vocation à être débranché comme un disque externe, cela ne nous concerne pas ici)

Vous avez donc maintenant un nouveau datastore disponible pour y stocker des sauvegardes !

---

### Connexion au cluster Proxmox
Une fois le PBS configuré, il faut maintenant rendre accessible le nouveau datastore au cluster Proxmox comme emplacement de sauvegarde.

Pour ce faire, rendez-vous dans `Datacenter` --> `Storage` --> Cliquez sur `Add` --> Sélectionnez `Proxmox Backup Server`
Vous avez maintenant plusieurs informations à remplir dans plusieurs onglets :

**Onglet `General`**
1. `ID` : Le nom d'affichage du nouveau datastore dans votre cluster (peut être différent du nom dans PBS)
2. `Server` : IP du PBS
3. `Username` : Nom d'utilisateur pour se connecter au PBS
4. `Password` : Mot de passe de l'utilisateur pour se connecter au PBS
5. `Datastore` : Nom exact du datastore dans PBS
6. `Fingerprint` : Empreinte trouvable sur le dashboard de votre PBS (bouton `Show fingerprint)

**Onglet `Backup Retention`**
Comme le petit message l'indique quand vosu cliquez dessus, il n'est pas conseillé de configurer la rétention des backups ici. Préférrez plutôt le faire sur le job de sauvegarde ou sur le datastore directement.

**Onglet `Encryption`**
Cet onglet permet d'ajouter une couche de sécurité sur les données en chiffrant celles-ci côté hyperviseur avantde les envoyer au PBS
Par défaut, cela est désactivé mais peut être activé simplement en sélectionnant `Auto-generate a client encryption key` pour générer une clé automatiquement.  
Il est aussi possible d'uploader une clée qu'on aura généré sur le PBS directement dans le menu `Configuration` --> `Encryption Keys`  
Si vous choisissez de l'activer, vous verrez dans votre datastore, un icône de cadenas à côté de vos sauvegardes  
<img width="3720" height="226" alt="encryption_logo_pbs" src="https://github.com/user-attachments/assets/ca6df439-e1be-47d8-a7ba-8282adcc7bac" />


Une fois cela fait, vous devriez voir le nouveau datastore sous chacun de vos nœuds de votre cluster.
Pour planifier vos sauvegardes, le procédé ne change pas avec ou sans PBS, tout se passe toujours dans l'onglet `Backup` dans `datacenter`.  
<img width="175" height="422" alt="pbs_datastore" src="https://github.com/user-attachments/assets/06d57ca4-a3dd-4e59-b0f8-e7351c6c9fb9" />

---

### Redondance
La redondance d'un PBS n'est pas la même qu'au sens d'un cluster proxmox. C'est à dire que si on déploie deux PBS, les deux ne pourront pas se répartir la charge ni être configurés en Fail-Over. 
A la place on peut déployer deux PBS sur deux sites différents où celui qui est distant va venir tirer les sauvegardes des machines afin de prévenir la perte de données à cause d'un événement sur le site principal.

Cette architecture en `pull` permet d'éviter que le PBS distant soit accessible via le PBS local et ainsi qu'un potentiel pirate remonte la chaine des PBS. Le PBS local n'a aucune information concernant le distant car c'est celui-ci qui vient tirer lui même les données.

Pour faire cela, allez sur le **PBS distant** qui va tirer les sauvegardes depuis le PBS local et rendez-vous dans `Configuration` --> `Remotes` --> CLiquez sur Add et complétez les informations requises :  
1. `Remote ID` : Nom donné au PBS principal (purement de l'affichage dans le PBS distant)  
2. `Host` : IP du PBS distant  
3. `Auth ID` : Compte pour se connecter au PBS principal  
4. `Password` : Mot de passe du compte pour se connecter au PBS principal  
5. `Fingerprint` : (Optionnel) Empreinte du certificat autosigné du PBS principal si vous n'avez pas utilisé une autorité de certification   
6. `Comment` : Un simple commentaire sur cette connexion  

Une fois connecté, le PBS principal devrait s'afficher  
<img width="604" height="89" alt="remote_pbs" src="https://github.com/user-attachments/assets/0e11dd60-30cb-4c90-917a-1319b535dd39" />

---

### Fonctions utiles
Si on ne va pas plus loin dans la réflexion on pourrait se dire que PBS n'apporte rien de plus par rapport à une sauvegarde sur un emplacement classique.
Mais en réalité, nous avons plusieurs gros avantages : 

#### Economie de stockage
Là où l'outil natif de Proxmox ne sait faire que des sauvegardes complètes des serveurs, PBS, lui, nous offre l'incrémentale. Pour rappel, celle-ci permet de repérer les données modifiées depuis la dernière sauvegarde afin de ne stocker que celle-ci pour la nouvelle.

De plus, quand un parc informatique possède plusieurs machines du même OS, cela ne sert  rien de sauvegarder 10 fois le même système de fichiers. PBS peut donc voir que certains blocs de données existe déjà et ne va donc pas les sauvegarder.

#### Restauration de fichiers
Si vous regardez les fichiers de sauvegarde dans votre nouveaux datastore PBS, vous remarquerez que vous avez maintenant un bouton `File Restore`. Vous avez en fait la possibilité d'explorer le système de fichiers de votre sauvegarde afin de restaurer un fichier en particulier...ou presque.

En effet, via la GUI, vous avez seulement la possibilité de télécharger le fichier/dossier mais vous devrez le restaurer manuellement sur la machine !
Toutefois, la CLI fournit un outil en ligne de commande appelé `proxmox-backup-client`. Il permet de monter une archive de sauvegarde directement comme un lecteur réseau dans le système d'exploitation de la VM ou sur un hôte, ce qui permet de copier-coller les fichiers directement là où ils doivent aller
Cela reste extrêmement utile mais n'est pas aussi automatisé que le bouton le laisse penser.  
<img width="557" height="433" alt="file_restor_pbs" src="https://github.com/user-attachments/assets/53e8a9e4-d688-42f6-8cf1-44d9e4d0bfa0" />

#### Vérification de l'intégrité des sauvegardes 
Dans le cas d'une sauvegarde sans PBS, l'outils natif de proxmox n'est pas capable de savoir si les fichiers de backups ne sont pas corrompus et utilisables pour remettre en état de marche une machine.
C'est là que PBS intervient en vérifiant le hash de chaque chunks de la sauvegarde afin de garantir qu'aucun n'ai changé à cause d'un éventuel `bit-rot` des disques dures. La sauvegarde passe donc dans un état corrompu puis à la prochaine backup, le chunk corrompu est remplacé par un sain.

PBS propose donc une vérification et une réparation automatique des sauvegardes afin de garantir de la restaurabilité des données en cas de besoin.

Cette tâche se paramètre directement sur le datastore visé dans le PBS `Datastore`--> Votre datastore --> `Verify Job` --> Cliquez sur Add et complétez les informations : 
1. `Namespace` : Il est possible de séléctionner seulement certaines sauvegardes du datastore en choisissant certains namespaces. Si vous n'avez rien configuré de tel, `Root` est le choix par défaut prenant toutes les sauvegardes
2. `Max. Depth` : Définit jusqu'où PBS doit descendre dans l'arborescence des Namespaces. "Full" signifie qu'il vérifiera tout, incluant tous les sous-dossiers, quel que soit leur niveau de profondeur
3. `Schedule` : L'intervalle de relance du job
4. `Skip Verified` : Permet d'éviter de re-vérifier des sauvegardes
5. `Re-Verify After` : Permet de re-vérifier certaines sauevegardes même s'il elles ont déjà été vérifiée. Utile pour les vieilles backups

> [!important]
> Cela peut etre un processus très long lors de la première execution et encore plus sur des HDD. Les suivants seront bien plus courts car ils ne feront que vérifier les nouvelles sauvegardes

#### Sécurité
Et pour finir, avant de quitter l'hyperviseur, les backups **peuvent être** chiffrées en AES-256. Cela implique donc que si le PBS seul est compromis, le pirate ne verras qu'une bouillis de données indéchiffrable.
De plus la possibilité d'utiliser un `removable device`, ou des bandes magnétiques, comme emplacement de stockage permet une sauvegarde hors-ligne très facile.

#### Namespaces
Depuis quelques versions, PBS intègre la notion de Namespaces. C'est très utile en contexte d'entreprise : cela permet de ranger les sauvegardes de différents clusters PVE ou différents départements dans le même Datastore sans que tout soit mélangé à la racine, tout en conservant la déduplication globale.


# Conclusion
Proxmox Backup Server fournis fiabilité, sécurité et praticité et s'intègre parfaitement à l’écosystème Proxmox sans entraîner de surcoûts de licences.

Mais à mon avis, le principal manque de la solution se trouve dans la fonctionnalité de `File restor` qui manque d'une vraie automatisation dans la restauration de fichiers/dossiers.
