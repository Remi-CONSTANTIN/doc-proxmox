# **Prérequis :**

- Un compte chez un fournisseur SMTP (Google, mailjet, etc...)
+ Un mot de passe application sur ce compte

# **Procédure :**

1.      Installer les paquets requis :
```
apt-get install postfix mailutils libsasl2-2 ca-certificates libsasl2-modules
```

2.      Activer le service au démarrage :
```
systemctl enable postfix
```

3.      Configurer les relais postfixe (exemple avec gmail) :
```
nano /etc/postfix/main.cf
```

a.      On change le relai SMTP en modifiant la ligne
``relayhost = [smtp.gmail.com]:587
Ou par exemple  : relayhost = ``[smtp.mailjet.com]:587

b.      On ajoute les lignes suivantes à la fin du fichier
```
smtp_sasl_auth_enable = yes
smtp_sasl_password_maps = hash:/etc/postfix/sasl_passwd
smtp_sasl_security_options = noanonymous 
smtp_tls_CAfile = /etc/postfix/cacert.pem
smtp_use_tls = yes
```

4.      Configurer les identifiants de connexion au relais SMTP (exemple gmail) :
```
nano /etc/postfix/sasl_passwd
```

a.      Dans le fichier vide on ajoute les identifiants (exemple Gmail)
```
[smtp.gmail.com]:587 USERNAME@gmail.com:MDP_Application
```

5.      Changer les permissions du fichier :
```
chmod 600 /etc/postfix/sasl_passwd
```

6.      On « transforme notre fichier d’identifiants en une sorte de base de données :
```
postmap /etc/postfix/sasl_passwd
```

7.      Créer un certificat .pem dans le répertoire dédié
```
cd /etc/ssl/certs
```
puis
```
openssl req -newkey rsa:2048 -new -nodes -x509 -days 3650 -keyout key-for-smtp.pem -out cert-for-smtp.pem
```

8.      On redirige le certificat généré dans notre postfix :
```
cat /etc/ssl/certs/cert-for-smtp.pem | sudo tee -a /etc/postfix/cacert.pem
```

9.      Recharger le service postfixe pour appliquer les modifications
```
/etc/init.d/postfix reload
```

10.  Optionnel : Vérifier son état
```
/etc/init.d/postfix status
```

11.  Tester l’envoie d’email :
```
echo "Test mail from postfix" | mail -s "Test Postfix" [adresse@mail.com](mailto:adresse@mail.com)
```

12.  Paramétrer l’adresse mail d’envoie dans proxmox (même que celle de réception)  
a.      Interface graphique : Datacenter → Notifications → Email  
b.      CLI : ``pvesh set /cluster/options --email-from [proxmox@domaine.fr](mailto:proxmox@domaine.fr)``
