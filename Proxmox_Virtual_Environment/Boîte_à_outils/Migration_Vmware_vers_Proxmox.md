# Introduction
Ce tutoriel a été conçu dans mon Home-Lab, il ne prends donc pas en compte toutes les contraintes d'un contexte professionnel. Voyez le plutôt comme un décorticage de l'outils 

# Mon contexte de test
- 1 cluster Proxmox VE 9
- 1 ESX 8

# Objectif
Migrer une/des VM(s) Linux/Windows de notre ESX vers notre cluster Proxmox

# Prérequis
1. Une VM Windows/Linux **éteinte** sur votre ESX 
2. Flux `HTTPS` ouvert entre les nœuds PVE et le/les ESX (dans le sens `Proxmox` --> `ESX`). 

# Procédure

#### Préparation du cluster Proxmox
1. Vérifiez que les flux entre le cluster et l'ESX sont ouverts
2. Connectez-vous sur le cluster Proxmox
3. Rendez-vous dans `Datacenter` --> `Storage` --> Cliquez sur "Add" --> `ESXi`
	Complétez les informations suivantes : 
- `ID` : Nom d'affichage de l'ESX dans l'interface Proxmox
- `Server` : IP de l'ESX
- `Username` : Nom d'utilisateur pour se connecter à l'ESX
- `Password` : Mot de passe de l'utilisateur servant à la connexion à l'ESX
- `Nodes` : Si vous désirez limiter l'accès au stockage à certains nœuds
- `Enable` : Cochez pour activer le stockage
- `Skip Certificate Verification` : Pour désactiver la vérification du certificat de l'ESX si il ne provient pas de l'autorité de certification de votre domaine
Une fois tout cela complété et validé, le stockage devrait apparaître sous chacun des nœuds
<img width="201" height="424" alt="esxi_datastore" src="https://github.com/user-attachments/assets/d4b0c6f9-873d-45a8-97bc-3c50b381fd84" />

#### Migration
1. Vérifiez que la machine est éteinte sur l'ESX
2. Retournez sur le cluster proxmox puis allez dans le stockage nouvellement créé
3. Vous avez devant vous la liste des VMs. Vous remarquerez que leur datastore apparaît dans leur nom
4. Sélectionnez la VM à transférer et cliquez sur `Import`, vous aurez accès aux paramètres de la VM adaptés à proxmox par rapport à ceux sur l'ESX.
<img width="815" height="440" alt="vm_import_from_esx_to_proxmox" src="https://github.com/user-attachments/assets/9c886f1e-49af-4a9c-bf66-de1b67dff8d4" />


Vérifiez donc que les caractéristiques sont bonnes dans les deux onglets :
###### Onglet "General" :
- `VM ID` : ID de la VM sur le cluster Proxmox
- `Sockets` : Nombre de cœurs physique
- `Cores` : Nombre de cœurs logiques
- `Memory` : Allocation RAM
- `Name` : Nom de la VM
- `CPU Type` : Type de CPU virtuel. Il se peut que dans certains cas, il faille changer le choix par défaut mais `x86-64-v2-AES` offre le plus de compatibilité
- `OS Type` :  Type d'OS de la machine (la plupart du temps Linux ou Windows)
- `Version` : Version du noyau pour Linux et version de l'OS pour Windows
- `Default Storage` : **Important** car n'est pas forcément le bon si vous avez des datastores CEPH et/ou sur le cluster
- `Format` : Format des fichiers de la VM sur le datastore. La plupart du temps RAW (brut) car géré par le datastore
- `Default Bridge` : Le bridge par défaut des cartes réseau de la VM. Cela reste modifiable dans l'onglet "Advanced" juste après
- `Live import` : Permet de démarrer la VM avant même que tout les disques aient finis d'être importés. Cela permet de minimiser le temps d'indisponibilité sur les grosse machines.
> [!WARNING]
> Dans tout les cas il y aura coupure car la machine doit être **arrêtée** sur l'ESX. De plus les performances en sont **extrêmement dégradées**. Et pour finir, cela nécessite une connexion réseau stable, car en cas de coupure vous risquez le **plantage de la machine**

**Onglet "Advanced" :**
- `Disks` : Les paramètres des disques proposés correspondent à ceux sur l'ESX. Il n'est pas possible de changer la taille ici, mais juste de définir, au cas par cas, dans quel datastore ils vont atterrir et s'il faut les activer
- `SCSI Controller` : Le contrôleur par défaut est calibré sur `VMWare PVSCSI` afin de se rapprocher au maximum des paramètres de l'ancien environnement de la machine pour éviter les plantages
- `CD/DVS Drives` : L'outil de migration ne supporte pas l'importation des disques attachés à la machine côté ESX mais vous propose à la place d'en ajouter un juste après l'importation
- `Network Interfaces` : Comme pour les disques, proxmox propose un modèle de carte réseau VMWare `vmxnet3` pour offrir la meilleure compatibilité. En l’occurrence, c'est le même modèle virtuel que sur un ESX. Cela reste modifiable après import
- `Unique MAC addresses` : Par défaut l'adresse MAC de la carte réseau ESX est gardée mais il est possible d'en générer une nouvelle en cochant l'option. Il peut être utile de garder la même pour certains logiciels à licence

Le transfert démarre, plus qu'à attendre la fin !

# Post-migration
L'outil importe la machine telle quelle, avec ses anciens paramètres. Pour garantir des performances optimales et une bonne communication avec Proxmox, voici les actions à réaliser une fois la VM démarrée :
- Désinstallez les anciens utilitaires (`VMware Tools` sur Windows, `open-vm-tools` sur Linux).
- Installez le paquet `qemu-guest-agent` dans la VM pour que Proxmox puisse lire son IP et l'éteindre proprement
- Installez les pilotes `VirtIO` puis modifiez la configuration matérielle de la VM (réseau, disques) pour remplacer la carte réseau `vmxnet3` par `VirtIO (paravirtualized)`
- Si vous avez beaucoup de machines, ces étapes répétitives sont d'excellentes candidates pour être automatisées avec Ansible (via AWX par exemple ?)


# Conclusion
Cet outil de transfère offre en général une bonne expérience d'utilisation grâce au fait qu'il soit natif et donc très bien intégré à l'environnement PVE. Il permet aussi de simplifier des processus qui pourraient générer beaucoup d'erreurs manuels (même si l'import n'est pas automatique). Le `Live import` permet de grandement réduire les temps interruption même s'il faut quand même s'assurer que l'infrastructure réseau tient la route. On appréciera aussi la "traduction" intelligente du matériel afin d'avoir la meilleure compatibilité avec le nouvel environnement proxmox.

Toutefois tout n'est pas parfait. Le transfère des pilotes VMWare vers ceux de proxmox n'est pas automatique et nous restons très dépendant du réseau, même si la plupart des solutions qu'on pourra trouver aurons les mêmes désavantages.
