# Utilité
Le **QEMU Guest Agent** est un petit service à installer à l'intérieur de vos machines virtuelles (VM). Il sert de "pont" de communication direct entre l'hyperviseur Proxmox et le système d'exploitation de la VM.

Il permet notamment :
- Gestion plus propre de l'alimentation
- "Geler" le système de fichiers (fsfreeze) de la VM pendant une sauvegarde ou un snapshot
- Avoir des informations réseau dans l'interface proxmox

# Installation
L'installation de l'agent est très rapide et ne nécessite aucune configuration
```
apt-get install qemu-guest-agent
systemctl start qemu-guest-agent
systemctl enable qemu-guest-agent
```
Une fois fait, vous devriez voir l'IP de la machine sur son "Summary"
# Sources
[Wiki officiel](https://pve.proxmox.com/wiki/Qemu-guest-agent#Linux)
