# Introduction
Lors de la mise en place d'un cluster Pve, il est généralement conseillé de séparer les flux gourmand en bande passante (migration VMs, réplication ZFS, CEPH etc...) des flux sensibles à la latence (ceux du cluster notamment).  

## Objectif
C'est justement ce que nous allons faire ici, en ajoutant un câble USB4-C entre deux nœuds Pve (bien qu'il existe plusieurs façons de faire en lien entre eux)

## Avantage et inconvénient

**Cool :**  
- Débit théorique de l'USB4 est à 40Gps déplaçant le goulot d'étranglement au niveau du SSD (qui est très haut si vous en avez un du type NVME !)

**Pas cool :**  
- Ne permet pas de relier les 3 nœuds car les plupart des mini-PC n'ont qu'un seul port USB4

## Comment savoir si vous pouvez le faire ?
Il vous suffit de regarder si vos machines possèdent un port USB-C où il est écrit `USB4`.  
Si vous ne savez pas, référez vous à la fiche technique.  

</br>

---
</br>

# Procédure

## Configuration des liens

1. Commencez par brancher le câble USB-C entre les deux nœuds. Pas besoin d'éteindre les nœuds, faites le à chaud

2. Connectez-vous à la console des nœuds et activez le module thunderbolt
```
modprobe thunderbolt-net
```
puis faites en sorte qu'il soit chargé au démarrage du nœud
```
echo "thunderbolt-net" > /etc/modules-load.d/thunderbolt-net.conf
```

3. Vérifiez les logs du système afin de vérifier qu'il soit soit détecté
```
dmesg | grep -i -E "thunderbolt|usb4|net" | tail -n 20
```

4. Puis vérifiez la liste des cartes réseaux dans la console
```
ip link | grep thunderbolt
```
puis dans l'interface graphique du Pve dans "Votre nœud" --> `System` --> `Network` 
<img width="2088" height="290" alt="thunderbolt_pve" src="https://github.com/user-attachments/assets/a6516a3c-0f76-45aa-8c17-c6ff785ec538" />

On voit bien qu'une nouvelle interface physique `thunderbolt0` est apparue !

5. Une fois que vous avez fait les manipulations précédentes sur vos deux nœuds Pve, plus qu'à configurer les interfaces physiques avec une IP comme ceci :

**Mon premier Pve**  
<img width="410" height="226" alt="thunderbolt_pve_ip_configuration_hephaistos" src="https://github.com/user-attachments/assets/497b2a46-8b47-4374-a226-40d394f20a90" />

**Mon deuxième Pve**  
<img width="410" height="224" alt="thunderbolt_pve_ip_configuration_zeus" src="https://github.com/user-attachments/assets/84f0d0dc-af30-41cb-9a05-ba5ee7b724c1" />

Sans oublier d'appliquer la configuration à chaque modification !  
<img width="1654" height="232" alt="apply_ip_configuration_thunderbolt_hepaistos" src="https://github.com/user-attachments/assets/a7e36806-b990-4dbd-bb00-3c23fe4ee187" />
</br><br/>

## Configuration des nœuds Pve

1. Aller dans `Datacenter` --> `Options` et modifiez le réseau dans l'option `Migration Settings`
<img width="1702" height="523" alt="datacenter_migration_configuration" src="https://github.com/user-attachments/assets/c4e86e36-4aa2-4c14-b058-e16bdb2a936b" />

2. Si vous avez activé le firewall proxmox alors ouvrez les flux entre vos deux nœuds pour ce nouveau réseau

Personnellement, je n'ai pas trop cherché à me compliquer la vie, j'ai ouvert tout les flux entrants entre les deux nœuds sur cette interface :  
<img width="410" height="237" alt="firewall_rule_usb_lan" src="https://github.com/user-attachments/assets/c6abfe13-85f1-4db8-bab4-5e0fd4d9473c" />  
J'ai placé cette règle sur les deux nœuds Pve   

3. Une fois cela fait, plus qu'à tester en migrant une machine vers le deuxième nœud  
<img width="417" height="190" alt="migrate_214" src="https://github.com/user-attachments/assets/f9142ef2-5ea3-4bf6-9c14-41bccb29974b" />  

Cette machine est allumée et pèse environ 70Go  

**On constate plusieurs choses :**  

- <img width="476" height="22" alt="migrate_214_log" src="https://github.com/user-attachments/assets/0e3401f7-36a5-474a-87ee-0560858f4bd4" />  

On voit ici que le disque de 70Go s'est transféré en seulement **47s** !  

- <img width="536" height="22" alt="migrate_214_log_3" src="https://github.com/user-attachments/assets/2a8e4e1e-c25d-4039-a2f4-f1ad4b63bb73" />  

Sur cette capture on constate un downtime de seulement **102ms**

- <img width="486" height="22" alt="migrate_214_log_2" src="https://github.com/user-attachments/assets/852ac7d9-3a83-45af-a1bc-dcfc981ea748" />  

Et ici on constate que la migration à chaud s'est faites en seulement **1min** !  

