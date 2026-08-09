# Introduction
Il arrive que le disque système de PBS vienne à manquer de place. Cela est courant et est plus ou moins fréquent en fonction de la taille du disque
<br>

Dans mon cas, PBS est une machine virtuelle avec un disque de 40Go dédié au système
<br>

> [!warning]
> L'utilisation d'un serveur de sauvegarde dans une machine virtuelle n'est généralement pas conseillée car si votre cluster Proxmox tombe, la solution que aurait pu vous permettre de restaurer vos sauvegardes tombe aussi !
> 
> Je fais cela chez moi à cause de mes contraintes financières
<br>

# Problème et diagnostique
J'ai tout d'abord reçu une alerte de ma supervision
<img width="1535" height="33" alt="supervision_alert_root_fs_pbs" src="https://github.com/user-attachments/assets/ff7c6e09-5942-44d4-bf3a-2ef77f30edf2" />

Effectivement, ma partition unique montée sur la racine `/` allait venir à manquer de place
<img width="764" height="225" alt="df-h_pbs" src="https://github.com/user-attachments/assets/89f391cf-45a4-46c6-b2db-1a8bf7a59842" />

Après investigation, j'ai découvert que mon répertoire `/usr/lib/modules` occupait 18 Go sur 40 Go !
<img width="293" height="417" alt="usr-lib-modules_pbs" src="https://github.com/user-attachments/assets/2e409ad7-d6bb-49a1-91bf-4461cd0b63b8" />

**Mais pourquoi ce dossier contient autant de données ?**  
--> Ce repertoire contient les anciens kernels PVE car PBS les stocke afin de pouvoir revenir en arrière en cas de problème

Nous allons donc voir comment le nettoyer simplement

# Résolution
1. Commencez par vérifier la version de votre noyau avec `uname -r`
<img width="367" height="66" alt="uname-r_pbs" src="https://github.com/user-attachments/assets/d64d756d-c90b-4e51-8bd5-1419763df05e" />
<br><br>

2. Laissez le gestionnaire de paquets faire le nettoyage avec les commandes :
- `apt update` afin de rafraîchir le catalogue des paquets  
- `apt autoremove --purge` pour nettoyer les paquets obsolètes  
<img width="1381" height="278" alt="apt-autoremove--purge_pbs" src="https://github.com/user-attachments/assets/7e2877ad-cbfa-4700-98b9-b2b0f58bcc24" />  

On voit bien que tous les vieux noyaux seront supprimés et que nous allons gagner 15Go sur notre disque !
<br><br>

4. Suite à ça, vérifions les noyaux restants avec `du -h  -d 1 /usr/lib/modules | sort`
<img width="363" height="177" alt="left_kernel_pbs" src="https://github.com/user-attachments/assets/65b48168-e78a-4baa-a9d1-d7644d2afc52" />  

Les gestionnaire de paquets conserve par défaut les noyaux les plus récents afin d'avoir une solution de secours en cas de problème
<br><br>

5. Puis vérifions que le disque est de nouveau dans un utilisation normale
<img width="492" height="94" alt="df-h_after_cleanup_pbs" src="https://github.com/user-attachments/assets/40d11da0-de87-4f9a-8e8f-f649744c5cf7" />  

## Perspective d'amélioration
Afin de ne plus à avoir ce genre de problème il serait bon d'automatiser le nettoyage via script
