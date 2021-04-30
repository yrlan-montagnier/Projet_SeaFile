# CLOUD - Projet Infra

**Le but de notre projet est la création d'un espace de stockage de fichier en ligne à l'aide de la plateforme SeaFile.**

**A la fin de ce projet nous pourrons nous connecter à un site web, où nous aurons la possibilité de stocker des fichiers dans un espace personnel et dédié.**


## Prérequis

- Machine sous Debian 10 sans interface graphique
- Un accès à internet
- Ouvrir les ports 80 et 443 du routeur
- Une carte bridge pour avoir une adresse IP dans le réseau physique
- Une carte nat pour l'accés à Internet

> Si vous ne voulez pas ouvrir les ports, vous pouvez aussi passer par un hébergeur qui vous livrera une machine où vous pourrez choisir les ports à ouvrir en toute sécurité.

## Procédure d'installation 

### Port forwarding et cartes réseau : 

**On a créé une carte bridge grâce au virtual network editor de vmware.**

**Pour ouvrir les port 80 et 443, on se rend sur l'interface de la box (192.168.1.1 pour box orange), onglet réseau/NAT puis on set les règles pour l'équipement qui correspond à l'IP de notre carte bridge de la VM ( vérifiée avec ip a)**

### Téléchargement de SeaFile et des dépendances

```
mkdir /opt/seafile
```

#### Téléchargement de  l'archive de Seafile :

```
cd /opt/seafile
wget O /var/safile-server_7.1.5_x86-64.tar.gz https://s3.eu-central-1.amazonaws.com/download.seadrive.org/seafile-server_7.1.5_x86-64.tar.gz
```

On décompresse les fichiers de l'archive, et on supprime ou déplace l'archive dans un dossier "installed" :
```
tar -xzf seafile-server_7.1.5_x86-64.tar.gz
mkdir installed
mv seafile-server_7.1.5_x86-64.tar.gz installed
```

#### Téléchargement et installation des dépendances :
```
apt-get install python3 python3-setuptools python3-pip -y nginx mariadb-server

pip3 install --timeout=3600 Pillow pylibmc captcha jinja2 sqlalchemy==1.3.8 \ django-pylibmc django-simple-captcha python3-ldap
```

#### Activation de Nginx et de MariaDB

```
systemctl enable nginx
systemctl start nginx
systemctl enable mariadb
```

### Création des bases de données et assignation des permissions à l'utilisateur "Seafile"

#### Setup de MariaDB:
**IMPORTANT : Il faut choisir "yes" pour toutes les étapes lors du Setup de MariaDB.**

```mysql_secure_installation``` ==> Script d'installation de MariaDB

#### Création des bases de données :
```
mysql -u root -p
create database `ccnet-db` character set = 'utf8';
create database `seafile-db` character set = 'utf8';
create database `seahub-db` character set = 'utf8';
```

#### Attribution des privilèges max à l'utilisateur de la BDD que l'on crée :
```
create user 'seafile'@'localhost' identified by 'Password123';
GRANT ALL PRIVILEGES ON `ccnet-db`.* to `seafile`@localhost;
GRANT ALL PRIVILEGES ON `seafile-db`.* to `seafile`@localhost;
GRANT ALL PRIVILEGES ON `seahub-db`.* to `seafile`@localhost;
FLUSH PRIVILEGES;
exit
```

### Installation et Configuration de SeaFile

#### Setup de Seafile

On donne les perms du dossier de Seafile à l'utilisateur seafile et on lance le setup depuis cet user

**IMPORTANT : Pendant le setup, on va nous demander :**
* **Le domaine (IP Publique ou dns dynamique), on met l'IP Publique car on va mettre en place le DNS plus tard.**
* **On laisse les ports par défaut**
* **Si l'on souhaite utiliser des bases de données prédéfinies (option 1) ou déja existantes (option 2)
Il faut choisir l'option 2 et préciser le nom des bases de données que nous avons crées plus haut : `ccnet-db`,`seafile-db`,`seahub-db`**

```
chown -R seafile:seafile /opt/seafile/seafile-server-7.1.5
cd /opt/seafile/seafile-server-latest
./setup-seafile-mysql.sh
mkdir /opt/seafile/logs
mkdir /opt/seafile/pids
chown -R seafile:seafile /opt/seafile/logs 
chown -R seafile:seafile /opt/seafile/pids
chown -R seafile:seafile /opt/seafile/seafile-server-latest
chown -R seafile:seafile /opt/seafile/seafile-data 
chown -R seafile:seafile /opt/seafile/ccnet 
chown -R seafile:seafile /opt/seafile/seahub-data 
chown -R seafile:seafile /opt/seafile/conf
```

#### Configuration de Seafile
**seafile.conf**

```nano /etc/nginx/sites-available/seafile.conf```
```
log_format seafileformat '$http_x_forwarded_for $remote_addr [$time_local] "$request" $status $body_bytes_sent "$http_referer" "$http_user_agent" $upstream_response_time';

server {
    listen 80;
    server_name ynov_cloud;

    proxy_set_header X-Forwarded-For $remote_addr;

    location / {
         proxy_pass         http://127.0.0.1:8000;
         proxy_set_header   Host $host;
         proxy_set_header   X-Real-IP $remote_addr;
         proxy_set_header   X-Forwarded-For $proxy_add_x_forwarded_for;
         proxy_set_header   X-Forwarded-Host $server_name;
         proxy_read_timeout  1200s;

         # used for view/edit office file via Office Online Server
         client_max_body_size 0;

         access_log      /var/log/nginx/seahub.access.log seafileformat;
         error_log       /var/log/nginx/seahub.error.log;
    }

    location /seafhttp {
        rewrite ^/seafhttp(.*)$ $1 break;
        proxy_pass http://127.0.0.1:8082;
        client_max_body_size 0;
        proxy_set_header   X-Forwarded-For $proxy_add_x_forwarded_for;

        proxy_connect_timeout  36000s;
        proxy_read_timeout  36000s;
        proxy_send_timeout  36000s;

        send_timeout  36000s;

        access_log      /var/log/nginx/seafhttp.access.log seafileformat;
        error_log       /var/log/nginx/seafhttp.error.log;
    }
    location /media {
        root /home/user/haiwen/seafile-server-latest/seahub;
    }
}
```
**On efface la config par défaut et on crée un raccourci dans sites-enabled qui pointe vers le fichier de config dans sites-available :**
```
rm /etc/nginx/sites-enabled/default
ln -s /etc/nginx/sites-available/seafile.conf /etc/nginx/sites-enabled/seafile.conf
```
**ccnet.conf**

```nano /opt/seafile/conf/ccnet.conf```
```
[General]
SERVICE_URL = http://ynovcloud.ddns.net:8000/

[Database]
ENGINE = mysql
HOST = 127.0.0.1
PORT = 3306
USER = seafile
PASSWD = Password123
DB = ccnet-db
CONNECTION_CHARSET = utf8
```

### Mise en place d'un DNS Dynamique 

**Pour mettre en place une adresse fixe pour notre espace de stockage, on a utilisé de site no-ip et son logiciel, qui nous permet de rediriger notre ip publique sur une adresse fixe qui est ynovcloud.ddns.net**

1. On s'inscrit sur https://www.noip.com/ et on crée le domaine ynovcloud.ddns.net
2. On installe le client de NoIP pour rediriger l'IP publique sur ynovcloud.ddns.net
3. On se rend dans l'interface de configuration de la box : http://192.168.1.1/
4. Onglet Réseau/DynDNS ( dépend de la box internet ou routeur )
5. On rentre les logs de noip.com ainsi que l'URL ynovcloud.ddns.net dans "nom d'hôte"

### Lancement et contrôle du serveur 

**Pour lancer le serveur, il faut se connecter avec l'utilisateur attribués à la BDD et aux fichiers de Seafile.**
1. Pour le faire, on se met déja en root `su -` puis on fais un `su - seafile`.
2. Puis on fais un `cd /opt/seafile/seafile-server-latest`
3. Enfin on peut utiliser les commandes `./seafile.sh start/stop/restart` et `./seahub.sh start/stop/restart`

## Connexion au stockage en ligne

**Pour accéder à l'espace de stockage en ligne qui vient d'être installé sur votre machine :**
1. Entrer l'adresse du serveur (http://ynovcloud.ddns.net/) que nous avons mis en place à l'aide de noip.com dans un navigateur internet.
2. Vous pouvez ensuite créer un compte ou vous connecter.