# Procédure

1. Télécharger/importer le .ova sur le nœud proxmox (wget/WinSCP etc…)
<img width="900" height="60" alt="debian_lxc_docker ova" src="https://github.com/user-attachments/assets/a0983f87-35cc-4da7-b766-3984a2c62cfe" />


2. Extrayez le fichier OVA à l'aide de la commande tar
```
tar -xf Debian-LXC-Docker.ova
```
Vous avez maintenant au moins 2 nouveaux fichiers : 
- 1 .OVF
- 1 .vmdk
- D'autres fichiers utiles en fonction de celui qui a compresser cette VM

<img width="900" height="60" alt="tar_-xf_Debian-LXC-Docker ova" src="https://github.com/user-attachments/assets/4ef2bd55-00dd-4c75-957d-f68b42f0bae5" />

3. Plus qu’à importer la VM à partir du fichier .OVF
```
qm importovf  117 Debian-LXC-Docker.ovf local-lvm --format qcow2
```

> [!NOTE] A faire avant exécution
> Remplacer 117 par un ID de VM libre dans votre proxmox
> Remplacer Debian-LXC-Docker.ovf par le nom de votre OVF
> Remplacer local-lvm par le stockage où vous souhaitez importer la VM

Il est normal d'avoir une erreur à propos du QCOW2. Visiblement le format Qcow2 n’est pas supporté dans le local-lvm et il utilise plutôt le format RAW à la place…
Allez voir la documentation Proxmox si ça vous intéresse


4. Démarrez la VM qui vient d’apparaitre dans votre interface Web
Dans mon exemple, la nouvelle VM n’a pas de carte réseau, soit parce qu’elle vient d’un autre hyperviseur avec des noms de carte réseau différents, soit parce que la personne n’en a pas mis avant de l’exporter… Chai pas
