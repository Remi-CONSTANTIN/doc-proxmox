# Contexte
Une erreur survient lors de la sauvegarde d'un conteneur LXC sur un datastore NFS.  
La sauvegarde est ici gérée par PVE. Aucun PBS n'intervient ici.

# Exemple

Logs du job de backup
```
root@pve2:~# cat /var/log/vzdump/lxc-104.log
2025-08-24 00:05:02 INFO: Starting Backup of VM 104 (lxc)
2025-08-24 00:05:02 INFO: status = running
2025-08-24 00:05:02 INFO: CT Name: proxy-IoT
2025-08-24 00:05:02 INFO: including mount point rootfs ('/') in backup
2025-08-24 00:05:02 INFO: backup mode: snapshot
2025-08-24 00:05:02 INFO: ionice priority: 7
2025-08-24 00:05:02 INFO: create storage snapshot 'vzdump'
2025-08-24 00:05:02 INFO: creating vzdump archive '/mnt/pve/NAS_Backup-Proxmox/dump/vzdump-lxc-104-2025_08_24-00_05_02.tar.zst'
2025-08-24 00:05:02 INFO: tar: /mnt/pve/OniiNAS_Backup-Proxmox/dump/vzdump-lxc-104-2025_08_24-00_05_02.tmp: Cannot open: Permission denied
2025-08-24 00:05:02 INFO: tar: Error is not recoverable: exiting now
2025-08-24 00:05:02 INFO: cleanup temporary 'vzdump' snapshot
2025-08-24 00:05:03 ERROR: Backup of VM 104 failed - command 'set -o pipefail && lxc-usernsexec -m u:0:100000:65536 -m g:0:100000:65536 -- tar cpf - --totals --one-file-system -p --sparse --numeric-owner --acls --xattrs '--xattrs-include=user.*' '--xattrs-include=security.capability' '--warning=no-file-ignored' '--warning=no-xattr-write' --one-file-system '--warning=no-file-ignored' '--directory=/mnt/pve/OniiNAS_Backup-Proxmox/dump/vzdump-lxc-104-2025_08_24-00_05_02.tmp' ./etc/vzdump/pct.conf ./etc/vzdump/pct.fw '--directory=/mnt/vzsnap0' --no-anchored '--exclude=lost+found' --anchored '--exclude=./tmp/?*' '--exclude=./var/tmp/?*' '--exclude=./var/run/?*.pid' ./ | zstd '--threads=1' >/mnt/pve/OniiNAS_Backup-Proxmox/dump/vzdump-lxc-104-2025_08_24-00_05_02.tar.dat' failed: exit code 2
```

---
# Solution

1. SSH sur le noeud PVE du LXC
2. Edition du fichier `/etc/vzdump.conf`
3. Changer la ligne ``#tmpdir: DIR`` en : ``tmpdir : /tmp``

# Explication
La construction de l'archive sur le partage NFS échoue car les LXC utilisent un utilisateur qui n'a pas les droits d'écrire sur le partage NFS. Nous précisons donc au PVE de la construire en local et ensuite de l'envoyer dans le partage

# Source

[**[SOLVED] Cannot backup only LXC to NFS, VM works**](https://forum.proxmox.com/threads/cannot-backup-only-lxc-to-nfs-vm-works.90797/)
