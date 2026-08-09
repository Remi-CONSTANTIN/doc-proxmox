# Introduction
Par défaut, Proxmox demande qu’une majorité des nœuds du cluster soient en ligne pour garder le quorum (et donc permettre de gérer les VMs).  
Donc si tu n’as que **2 nœuds**, il faut 2 votes pour avoir un quorum → si un tombe, l’autre perd le quorum et tu ne peux plus rien administrer via l’interface.

> [!caution]
> - Mettre le quorum à 1 (forcer le cluster à continuer même seul) enlève la protection contre les **split-brain** (les deux nœuds qui pensent être maître en même temps)
> - C’est acceptable **seulement dans un lab ou un cluster à 2 nœuds max**, pas en production

# Procédure

### Méthode pour mettre le quorum à 1

Tu dois modifier la config **corosync**.

1. Édite le fichier de config :
```
nano /etc/pve/corosync.conf
```

2. Dans la section _quorum { ... }_, ajoute :
```
quorum {
	provider: corosync_votequorum
	two_nodes: 1
	wait_for_all: 0
}
```
**Explications :**
- two_node: 1 → active le mode spécial pour les clusters à 2 nœuds.
- wait_for_all: 0 → ne bloque pas le quorum si un nœud est absent au démarrage.

3. Redémarre corosync :
```
systemctl restart corosync
```

4. Vérifie l’état du quorum :
```
pvecm status
```
