# Guide de mise en place de la notification Proxmox sur Discord
Vous trouverez ici des ressources sur la mise en place des notifications Proxmox dans un salon Discord via l'utilisation de l'option `Webhook`

> [!note]
> Documentation réalisée sur PVE 9

---

# Mise en place

1. Commencez par créer un `Webhook` dans le serveur Discord de votre choix en allant dans ses paramètres : `Paramètres du serveur` --> `Intégrations` --> `Webhooks` --> `Créer un webhook` --> Paramétrez son nom, sa photo et le salon dans lequel envoyer les notifications.  
Il ne vous reste plus qu'à copier le lien en cliquant sur `Copier l'URL du webhook`  

2. Une fois cela fait, retournez sur Proxmox pour configurer l'envoi vers ce canal Discord en cliquant sur `Add` (toujours dans `Notification Targets`)

3. Complétez les informations en les adaptant à votre contexte :
- `Endpoint Name` : Le nom de votre cible (Exemple : Discord)
- `Enable` : Laissez coché pour l'activer
- `Method/URL` (POST) : L'URL de votre webhook Discord que vous venez de copier
- `Headers` : Ajoutez-en un et mettez-y la valeur `Content-Type` dans la case de gauche et `application/json` dans la case de droite
- `Body` : Si vous voulez un exemple tout prêt, vous pouvez utiliser celui que je vous mets en annexe "BODY pour la notification Discord", sinon faites-le vous-même
- `Comment` : Un simple commentaire

Avec le `body` que je vous propose en annexe, cela peut donner ceci lors d'une alerte de type inconnue :  
<img width="404" height="398" alt="exemple_notification_discord" src="https://github.com/user-attachments/assets/c4250300-95e7-486c-93fb-ca5ab59f440d" />

---

# Annexes
## BODY pour la notification Discord
```
{
  "username": "Proxmox VE",
  "avatar_url": "https://www.proxmox.com/apple-touch-icon.png",
  "content": "🚨 **Alerte Proxmox**",
  "embeds": [
    {
      "title": "{{ escape title }}",
      "description": "**Message :**\n```\n{{ escape message }}\n```\n📅 **Date :** <t:{{ timestamp }}:f>",
      "color": 14839040,
      "fields": [
        {
          "name": "Sévérité",
          "value": "`{{ severity }}`",
          "inline": true
        }
        {{#if fields.type }}
        ,{
          "name": "Type",
          "value": "`{{ fields.type }}`",
          "inline": true
        }
        {{/if}}
        {{#if fields.hostname }}
        ,{
          "name": "Hôte",
          "value": "`{{ fields.hostname }}`",
          "inline": true
        }
        {{/if}}
      ],
      "footer": {
        "text": "Proxmox Notification System"
      }
    }
  ]
}
```
