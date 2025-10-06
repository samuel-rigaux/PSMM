# PSMM

> Projet : Python, Shell, Mariadb, Mail

## SOMMAIRE
1. [FTP](#serveur-ftp)
2. [MariaDB](#server-mariadb)
3. [Web](#server-web)
4. [SSH](#ssh)
5. [Python](#python)
6. [Job 2](#mail)


> Toute la suite de ce tutoriel sera fait en ``su -``

# Préparation des serveurs

## VMs

- Serveur FTP : 1 Go - 1 Vcpu – 8 Go HDD.
- Serveur Web : 1 Go - 1 Vcpu – 8 Go HDD (apache).
- Serveur SQL : 2 Go - 2vcpu – 8 Go HDD.

## Serveur FTP

Mise à jour du système :

    apt update && apt upgrade -y

Installation de ProFTPD :

    apt install proftpd -y

Création du groupe d’utilisateurs FTP et configuration de base :

    addgroup ftpusers
    usermod -aG ftpusers monitor

Modifier le fichier`/etc/proftpd/proftpd.conf`  :

    ServerName "server-ftp"
    DefaultRoot ~
    RootLogin off
    <Limit LOGIN>
      AllowGroup ftpusers
      DenyAll
    </Limit>

Redémarre ProFTPD :

    systemctl restart proftpd


## Server MariaDB

Installation :

    apt install mariadb-server mariadb-client -y

Lancement et activation :

    systemctl enable mariadb
    systemctl start mariadb

Sécurisation : 

    mysql_secure_installation

> _Définis un mot de passe root, supprime les utilisateurs anonymes et la base de test._

Création d’un utilisateur administrateur :

    mariadb
    CREATE USER 'monitor'@'localhost' IDENTIFIED BY 'monMotDePasse';
    GRANT ALL PRIVILEGES ON *.* TO 'monitor'@'localhost' WITH GRANT OPTION;
    FLUSH PRIVILEGES;
    EXIT;

## Server WEB

> Apache 2 était déjà pré-installé avec ma VM, mais sinon, vous pouvez
> l'installer avec ``apt install apache2 -y``

Vérification et redémarrage d’Apache2 :

    systemctl status apache2
    systemctl restart apache2
    
Sécurisation – Configuration de base :

Modifier le fichier ``/etc/apache2/apache2.conf`` :

    ServerSignature Off
    ServerTokens Prod
    
> _Ces paramètres limitent les infos techniques envoyées par le serveur web._

Redémarre le service Apache2 :

    systemctl restart apache2

## SSH

### Configuration SSH sécurisée pour tous les serveurs : 

-   Modification du compte monitor avec sudo
    
-   Désactivation de l’accès root par SSH
    
-   Accès SSH pour monitor uniquement par clé

> À faire pour chaque serveurs indépendamment !

Modifier le fichiers ``/etc/ssh/sshd_config``:

    PermitRootLogin no
    AllowUsers monitor
    PasswordAuthentication no
    PubkeyAuthentication yes
    

> Et bien sûr décocher le #Port 22

Création de la clé sur Windows :

    ssh-keygen -t rsa -b 4096 -C "monitor@server"

-   Appuie sur Entrée à chaque question (chemin, mot de passe facultatif).
    
-   Une paire de fichiers sera créée dans le dossier  `C:\Users\<ton utilisateur>\.ssh\`

	-   `rsa_keys`  (clé privée)
    
	-   `rsa_keys.pub`  (clé publique)

Récupère ta clé publique :

-   Copie tout le texte affiché (commence par  `ssh-rsa...`).


Ajouter la clé sur la VM Debian (console en local sur la VM sous root/monitor) :

-   Connecte toi à la VM en console ou via autre compte administrateur (pas en SSH).
    
-   Colle la clé dans  `/home/monitor/.ssh/authorized_keys` :*

    mkdir -p /home/monitor/.ssh
    nano /home/monitor/.ssh/authorized_keys

Colle la clé une ligne unique, puis sauvegarde.

Mettre les bonnes permissions :

    chown monitor:monitor /home/monitor/.ssh/authorized_keys
    chmod 600 /home/monitor/.ssh/authorized_keys
    chmod 700 /home/monitor/.ssh

_Important pour le fonctionnement du SSH par clé._

Ta clé publique Windows est maintenant acceptée et tu peux te connecter avec :

    ssh monitor@<IP_DE_TA_VM>


## Python

> Création d'une VM sans interface graphique avec python les outils
> client pour s'envoyer des mails.

### 1. Mise à jour du système

    apt update && sudo apt upgrade -y

### 2. Installation de Python 3

    apt install python3 python3-pip -y

-   Vérifie l’installation :

```
python3 --version
pip3 --version
```

### 3. Installation du client MariaDB

    apt  install mariadb-client -y

### 4. Installation du client FTP

    sudo  apt  install  ftp -y

### 5. Envoi de mail en Python 

-   La bibliothèque standard Python ("smtplib") suffit pour envoyer des mails.

-   Exemple simple :

```
import smtplib
from email.mime.text import MIMEText

msg = MIMEText("Contenu du mail")
msg['Subject'] = "Sujet"
msg['From'] = "ton_email@domaine.fr"
msg['To'] = "destinataire@domaine.fr"

s = smtplib.SMTP('adresse_smtp')
s.login("user", "motdepasse")  # Optionnel selon SMTP utilisé
s.sendmail(msg['From'], [msg['To']], msg.as_string())
s.quit()
```

> _Adaptation nécessaire selon le serveur SMTP utilisé (Gmail, OVH, etc.)._

### 6. Vérification des installations

    python3 --version
    pip3 --version
    mariadb --version  # Pour MariaDB ou MySQL client
    ftp --version


## Mail

> Ce document explique comment utiliser un script Python d’envoi de mail
> avec détection automatique du serveur SMTP selon l’adresse email,
> incluant une configuration spéciale pour le domaine `laplateforme.io`.
> Ce dernier utilise le serveur SMTP de Google et nécessite un mot de
> passe d'application Google.

---

### Script Python résumé

    import smtplib  
    from email.mime.text import MIMEText  
    from getpass import getpass
    
    def get_smtp_server_and_port(email):  
    domain = email.split('@')[-1].lower()  
    if domain == 'gmail.com':  
    return 'smtp.gmail.com', 587  
    elif domain == 'yahoo.com':  
    return 'smtp.mail.yahoo.com', 587  
    elif domain in ['outlook.com', 'hotmail.com']:  
    return 'smtp.office365.com', 587  
    elif domain == 'laplateforme.io':  
    # Utilise Google SMTP pour laplateforme.io  
    return 'smtp.gmail.com', 587  
    else:  
    return 'smtp.' + domain, 587
    
    username = input("Identifiant (adresse mail) : ")  
    password = getpass("Mot de passe (ou mot de passe d'application Google) : ")  
    smtp_server, smtp_port = get_smtp_server_and_port(username)
    
    destinataire = input("Destinataire : ")  
    sujet = input("Sujet : ")  
    corps = input("Corps du mail : ")
    
    msg = MIMEText(corps)  
    msg['Subject'] = sujet  
    msg['From'] = username  
    msg['To'] = destinataire
    
    with smtplib.SMTP(smtp_server, smtp_port) as server:  
    server.starttls()  
    server.login(username, password)  
    server.sendmail(username, [destinataire], msg.as_string())
    
    print("Mail envoyé avec succès !")



### Points importants

- **Pour le domaine `laplateforme.io`**, le script utilise automatiquement le serveur SMTP Google (`smtp.gmail.com` port 587).
- Gmail **bloque les connexions SMTP classiques avec mot de passe standard** pour des raisons de sécurité.
- Pour contourner cela, il est **obligatoire de créer un mot de passe d'application** dans ton compte Google.
- Ce mot de passe d’application est différent du mot de passe principal de ton compte Gmail et permet une authentification sécurisée par SMTP.
- Pour créer un mot de passe d’application Google, rendez-vous sur : https://myaccount.google.com/security > Section "Mots de passe d’application".



## Exécution du script

1. Copier le script dans un fichier `send_mail.py`.
2. Lancer dans un terminal avec :

```
python3 mail.py
```

3. Saisir les informations demandées (adresse mail, **mot de passe d’application**, destinataire, sujet, message).
4. Le mail sera envoyé avec succès si l’authentification est correcte.



### Conseils

- Pour les autres domaines (Yahoo, Outlook, etc.), le script détecte automatiquement le serveur SMTP.
- Pour des domaines non listés, il essaie de préfixer le domaine de l’email par "smtp." par défaut.
- Adapte le script si besoin pour ajouter d’autres domaines spécifiques.










