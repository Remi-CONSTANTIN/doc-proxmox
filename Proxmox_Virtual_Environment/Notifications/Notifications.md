# Guide de paramétrage des notifications Discord
Ce guide a pour but de vous expliquer le fonctionnement et la mise en place de règles de notifications dans PVE, dans une optique de supervision de l'infrastructure.

> [!note]
> Documentation réalisée sur PVE 9

---

# Mise en place  
Tout se passe dans l'onglet `Datacenter` --> `Notifications` et se divise en deux parties :

## 1. Destinataire  
Nous allons commencer par créer un destinataire pour nos futures notifications.   
Allez dans la partie `Notification Targets` (ou `Endpoints`) --> bouton `Add`  

### A. Les plateformes de notification

Si vous cliquez sur `Add`, vous remarquerez que vous avez plusieurs choix :  
- `Gotify` : Permet d'utiliser un serveur de notification externe pour envoyer des notifications push sur les appareils disposant de l'application smartphone Gotify ou via l'interface web
- `SendMail` : Permet d'envoyer des mails via le paquet `postfix`, nécessite de configurer un relais SMTP en CLI
- `SMTP` : Permet également d'envoyer des mails, mais tout se configure via l'interface web. C'est donc plus simple
- `Webhook` : Très versatile car c'est une méthode assez universelle permettant l'envoi de notifications dans Discord, Slack, Teams et autres messageries

> [!tip]
> Vous trouverez un cas pratique avec Discord dans le même dossier que cette documentation [Discord.md](Discord.md)

Il ne reste plus qu'à créer les règles de déclenchement !  

## 2. Règles de notifications  
La deuxième étape consiste à créer un déclencheur dans la partie `Notification Matcher` --> bouton `Add`, en passant par plusieurs onglets :

**- Onglet General**  
- `Matcher Name` : Le nom de votre règle.
- `Enable` : Laissez coché pour l'activer.
- `Comment` : Un simple commentaire si besoin.

**- Onglet Match Rules**  
C'est ici que l'on va ajuster les conditions de déclenchement de la notification.

`Match-field / Conditions` : Permet de choisir dans quel cas la notification se déclenche en fonction des règles que l'on va définir ensuite :  
  a. `All rules match` : Toutes les règles définies en dessous doivent correspondre  
  b. `Any rules match` : N'importe laquelle des règles doit correspondre  
  c. `At least one rule does not match` : Au moins une règle ne correspond pas  
  d. `No rules match` : Aucune des règles ne correspond  

Après avoir choisi les conditions globales, il faut créer des règles spécifiques basées sur plusieurs paramètres :  
- `Node Type` : Propose 3 choix qui ont leurs propres sous-choix :  
  1. `Match Field` : Filtre sur une ressource Proxmox spécifique  
     a. `Match Type` : `Exact` pour cibler une ressource précise ou `Regex` pour cibler plusieurs ressources  
     b. `Field` : Le type de ressource concernée par la règle, ce qui permet de choisir la ressource exacte dans le champ `Value` juste en dessous  
  2. `Match Severity` : Filtre sur la sévérité de l'événement  
     a. `Info` / `Notice` : Information basique. Très pratique pour tester votre règle après création  
     b. `Warning` : Événement de gravité "Avertissement". Non bloquant, mais à surveiller  
     c. `Error` : Une erreur grave signifiant une défaillance (souvent l'échec d'une tâche)  
     d. `Unknown` : Événement qui n'a pas pu être classé par PVE  
  3. `Match Calendar` : Permet de ne déclencher l'alerte que pendant certaines plages horaires  
 
Exemple simple permettant de recevoir des notifications à chaque exécution de n'importe quel job (pratique pour vérifier si la règle fonctionne) :  
<img width="530" height="89" alt="exemple_notification_rule_proxmox" src="https://github.com/user-attachments/assets/10589e53-ea64-45af-8d97-1c77e8db2863" />

**- Onglet Targets to notify**  
C'est ici que vous sélectionnez tout simplement la méthode de notification (le Endpoint Discord) que nous avons créée à l'étape précédente  
