# PSMM

> Projet : Python, Shell, Mariadb, Mail

## SOMMAIRE
1. [FTP](#serveur-ftp)
2. [MariaDB](#server-mariadb)
3. [Web](#server-web)
4. [SSH](#ssh)
5. [Python](#python)
6. [Job 2](#mail)
7. [JOB 4](#JOB-4-ssh_login_sudo.py)
8. [JOB 5](#JOB-5-ssh_mysql.py)
9. [JOB 6](#JOB-6-ssh_mysql_error.py)
10. [JOB 7](#JOB-7-ssh_ftp_error.py)
11. [JOB 8](#JOB-8-ssh_web_error.py)
12. [JOB 9](#JOB-9-ssh_serveur_mail.py)


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


## Job 3

```
import paramiko

hostname = "192.168.42.129"
port = 22
username = "monitor"
key_path = "/home/monitor/.ssh/id_rsa"  # Mettre ici le chemin complet vers ta clé privée sécurisée

# Commande à exécuter sur la machine distante
command = "ls"

try:
    # Charger la clé privée
    key = paramiko.RSAKey.from_private_key_file(key_path)
    
    # Initialiser la connexion SSH
    client = paramiko.SSHClient()
    client.set_missing_host_key_policy(paramiko.AutoAddPolicy())
    
    # Connexion avec la clé privée (sans mot de passe)
    client.connect(hostname=hostname, port=port, username=username, pkey=key)
    
    # Exécution de la commande
    stdin, stdout, stderr = client.exec_command(command)
    
    # Affichage du résultat et erreurs s'il y en a
    output = stdout.read().decode()
    errors = stderr.read().decode()
    
    print(f"Output de '{command}':\n{output}")
    if errors:
        print(f"Erreurs :\n{errors}")
    
    client.close()
except Exception as e:
    print(f"Erreur lors de la connexion ou de l'exécution : {e}")
```

## JOB 4 ssh_login_sudo.py

    import  paramiko
    import  getpass
    hostname  =  "192.168.42.129"
    port  =  22
    username  =  "monitor"
    key_path  =  "/home/monitor/.ssh/id_rsa"  # Mettre ici le chemin vers ta clé privée SSH sécurisée
    sudo_password  =  getpass.getpass("Mot de passe sudo : ")
    command  =  "sudo cat /etc/shadow"  # commande nécessitant root
    
    try:
	    # Charger la clé privée
	    key  =  paramiko.RSAKey.from_private_key_file(key_path)
   
	    client  =  paramiko.SSHClient()
	    client.set_missing_host_key_policy(paramiko.AutoAddPolicy())
   
	    # Connexion avec la clé privée (sans mot de passe SSH)
	    client.connect(hostname=hostname, port=port, username=username, pkey=key)
    
	    # Préparation de la commande sudo (le -S pour lire mot de passe depuis stdin)
	    sudo_command  =  f"echo {sudo_password} | sudo -S {command}"
    
	    stdin, stdout, stderr  =  client.exec_command(sudo_command)
	    output  =  stdout.read().decode()
	    errors  =  stderr.read().decode()
   
	    print(f"Résultat de 'sudo {command}':\n{output}"
	    if  errors:
		    print(f"Erreurs :\n{errors}")
    
	    client.close()
    except  Exception  as  e:
		  print(f"Erreur : {e}")


## JOB 5 ssh_mysql.py

    import  paramiko
    hostname  =  "192.168.42.130"  
    port  =  22
    username  =  "monitor"
    key_path  =  "/home/monitor/.ssh/id_rsa"  # Chemin vers la clé privée SSH sécurisée

    # Identifiants MariaDB (à modifier avec les valeurs souhaitées)
    mysql_user  =  "monitor"
    mysql_pass  =  "root"
    
    # Commande pour afficher les bases de données
    mysql_command  =  f'mysql -u {mysql_user} -p"{mysql_pass}" -e "SHOW DATABASES;"'
    try:
	    # Charger la clé privée SSH
	    key  =  paramiko.RSAKey.from_private_key_file(key_path)
	    client  =  paramiko.SSHClient()
	    client.set_missing_host_key_policy(paramiko.AutoAddPolicy())
	    
	    # Connexion SSH avec clé privée
	    client.connect(hostname=hostname, port=port, username=username, pkey=key)

	    # Commande complète avec sudo (mot de passe sudo sera demandé manuellement si nécessaire)
	    full_command  =  f"sudo {mysql_command}"

	    stdin, stdout, stderr  =  client.exec_command(full_command)
	    output  =  stdout.read().decode()
	    errors  =  stderr.read().decode()
    
	    print("Résultat d'accès MariaDB :\n"  +  output)
	    if  errors:
		    print(f"Erreurs :\n{errors}")

	    client.close()
    except  Exception  as  e:
	    print(f"Erreur : {e}")

## JOB 6 ssh_mysql_error.py

    import  paramiko
    import  re
    import  mysql.connector
    from  getpass  import  getpass
    from  datetime  import  datetime
    
    # --- CONFIG SSH ---
    SSH_HOST  =  "192.168.42.130"
    SSH_PORT  =  22
    SSH_USER  =  "monitor"

    # --- CONFIG MySQL --- 
    DB_HOST  =  "192.168.42.130"
    DB_USER  =  "monitor"
    DB_NAME  =  "archive_db"
    DB_PORT  =  3306
    
    LOG_COMMAND  =  "sudo journalctl -u mariadb.service | grep 'Access denied'"
    
    def  get_mysql_errors_via_ssh(password_ssh):
    
	    """Récupère les lignes 'Access denied' via SSH"""
	    ssh  =  paramiko.SSHClient()
	    ssh.set_missing_host_key_policy(paramiko.AutoAddPolicy())
	    ssh.connect(SSH_HOST, port=SSH_PORT, username=SSH_USER, password=password_ssh)
    
    stdin, stdout, stderr  =  ssh.exec_command(LOG_COMMAND)
    lines  =  stdout.read().decode().splitlines()
    ssh.close()
    return  lines
    
    def  insert_errors_to_db(password_db, errors):
	    """Insère uniquement les nouvelles erreurs dans MariaDB"""
	    try:
		    connection  =  mysql.connector.connect(
			    host=DB_HOST,
			    user=DB_USER,
			    password=password_db,
			    database=DB_NAME,
			    port=DB_PORT
			   )
		    cursor  =  connection.cursor()
    
		    # Récupérer la dernière date déjà insérée
		    cursor.execute("SELECT MAX(date) FROM login_attempts")
		    last_date  =  cursor.fetchone()[0]

		    pattern  =  re.compile(r"Access denied for user '(.+)'@'(.+)' .*")
		    count  =  0
    
		    for  line  in  errors:
			    match  =  pattern.search(line)
			    if  match:
				    username, ip  =  match.groups()
				    now  =  datetime.now()
    
				    # Si on a une dernière date, ignorer les logs plus anciens
					  if  last_date  and  now  <=  last_date:
					    continue

				    cursor.execute(
					   "INSERT INTO login_attempts (username, ip, date, message) VALUES (%s, %s, %s, %s)",
					    (username, ip, now, line)
				    )
				    count  +=  1
    
      
    
			    connection.commit()
			    cursor.close()
			    connection.close()
			    print(f"{count} erreurs insérées dans la base.")

	    except  mysql.connector.Error as  e:
			    print("Erreur MySQL :", e)
    
    if  __name__  ==  "__main__":
	    ssh_password  =  getpass(f"Mot de passe SSH pour {SSH_USER}@{SSH_HOST}: ")
	    db_password  =  getpass(f"Mot de passe MySQL pour {DB_USER}@{DB_HOST}: ")

	    errors  =  get_mysql_errors_via_ssh(ssh_password)
    if  errors:
				 insert_errors_to_db(db_password, errors)
    else:
			   print("Aucune erreur Access denied trouvée.")


## JOB 7 ssh_ftp_error.py
  

    import  paramiko
    import  re
    import  mysql.connector
    from  getpass  import  getpass
    from  datetime  import  datetime
    
    # --- CONFIG SSH ---
    SSH_HOST  =  "192.168.52.131"
    SSH_PORT  =  22
    SSH_USER  =  "monitor"

    # --- CONFIG MySQL ---
    DB_HOST  =  "192.168.52.131"
    DB_USER  =  "monitor"
    DB_NAME  =  "ftp_logs_archive"
    DB_PORT  =  3306
    
    LOG_COMMAND  =  "sudo journalctl -u mariadb.service | grep 'Access denied'"
    
    def  get_mysql_errors_via_ssh(password_ssh):
	    """Récupère les lignes 'Access denied' via SSH"""
	    ssh  =  paramiko.SSHClient()
	    ssh.set_missing_host_key_policy(paramiko.AutoAddPolicy())
	    ssh.connect(SSH_HOST, port=SSH_PORT, username=SSH_USER, password=password_ssh)
    
	    stdin, stdout, stderr  =  ssh.exec_command(LOG_COMMAND)
	    lines  =  stdout.read().decode().splitlines()
	    ssh.close()
	    return  lines
	    
    def  insert_errors_to_db(password_db, errors):
	    """Insère uniquement les nouvelles erreurs dans MariaDB"""
		   try:
    
				connection  =  mysql.connector.connect(
			    host=DB_HOST,
			    user=DB_USER,
			    password=password_db,
			    database=DB_NAME,
			    port=DB_PORT
		    )
		    cursor  =  connection.cursor()
    
		    # Récupérer la dernière date déjà insérée
		    cursor.execute("SELECT MAX(date) FROM ftp_errors")
		    last_date  =  cursor.fetchone()[0]
    
		    pattern  =  re.compile(r"Access denied for user '(.+)'@'(.+)' .*")
		    count  =  0

		    for  line  in  errors:
			    match  =  pattern.search(line)
			    if  match:
				    username, ip  =  match.groups()
				    now  =  datetime.now()
    
				    # Si on a une dernière date, ignorer les logs plus anciens
				    if  last_date  and  now  <=  last_date:
					    continue
    
				    cursor.execute(
					    "INSERT INTO ftp_errors (username, ip, date, message) VALUES (%s, %s, %s, %s)",
					    (username, ip, now, line)
					   )
				    count  +=  1
    
			    connection.commit()
			    cursor.close()
			    connection.close()
			    print(f"{count} erreurs insérées dans la base.")
    
      
    
			except  mysql.connector.Error as  e:
				print("Erreur MySQL :", e)

    if  __name__  ==  "__main__":
	    ssh_password  =  getpass(f"Mot de passe SSH pour {SSH_USER}@{SSH_HOST}: ")
	    db_password  =  getpass(f"Mot de passe MySQL pour {DB_USER}@{DB_HOST}: ")
    
	    errors  =  get_mysql_errors_via_ssh(ssh_password)
	    if  errors:
		    insert_errors_to_db(db_password, errors)
	    else:
		    print("Aucune erreur Access denied trouvée.")


## JOB 8 ssh_web_error.py
  

    import  paramiko
    import  re
    import  mysql.connector
    from  getpass  import  getpass
    from  datetime  import  datetime

    # --- CONFIG SSH ---
    SSH_HOST  =  "192.168.52.133"
    SSH_PORT  =  22
    SSH_USER  =  "monitor"

    # --- CONFIG MySQL ---
    DB_HOST  =  "192.168.52.131"
    DB_USER  =  "monitor"
    DB_NAME  =  "ftp_logs_archive"
    DB_PORT  =  3306
    
    # Commande pour récupérer les erreurs Apache auth_basic
    LOG_COMMAND  =  "sudo tail -n 100 /var/log/apache2/error.log | grep 'auth_basic:error'"
    
    def  get_web_errors_via_ssh(password_ssh):
	    ssh  =  paramiko.SSHClient()
	    ssh.set_missing_host_key_policy(paramiko.AutoAddPolicy())
	    ssh.connect(SSH_HOST, port=SSH_PORT, username=SSH_USER, password=password_ssh)
    
	    stdin, stdout, stderr  =  ssh.exec_command(LOG_COMMAND)
	    lines  =  stdout.read().decode().splitlines()
	    ssh.close()
	    return  lines
    
    def  insert_web_errors_to_db(password_db, errors):
	    try:
		    connection  =  mysql.connector.connect(
			    host=DB_HOST,
			    user=DB_USER,
			    password=password_db,
			    database=DB_NAME,
			    port=DB_PORT
		    )
		    cursor  =  connection.cursor()
    
		    cursor.execute("SELECT MAX(date) FROM web_errors")
		    last_date  =  cursor.fetchone()[0]
    
			pattern  =  re.compile(r".*\[client (?P<ip>[\d\.]+):\d+\] .*user (?P<user>\S+) not found.*")

		    count  =  0
		    for  line  in  errors:
			    match  =  pattern.search(line)
			    if  match:
				    ip  =  match.group('ip')
				    username  =  match.group('user')
				    now  =  datetime.now()
    
				    if  last_date  and  now  <=  last_date:
					    continue

				    cursor.execute(
					    "INSERT INTO web_errors (username, ip, date, message) VALUES (%s, %s, %s, %s)",
    (username, ip, now, line)
				    )
				    count  +=  1
    

		    connection.commit()
		    cursor.close()
		    connection.close()
		    print(f"{count} erreurs insérées dans la table web_errors.")
    
		except  mysql.connector.Error as  e:
			print("Erreur MySQL :", e)
    
    if  __name__  ==  "__main__":
	    ssh_password  =  getpass(f"Mot de passe SSH pour {SSH_USER}@{SSH_HOST}: ")
	    db_password  =  getpass(f"Mot de passe MySQL pour {DB_USER}@{DB_HOST}: ")
    
      
    
	    errors  =  get_web_errors_via_ssh(ssh_password)
	    if  errors:
		    insert_web_errors_to_db(db_password, errors)
	    else:
		    print("Aucune erreur d'accès web trouvée.")


## JOB 9 ssh_serveur_mail.py

    import  pymysql
    import  smtplib
    from  email.mime.text  import  MIMEText
    from  datetime  import  datetime, timedelta
    
    # Config BDD
    sql_host  =  "192.168.42.130"
    sql_user  =  "monitor"
    sql_password  =  "root"
    sql_db  =  "archive_db"
    
    # Config mail
    smtp_server  =  "smtp.gmail.com"
    smtp_port  =  587
    smtp_user  =  "samuel.rigaux@laplateforme.io"  # Adresse expéditrice du mail
    smtp_password  =  "qnjvfduywrhjrnbg"
    mail_to  =  "samuel.rigaux@icloud.com"  # Destinataire admin
    
    # Calcul de la date d'hier
    yesterday  = (datetime.now() -  timedelta(days=1)).date()
    
    # Récupérer les tentatives de la veille
    conn  =  pymysql.connect(host=sql_host, user=sql_user, password=sql_password, database=sql_db)
    cursor  =  conn.cursor()

    cursor.execute("""
	    SELECT username, attempt_time, ip_address
	    FROM login_attempts
	    WHERE DATE(attempt_time) = %s
	    ORDER BY attempt_time
    """, (yesterday,))
    attempts  =  cursor.fetchall()
    conn.close()
    
    if  not  attempts:
	    mail_content  =  "Aucune tentative de connexion échouée enregistrée pour le "  +  str(yesterday)
    else:
	    mail_content  =  f"Résumé des tentatives échouées du {yesterday}:\n\n"
	    for  username, attempt_time, ip  in  attempts:
		    mail_content  +=  f"{attempt_time} | Utilisateur : {username} | IP : {ip}\n"
    
    # Construction du mail
    msg  =  MIMEText(mail_content)
    msg['Subject'] =  f'Rapport tentatives échouées du {yesterday}'
    msg['From'] =  smtp_user
    msg['To'] =  mail_to
    
    # Envoi SMTP
    server  =  smtplib.SMTP(smtp_server, smtp_port)
    server.starttls()
    server.login(smtp_user, smtp_password)
    server.sendmail(smtp_user, [mail_to], msg.as_string())
    server.quit()







