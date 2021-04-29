# CLOUD - Projet Infra

**Le projet est une création d'un stockage de fichier en ligne à l'aide de la plateforme SeaFile.**

**A la fin de ce projet nous pourrons nous connecter à un site web, où nous aurons la possibilité de stocker des fichiers dans un espace personnel et dédié.**


## Prérequis

- Machine sous Debian 10
- Un accès à internet
- Ouvrir les ports 443 de votre routeur
> Si vous ne voulez pas ouvrir les ports, vous pouvez aussi passer par un hébergeur qui vous livrera une machine où vous pourrez choisir les ports à ouvrir en toute sécurité.

## Procédure d'installation 

### Installation de SeaFile

```mkdir /opt/seafile```

**Téléchargement de l'archive suivante :**

[https://s3.eu-central-1.amazonaws.com/download.seadrive.org/seafile-server_7.1.5_x86-64.tar.gz](https://s3.eu-central-1.amazonaws.com/download.seadrive.org/seafile-server_7.1.5_x86-64.tar.gz "https://s3.eu-central-1.amazonaws.com/download.seadrive.org/seafile-server_7.1.5_x86-64.tar.gz")

```tar -xzf seafile-server_7.1.5_x86-64.tar.gz```

```mkdir installed```

```mv seafile-server_7.1.5_x86-64.tar.gz installed```

```apt install mariadb-server -y```

```systemctl enable mariadb```

```apt-get install python3 python3-setuptools python3-pip -y```

```
pip3 install --timeout=3600 Pillow pylibmc captcha jinja2 sqlalchemy==1.3.8 \ django-pylibmc django-simple-captcha python3-ldap
```

### Installation et configuration de Nginx

```apt install nginx -y```

```systemctl start nginx```

```systemctl enable nginx```

```useradd -m -s /bin/bash seafile```

```mysql_secure_installation```

### Création de la base de donnée et assignation des permissions

```mysql -u root -p```

```
create database `ccnet-db` character set = 'utf8'; create database `seafile-db` character set = 'utf8'; create database `seahub-db` character set = 'utf8';
```

```
create user 'seafile'@'localhost' identified by 'Password123'; GRANT ALL PRIVILEGES ON `ccnet-db`.* to `seafile`@localhost; GRANT ALL PRIVILEGES ON `seafile-db`.* to `seafile`@localhost; GRANT ALL PRIVILEGES ON `seahub-db`.* to `seafile`@localhost; FLUSH PRIVILEGES; exit
```

```chown -R seafile:seafile /opt/seafile/seafile-server-7.1.5```

```./setup-seafile-mysql.sh```

```
chown -R seafile:seafile /opt/seafile/seafile-server-latest
chown -R seafile:seafile /opt/seafile/seafile-data 
chown -R seafile:seafile /opt/seafile/ccnet 
chown -R seafile:seafile /opt/seafile/seahub-data 
chown -R seafile:seafile /opt/seafile/conf
```

```mkdir /opt/seafile/logs```

```mkdir /opt/seafile/pids```

```
chown -R seafile:seafile /opt/seafile/logs 
chown -R seafile:seafile /opt/seafile/pids
```

### Configuration de SeaFile

```nano /etc/nginx/sites-available/seafile.conf```

```
log_format seafileformat '$http_x_forwarded_for $remote_addr [$time_local] "$request" $status $body_bytes_sent "$http_referer" "$http_user_agent" $upstream_response_time'; server { listen 80; server_name ynov_cloud; proxy_set_header X-Forwarded-For $remote_addr; location / { proxy_pass http://127.0.0.1:8000; proxy_set_header Host $host; proxy_set_header X-Real-IP $remote_addr; proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for; proxy_set_header X-Forwarded-Host $server_name; proxy_read_timeout 1200s; # used for view/edit office file via Office Online Server client_max_body_size 0; access_log /var/log/nginx/seahub.access.log seafileformat; error_log /var/log/nginx/seahub.error.log; } location /seafhttp { rewrite ^/seafhttp(.*)$ $1 break; proxy_pass http://127.0.0.1:8082; client_max_body_size 0; proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for; proxy_connect_timeout 36000s; proxy_read_timeout 36000s; proxy_send_timeout 36000s; send_timeout 36000s; access_log /var/log/nginx/seafhttp.access.log seafileformat; error_log /var/log/nginx/seafhttp.error.log; } location /media { root /home/user/haiwen/seafile-server-latest/seahub; } }
```

```rm /etc/nginx/sites-enabled/default```

```ln -s /etc/nginx/sites-available/seafile.conf /etc/nginx/sites-enabled/seafile.conf```

## Connexion au stockage en ligne

Pour accéder au stockage en ligne qui vient d'être installé sur votre machine, vous allez devoir entrer l'ip publique de votre machine dans un navigateur internet.