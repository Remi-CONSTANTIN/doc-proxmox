# Restauration système
La restauration entière d'une VM se passe de la même manière avec ou sans PBS

`Datacenter` --> `noeud` --> `Stockage` --> `Backup` --> `Backup à restaurer`

<img width="1224" height="419" alt="restauration-sys-proxmox" src="https://github.com/user-attachments/assets/f4000de1-4c14-4bb9-ae7c-a3cc17ecfd94" />


---
# Restauration d'un fichier/dossier
La restauration de fichier est presque au même endroit que la restauration d'un système.
Avec la restauration de fichier, vous pouvez naviguer dans le système de fichier de la VM !

<img width="1241" height="638" alt="restauration-file-proxmox" src="https://github.com/user-attachments/assets/39ebd0ae-067a-4467-baea-5c0fbd6b2c31" />


Dans cet exemple nous avons le choix entre ``.zip`` ou ``.tar.zst`` car j'ai sélectionné un dossier.
Si vous sélectionnez un fichier vous aurez une simple option "Download".

> [!warning]
> L'option "File restore "n'est affichée que si le datastore provient d'un PBS !

Vous l'aurez sans doute compris, la restauration automatique du fichier dans la VM n'est pas proposée par proxmox, vous devrez le faire à la main (via WinSCP par exemple)...
