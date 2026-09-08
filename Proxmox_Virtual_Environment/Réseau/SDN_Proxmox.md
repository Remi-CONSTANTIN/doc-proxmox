# Introduction
L'objectif ici va être de décortiquer le SDN proposé par Proxmox.
Nous allons tout d'abord essayer d’appréhender le principe théorique de chacun de ses composants puis nous verrons son utilisation concrète afin de comprendre l'interface pas toujours très intuitive de PVE

---

# La théorie
Il aurait été possible de se lancer directement dans la pratique mais le SDN comporte un nombre conséquent de fonctionnalités nécessitant d'avoir une base théorique afin de savoir comment les configurer.  
Si vous n'avez pas bien suivi vos cours de réseau, cela est d'autant plus vrai...
<br><br>

## Zones
Les zones définissent le protocole utilisé par les switchs virtuels que nous allons créer à l'intérieur (VNet). Elles définissent comment les données vont voyager.  
Cela peut paraître abstrait pour l'instant mais vous comprendrez en suivant ce cours.  

### Simple
Utiliser un Linux bridge (vmbr) ou un VNet issu d'une zone simple est assez similaire sur certains points, mais en y regardant de plus près, on remarque que le SDN arrive à tirer son épingle du jeu.  

Pour comprendre cela, comparons-les : 
|Linux bridge|Zone simple|
|------------|-----------|
|Permet de créer un réseau isolé|Idem|
|Ne permet pas de faire communiquer des VMs situées sur des PVE différents|Idem|
|Peut donner un accès à internet en l'associant à une carte physique|Peut donner un accès à internet en activant le `SNAT` dans le `subnet` de son `VNet`|
|Pour avoir le même "vmbr" sur tous vos nœuds, il faut tous les créer à la main|Doit être configuré dans l'onglet Datacenter puis est répliqué sur tous les nœuds|
|Nécessite une machine connectée au "vmbr" pour avoir le DHCP|Profite du DHCP et de l'IPAM du SDN Proxmox|

En résumé, les deux sont assez similaires dans les fonctionnalités qu'ils proposent mais le SDN apporte quelques fonctionnalités supplémentaires non négligeables. L'IPAM intégré est particulièrement utile pour les laboratoires.
<br><br>

### VLAN
Contrairement à la zone simple ou au Linux bridge qui créent un réseau local au PVE, le Linux Bridge (tagué) et la zone VLAN n'isolent pas eux-mêmes les flux et délèguent cela à l'infrastructure réseau de l'entreprise.  
Comme pour la zone simple, la zone VLAN ne révolutionne rien mais vient apporter quelques avantages liés au SDN.

Encore une fois, une comparaison des deux sera plus simple pour comprendre : 
|Linux Bridge|Zone VLAN|
|-|-|
|Permet de taguer les flux réseau d'une machine|Idem|
|Nécessite de taguer la carte réseau de la machine ou de créer un Linux VLAN sur le PVE pour avoir des flux tagués (Linux VLAN moins utilisé)|Nécessite d'utiliser le VNet de la zone VLAN comme carte réseau pour avoir des flux tagués|
|N'attribue pas d'IP car consiste juste en un tag ou un Linux VLAN dédié qui n'apporte pas la fonctionnalité|Ne fournit pas non plus de DHCP car consiste seulement en un tag de flux|

En résumé, les deux fournissent le même résultat, c'est-à-dire de taguer les flux d'une carte réseau afin que l'infrastructure réseau de la société puisse assigner les flux à un VLAN.
Ils sont donc très similaires dans les fonctionnalités qu'ils proposent mais le SDN apporte quelques fonctionnalités supplémentaires non négligeables. Très utile pour les environnements d'entreprise qui utilisent la plupart du temps les VLANs pour segmenter leur réseau.
<br><br>

### QinQ
La zone QinQ (802.1Q in 802.1Q) permet d'outrepasser la limite des 4096 VLANs imposée par la norme 802.1Q en mettant un tag VLAN dans un autre tag VLAN.  
Le `C-VLAN` correspond au tag interne et le `S-VLAN` à l'externe, ajouté et géré par le fournisseur ou l'hébergeur.  

 **Exemple**  
1. Le routeur de l'opérateur à Paris reçoit les données des clients.  
2. Il enferme toutes les trames du Client A dans un S-VLAN 100 (On aura donc le tag 10 dans le tag 100).
3. Il enferme toutes les trames du Client B dans un S-VLAN 200 (On aura donc le tag 10 dans le tag 200).
4. Sur le câble en fibre optique, les switchs de l'opérateur ne regardent que l'étiquette externe (100 ou 200). Il n'y a donc plus aucun conflit.
5. À Lyon, l'opérateur retire l'étiquette externe (100 et 200) et livre les données. Le Client A récupère son VLAN 10 intact, sans jamais savoir qu'il a traversé le pays dans une plus grosse enveloppe.
<br><br>

### VXLAN
Une zone VXLAN fonctionne de manière similaire à une zone simple, mais avec un avantage majeur : elle permet la communication entre les machines virtuelles même si elles ne sont pas hébergées sur le même nœud Proxmox. Le VXLAN crée un tunnel (Overlay) qui encapsule le trafic de Niveau 2 (Ethernet) dans des paquets IP de Niveau 3 (UDP). Ainsi, des VMs réparties sur plusieurs serveurs physiques se comportent comme si elles étaient branchées sur le même switch local. Comme d'habitude, un tableau sera plus clair :

|Zone Simple|Zone VXLAN|
|-----------|----------|
|Permet de créer un réseau virtuel isolé|Idem|
|Restreint au nœud Proxmox local|Étendu à travers plusieurs nœuds (Inter-PVE)|
|Fournie par Proxmox SDN|Idem|
|Natif (via Proxmox SDN)|Natif (via Proxmox SDN)|

En résumé, le VXLAN est une sorte de zone simple mais fournissant de la commutation (switching) inter-PVE.  
Cela peut être très utile dans des clusters de home-lab ou de petites infrastructures.

> [!tip]
> Une zone VXLAN standard est moins recommandée à plus grande échelle. En effet, le protocole VXLAN utilise par défaut la méthode du "Flood and Learn" (inonder le réseau pour trouver les adresses MAC inconnues), ce qui génère trop de trafic de diffusion (Broadcast/Multicast) sur les grosses infrastructures.
<br><br>

### EVPN
La zone EVPN vient corriger le défaut principal du `VXLAN` consistant à inonder le réseau de broadcast.

Pour comprendre l'EVPN, il faut le voir comme une architecture réseau à deux niveaux distincts travaillant ensemble :
- `Le transporteur (Data Plane)` : L'EVPN utilise toujours le protocole VXLAN pour encapsuler les trames Ethernet et les faire voyager entre les nœuds Proxmox.
- `Le cerveau (Control Plane)` : C'est ici que la magie opère. L'EVPN s'appuie sur le protocole de routage BGP. Au lieu d'inonder le réseau pour trouver où se cache une machine cible, les PVE se préviennent via BGP qu'ils hébergent une machine.

|Zone VXLAN classique|Zone EVPN|
|--------------------|---------|
|Permet de faire communiquer des VMs sur des PVE différents|Idem|
|Méthode de découverte "Flood and Learn" (génère beaucoup de Broadcast)|Découverte intelligente et ciblée via le protocole BGP|
|Totalement autonome (ne nécessite pas de Contrôleur)|Nécessite obligatoirement de configurer un Contrôleur BGP/EVPN|
|Recommandé pour les petites infrastructures ou les maquettes|Le standard de l'industrie pour les grands Datacenters et le multi-sites|

En résumé, l'`EVPN` étire un réseau virtuel de Niveau 2 par-dessus votre réseau physique, tout en gardant une table de routage parfaitement claire et optimisée. C'est la solution d'entreprise par excellence lorsque l'on souhaite interconnecter plusieurs clusters distants sans saturer les liens réseau physiques.
<br><br>
<br><br>

## VNets
Pour faire simple, un VNet est l'équivalent d'un switch virtuel. C'est le point de connexion direct sur lequel vous allez brancher les cartes réseaux de vos machines virtuelles et de vos conteneurs.

Là où la Zone définit la technologie et le protocole de transport sous-jacent (le "comment" : `Simple`, `VLAN`, `VXLAN`, `EVPN`), le VNet représente le domaine de diffusion isolé (le réseau logique). Un VNet est obligatoirement rattaché à une seule et unique Zone.

Pour bien visualiser l'imbrication logique des éléments du SDN :
- `Zone` : C'est la technologie de câblage et d'interconnexion.
- `VNet` : C'est le switch physique virtuel.
- `Subnet` : C'est la plage d'adresses IP (ex: 192.168.1.0/24) et le serveur DHCP configurés sur ce switch.

|Linux Bridge|VNet SDN|
|------------|--------|
|Doit être créé manuellement sur chaque nœud PVE|Créé globalement au niveau du Datacenter et répliqué partout|
|Ne gère pas l'adressage IP par lui-même|Intègre la gestion des Subnets et le lien avec l'IPAM|
|Fonctionnement basique et autonome|Hérite de la topologie de sa Zone (ex: devient inter-PVE s'il est dans une zone VXLAN)|

Un VNet est donc un switch virtuel sur lequel on vient brancher des machines virtuelles et qui est soumis au protocole de sa zone.
<br><br>
<br><br>

## IPAM
L'IPAM est l'inventaire central de votre réseau virtuel. Son rôle est de savoir exactement quelles adresses IP sont libres, et lesquelles sont assignées à quelles machines virtuelles. Le SDN automatise cette tâche de pair avec le DHCP natif : lorsqu'une VM démarre, le SDN pioche une IP libre dans l'IPAM, la configure, et verrouille cette IP dans son registre.

Proxmox propose trois plugins :
1. `pve` : Solution par défaut de Pve, fournis dès l'installation du système. Fonctionne très bien pour enregistrer les IPs des machines virtuelles du cluster
2. `netbox` : Mise en place plus complexe, mais considéré comme la référence en entreprise. Permet de gérer les IPs à l’échelle du parc et plus seulement du cluster.
3. `phpIPAM` : Autre outils populaire, spécialisé dans la gestion visuelle des sous-réseaux et des VLANs. Ne se contente pas non plus des IPs du cluster Pve mais englobe toute le parc.
<br><br>
<br><br>

## VNet Firewall
Jusqu'à présent, le pare-feu Proxmox se gérait au niveau Datacenter, Nœud ou VM. Le SDN introduit le pare-feu au niveau du réseau (VNet).  
Cela permet de contrôler les flux de manière centralisée (ex: isoler deux sous-réseaux entre eux au sein d'une même Zone).
Le fonctionnement reste le même.
<br><br>
<br><br>

## Fabrics
Une Fabric est l'infrastructure de routage sous-jacente qui permet à vos nœuds physiques de se découvrir et de se parler. Elle garantit que le chemin IP entre les serveurs est le plus rapide, redondant ou sécurisé possible, sans se soucier du trafic des VMs.

Proxmox propose quatre types de Fabrics selon l'architecture de votre datacenter :
- `WireGuard` : Crée un tunnel chiffré de bout en bout entre les nœuds. Idéal pour des clusters multi-sites ou lorsque les serveurs doivent communiquer à travers Internet. On pensera ici à l'EVPN.
- `OSPF` : Protocole de routage dynamique interne ultra-classique. Parfait pour intégrer un cluster Proxmox dans un réseau d'entreprise existant utilisant déjà OSPF.
- `OpenFabric / IS-IS` (Le standard Datacenter) : Conçu spécifiquement pour les architectures réseau "Spine-Leaf" à très grande échelle. Extrêmement rapide, très utilisé pour faire transiter du trafic Ceph massif.
- `BGP` : Le protocole qui fait tourner Internet. Indispensable pour annoncer vos réseaux virtuels aux routeurs physiques "Upstream", et cœur absolu du fonctionnement de l'EVPN.
<br><br>
<br><br>

---

# En pratique
Afin d'appliquer ces connaissances et de mieux comprendre comment utiliser ce savoir nouvellement acquis, vous êtes libre de suivre ces tutoriels de mise en place.

## Simple
Il sera abordé ici la création et l'utilisation d'une zone de type `simple`
<br><br>
1. La première étape est de créer la zone simple dans `Datacenter` --> `SDN` --> `Zones`
<img width="800" height="419" alt="simple_zone" src="https://github.com/user-attachments/assets/894ddb7e-1300-4a05-afaf-ac3d1442e596" />

- `ID` : Un simple nom d'affichage limité à 8 caractères
- `MTU` : Taille des paquets, laissez en auto dans le cas d'une zone simple
- `Nodes` : Vous pouvez limiter la création de cette zone à certains nœuds de votre cluster. À vous d'adapter en fonction de vos besoins
- `IPAM` : Par défaut `pve` car natif à Proxmox. Vous pouvez en choisir un autre. Le sujet sera abordé en détails dans la partie `IPAM`
- `DNS Server` + `Reverse DNS Server` + `DNS Zone` : Utile si vous avez un service DNS externe à Proxmox. Pas abordé ici
- `Automatic DHCP` : Active le DHCP sur la zone. L'IPAM ne sert à rien si vous choisissez de ne pas activer le DHCP
<br><br>

2. Allez dans `Datacenter` --> `SDN` --> `VNets` et créez un nouveau `VNet`
<img width="779" height="419" alt="vnets_simple" src="https://github.com/user-attachments/assets/8532beba-d4fb-4ee9-92c5-4440d8fa1cfb" />

- `Name` : Nom d'affichage du VNet
- `Alias` : Description ou alias du VNet
- `Zone` : La zone à utiliser. Dans notre cas, la zone `Simple1`
- `Isolate Ports` : Isole les machines entre elles dans le réseau sans couper l'accès à la passerelle
- `VLAN Aware` : Autorise les flux tagués si vous utilisez un routeur virtuel dans le réseau
<br><br>

3. Dans ce même VNet, créez un réseau dans la partie `Subnets` en complétant les deux onglets : 

**General**  
<img width="1141" height="383" alt="subnet1_vnets_simple" src="https://github.com/user-attachments/assets/2cda92d4-804f-435e-ad16-68051a30ed29" />

- `Subnet` : L'adressage IP du réseau à créer
- `Gateway` : La passerelle associée
- `SNAT` : Permet d'autoriser le NAT à travers le nœud pour avoir accès à internet par exemple
- `DNS Zone Prefix` : Permet d'associer un préfixe DNS aux machines du réseau si vous l'avez configuré dans la zone 
<br><br>

**DHCP Ranges**  
Vous pouvez ajouter une/des étendue(s) DHCP dans le second onglet
<img width="1267" height="430" alt="dhcp_subnet1_vnets_simple" src="https://github.com/user-attachments/assets/69d28267-aa82-4f24-8009-6919b940b1d7" />
<br><br>

4. Une fois tout cela fait, il faut appliquer les modifications dans `Datacenter` --> `SDN`
<br><br>

5. Pour utiliser ce nouveau switch virtuel, il nous suffit de créer/modifier une carte réseau et de choisir `VNsimp1` au lieu d'un Linux Bridge (vmbr)
<img width="1051" height="364" alt="vnet_VNsimp1_debian1" src="https://github.com/user-attachments/assets/20bbabd6-467e-48b1-b4cd-fe9938ca7b4e" />
<br><br>

6. Démarrez la machine pour vérifier l'IP
<img width="1325" height="262" alt="debian1_ip-a" src="https://github.com/user-attachments/assets/dc808bfb-b0dd-488f-a690-6736e6d0c280" />

On voit ici que nous avons obtenu la première IP de l'étendue DHCP.
<br><br>

7. Pour être sûr que cette IP a bien été distribuée par le DHCP, vous n'avez qu'à confirmer la présence du bail dans l'IPAM.
Pour cela rendez-vous dans `Datacenter` --> `SDN` --> `IPAM`
<img width="1361" height="479" alt="debian1_ipam_dhcp_lease" src="https://github.com/user-attachments/assets/b80d194b-9afa-4798-8a9f-09990b69f561" />

On retrouve bien notre machine et son IP !
<br><br>
C'est tout pour la mise en place d'une zone simple.
<br><br>

## VLAN
**A mettre en place sur proxmox physique car marche pas sur maquette**


## QinQ
Le QinQ étant assez niche, ce type de zone ne sera pas abordé ici
<br><br>

## VXLAN
Pour tester le fonctionnement de la zone VXLAN, vous aurez besoin d'un cluster Proxmox d'au moins deux nœuds et d'une machine sur chaque PVE.  

1. Comme pour les autres, créez la zone dans le SDN (`Datacenter` --> `SDN` --> `Zones`) puis commencez la configuration comme dans cet exemple :
<img width="598" height="306" alt="vxlan1_zone" src="https://github.com/user-attachments/assets/d868800e-ffba-4e5f-91ee-964781695f1b" />

- `ID` : Un simple nom d'affichage limité à 8 caractères
- `Peer Address List` : La liste des nœuds PVE concernés par la propagation du réseau. Peut inclure des PVE d'autres clusters à la seule condition que vous y ayez créé la même zone
- `SDN Fabric` : Si configurée, permet de se passer de la `Peer Address List`
- `MTU` : Taille des paquets, laissez en auto pour que Proxmox l'adapte à votre carte physique (le protocole VXLAN utilise 50 octets pour encapsuler le trafic, Proxmox va donc descendre le MTU à 1450 sur une installation basique de Proxmox avec une infrastructure réseau standard)
- `Nodes` : Permet de restreindre l'activation de la zone à certains nœuds. Utile dans le cas où il faut étendre un LAN entre quelques nœuds seulement de deux (ou plus) clusters distants
- `IPAM` : Même si le DHCP n'est pas proposé dans ce type de zone, il peut être utile de tenir un inventaire des IPs. De plus, peut être utilisé pour assigner une IP libre automatiquement avec Cloud-Init, un peu comme un DHCP. Le choix par défaut étant `pve`
- `DNS Server` + `Reverse DNS Server` + `DNS Zone` : Utile si vous avez un service DNS externe à Proxmox. Pas abordé ici
<br><br>

2. Une fois la zone créée, et comme pour les autres zones, créez un nouveau VNet dans `Datacenter` --> `SDN` --> `VNets`
<img width="855" height="444" alt="vnets_vxlan" src="https://github.com/user-attachments/assets/9dc4ea33-7649-484a-b16b-a8bccbf4e5ba" />

- `Name` : Nom d'affichage du VNet
- `Alias` : Description ou alias du VNet
- `Zone` : La zone à utiliser. Dans notre cas, la zone `VXLAN1`
- `Tag` : Correspond au tag VNI (VXLAN Network Identifier) et non pas au tag VLAN comme pour la zone VLAN. Nous ne sommes donc pas limités à 4096 tags mais à 16 777 215
- `Isolate Ports` : Isole les machines entre elles dans le réseau sans couper l'accès à la passerelle
- `VLAN Aware` : Autorise les flux tagués si vous utilisez un routeur virtuel dans le réseau
<br><br>

3. Dans ce même VNet, créez un réseau dans la partie `Subnets` en complétant les deux onglets : 
<img width="1055" height="375" alt="vxlan_vnet_subnet" src="https://github.com/user-attachments/assets/88680ef9-fb81-449a-8495-6e752f7542ba" />

**Onglet General**  
- `Subnet` : L'adressage IP du réseau à créer
- `Gateway` : La passerelle associée
- `SNAT` : Permet d'autoriser le NAT à travers le nœud pour avoir accès à internet ou accéder au réseau physique
- `DNS Zone Prefix` : Utile seulement si vous avez configuré une intégration avec un serveur DNS externe, ce n'est pas notre cas ici

**Onglet DHCP Range**  
Contrairement à la zone `simple`, la zone `VXLAN` ne fournit pas de DHCP à cause du fait qu'elle soit distribuée et que cela créerait des conflits d'IP.  
Vous pouvez tout de même faire des étendues car le DHCP n'est pas la seule façon de donner une IP à une machine ! En effet si vous installez des machines avec `Cloud-Init` il se basera sur l'IPAM pour en attribuer une de libre  
<br><br>

4. Et pour finir la configuration, appliquez les modifications dans `Datacenter` --> `SDN`
<br><br>

5. Vous n'avez plus qu'à tester avec deux machines virtuelles en suivant ces étapes :
- Créer deux machines virtuelles avec n'importe quel système d'exploitation
- Leur assigner notre nouveau VNet VXLAN et leur donner une IP manuellement
- Et pour finir, faire un ping entre elles

**C'est tout pour la mise en pratique d'une zone `VXLAN` !**
<br><br>

## EVPN
Ici sera abordée la mise en place d'une zone `evpn`.
Si vous souhaitez tester la solution il vous faudra avoir a minima un nœud qui n'est pas dans le même cluster afin de simuler le cluster distant.
C'est ici mon cas, de plus je l'ai placé dans un second réseau local. En effet, le cluster est dans l'étendue `XXX.XXX.150.0/24` et mon nœud "distant" est dans `XXX.XXX.160.0/24`.
<br><br>
1. Pour commencer, créez un contrôleur de type `evpn` dans `Datacenter` --> `SDN` --> `Options` (Controllers) et renseignez comme suit :
<img width="688" height="322" alt="evpn_controler" src="https://github.com/user-attachments/assets/0a3a785f-1763-4dfc-ae5d-1db8c818e293" />

- `ID` : Un simple nom d'affichage limité à 8 caractères
- `ASN #` : Le numéro d'identification privé de ce nœud. Il en existe des privés et des publics. Vous pouvez mettre `65101` pour ce test 
- `SDN Fabric` : Ne le configurez pas ici car nous n'en utilisons pas
- `Peers` : L'ip de chaque nœud distant avec lequel ton cluster doit échanger ses routes
<br><br>

2. Une fois le contrôleur créé, créez la zone dans `Datacenter` --> `SDN` --> `Zones`. Exemple :
<img width="638" height="501" alt="evpn_zone" src="https://github.com/user-attachments/assets/6160b815-30f6-4921-8994-89b8e08d5765" />

- `ID` : Un nom d'affichage simple limité à 8 caractères
- `Primary Controller` : Le contrôleur créé à l'étape précédente 
- `VRF-VXLAN Tag` : C'est l'identifiant unique (VNI) du routeur virtuel (VRF) qui gérera le trafic de Niveau 3 pour cette zone. Mettez un numéro (ex: 10000). Attention : Il doit être strictement identique sur le cluster principal et sur le nœud distant
- `VNet MAC Address` : Laissez en auto. Proxmox générera une adresse MAC Anycast. La passerelle aura la même adresse MAC physique sur tous vos serveurs. Si une VM migre d'un serveur à l'autre, elle ne se rendra même pas compte que le routeur a changé
- `Exit Nodes` : La/les passerelle(s) qui permet(tent) aux machines virtuelles de sortir vers internet ou un réseau local physique. Vous pouvez mettre tous les nœuds PVE
- `Primary Exit Node` : Si vous avez sélectionné plusieurs `Exit Nodes` pour la redondance, vous définissez ici lequel est le routeur principal. Mettez celui que vous voulez
- `Exit Nodes Local Routing` : Si coché, l'`Exit Node` ne fera pas de NAT et routera le trafic directement vers les réseaux physiques qu'il connaît
- `Advertise Subnets` : Si vous avez un routeur d'entreprise physique au-dessus de Proxmox qui "parle" BGP, cette option permet à Proxmox de lui annoncer l'existence de vos sous-réseaux virtuels. Inutile dans notre cas
- `Disable ARP-nd Suppression` : Ne pas cocher. Par défaut, l'EVPN intercepte les requêtes ARP et y répond silencieusement grâce à sa base BGP, évitant ainsi le Broadcast. Cocher cette case désactive cette optimisation majeure de l'EVPN
- `Route Target Import` : Paramètre BGP très avancé pour croiser des tables de routages entre plusieurs entreprises ou VRF. Laissez vide
- `MTU, Nodes, IPAM` : Laissez par défaut (auto, All, pve), comme pour une zone classique
<br><br>

3. Cela terminé, faites un VNet dans notre nouvelle zone `EVPN1`. Comme d'habitude, cela se passe dans `Datacenter` --> `SDN` --> `VNets`
<img width="643" height="303" alt="evpn_vnet" src="https://github.com/user-attachments/assets/753eff20-e3a8-4ace-a5c8-6f403fe3eb54" />

- `Name` : Un nom d'affichage simple limité à 8 caractères
- `Alias` : Une description si besoin
- `Zone` : Notre nouvelle zone EVPN
- `Tag` : Un tag obligatoirement différent de celui de `VRF-VXLAN Tag` définis dans notre zone, `10010` par exemple.

> [!caution]
> Le tag doit être **identique sur le cluster et le nœud/cluster distant**. C'est ce qui permet aux machines de communiquer

<br><br>
4. Et pour finir avec la configuration du cluster, créez un réseau dans le VNet que vous venez de configurer

**Onglet General**  
- `Subnet` : Le réseau que vous voulez
- `Gateway` : La passerelle associée de votre réseau
- `SNAT` : Permet à vos machines d'utiliser les `Exit Nodes` que vous avez configurés dans votre zone pour sortir sur internet ou sur le réseau physique
- `DNS Zone Prefix` : Pas besoin ici car nous n'avons pas configuré de service DNS externe

**Onglet DHCP Range**  
Contrairement à la zone `simple`, la zone `EVPN` ne fournit pas de DHCP à cause du fait qu'elle soit distribuée et que cela créerait des conflits d'IP.  
Vous pouvez tout de même faire des étendues car le DHCP n'est pas la seule façon de donner une IP à une machine ! En effet si vous installez des machines avec `Cloud-Init` il se basera sur l'IPAM pour en attribuer une de libre  
<img width="1053" height="395" alt="evpn_vnet_subnet" src="https://github.com/user-attachments/assets/a07d8a03-1cc0-4bea-a381-5d3bcc808265" />


---

# Erreurs connues

### Erreur 500 lors de la suppression d'un subnet dans un VNet
1. Passez en CLI sur un des nœuds et éditez le fichier `/etc/pve/sdn/subnets.cfg`
2. Supprimez les lignes qui correspondent à votre subnet
3. Vérifiez que le subnet est passé dans l'état `Deleted` dans `Datacenter` --> `SDN` --> `VNets` --> Votre-VNet
4. S'il est bien supprimé, appliquez les changements dans `Datacenter` --> `SDN`
