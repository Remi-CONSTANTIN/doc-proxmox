# Introduction
Cette documentation porte sur l'outil Proxmox Datacenter Manager (PDM) dans sa version 1 (1.1).
En l'état actuel des choses, PDM ne se positionne pas comme un équivalent à vCenter. L'interface de gestion de cluster PVE est beaucoup plus mature pour cet usage.

# Installation
N'étant pas un composant indispensable du cluster, il n'est pas obligatoire de lui dédier une machine physique.
Gardez toutefois à l'esprit que si le cluster sur lequel vous le mettez vient à être indisponible pour une quelconque raison, vous perdez l'accès à l'interface.

PDM se monte comme un PVE ou PBS. Il vous suffit de télécharger l'ISO sur le site officiel et de préparer un support bootable avec, puis de démarrer dessus sur la machine dédiée.

# Configuration

## Mise à jour
Une fois l'installation faite, il vous faudra configurer les dépôts Proxmox pour recevoir les mises à jour.
Par défaut, ce sont les dépôts Entreprise, nécessitant un abonnement, qui sont configurés (comme pour PVE et PBS), mais cela est bien sûr remplaçable par leur dépôt gratuit `pdm-no-subscription`.

Pour ce faire, rendez-vous dans `Administration` --> `Repositories`.
Si vous n'avez pas d'abonnement, sélectionnez le dépôt et désactivez-le avec le bouton au-dessus.  
<img width="760" height="48" alt="pdm_entreprise_repo" src="https://github.com/user-attachments/assets/e933a099-9336-45d4-a5d2-5c6dc56c989c" />  
Puis cliquez sur `Add` et ajoutez le dépôt `No-subscription`.  
<img width="759" height="48" alt="pdm_no-subscription_repo" src="https://github.com/user-attachments/assets/95010f2b-f685-4122-aad0-95fe898bdd2f" />  
Et pour finir, profitez-en pour faire les mises à jour dans `Administration` --> `Updates`.

## Connexion à un cluster PVE
L'intérêt de PDM étant d'avoir la main sur des clusters PVE, voyons comment en ajouter un.

Allez dans `Remotes` --> Cliquez sur `Add` --> Sélectionnez `Proxmox PVE`.
Vous avez maintenant à compléter plusieurs onglets d'informations :

### Onglet `Probe remote`
- `Server address` : IP/hostname d'une des machines du cluster PVE cible au format `<IP/hostname>:port`
- `Fingerprint` : Empreinte SHA-256 du certificat du PVE cible dans le cas où vous ne gérez pas les certificats avec une autorité de certification. Vous la trouverez dans le menu `System` --> `Certificate` --> Selectionnez sur `pve-ssl.pem` --> Cliquez sur `View Certificate` --> Copiez le `Fingerprint`  
<img width="966" height="476" alt="pve_fingerprint" src="https://github.com/user-attachments/assets/88d347b7-fcac-4af1-8e58-53130a3ed545" />  

### Onglet `Settings`
- `Remote ID` : Nom d'affichage du cluster distant dans l'interface PDM

L'authentification sur le cluster PVE se fait via un token. Si vous en avez déjà un pour PDM, alors sélectionnez `Use existing Token` et complétez avec vos informations.

Si ce n'est pas le cas, il vous faudra donner un compte admin sur le cluster pour que PDM le crée lui-même.
> [!caution]
> Le token héritera donc des permissions du compte utilisé ici ! Évitez d'utiliser root !
- `User` : Nom d'utilisateur pour se connecter au cluster
- `Password` : Mot de passe de l'utilisateur pour se connecter au cluster
- `Realm` : Domaine d'authentification associé à l'utilisateur renseigné (`pam` pour root, `pve` pour un compte Proxmox)
- `API Token Name` : Nom du token API créé dans le cluster PVE

### Onglet `Endpoints`
Vous devez maintenant préciser les nœuds PVE que PDM peut utiliser pour se connecter au cluster.  
Normalement, PDM les remonte tous automatiquement, mais il vous faudra possiblement corriger les informations.  
Par exemple, dans mon cas, PDM remonte le hostname de mes nœuds, mais ils ne sont pas résolvables via DNS, donc je les remplace par leur `IP:port`.  
<img width="609" height="358" alt="Endpoints_pve_for_PDM" src="https://github.com/user-attachments/assets/421990bf-7cf8-43f7-b3ef-08371e18c54d" />  
(Ne faites pas attention au petit icône de mon KeePassXC.)

### Onglet `Summary`
Un simple onglet pour récapituler certaines informations.
Vous pouvez continuer et valider la connexion, elle devrait s'afficher sous vos yeux.
<img width="707" height="136" alt="cluster-1_pdm_connexion" src="https://github.com/user-attachments/assets/b1a2d1ee-68ca-4602-8aac-29e2b7e867e3" />  
(Vous remarquerez que j'ai utilisé root dans le cadre de ce LAB, mais mon précédent avertissement tient toujours.)

PDM va maintenant rapatrier des informations du cluster et vous pourrez interagir avec celui-ci sur certains points.

## Connexion à un PBS
La connexion d'un PBS suit la même logique que le PVE aux seules différences que :
- Vous n'aurez pas l'onglet `Endpoints` car PBS ne fonctionne pas en cluster
- Et que l'empreinte du certificat est aussi affichable dans le menu `Dashboard`
