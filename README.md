## Installation des diverses services usuelles (à faire 1 fois pour toute)

### Création d'un compte «utilisateur» pour gérer l'application web (code source) 

```
$adduser web
$usermod -a -G www-data web
```

### Installation du serveur web Nginx
```
$apt-get install nginx-full
```
### Installation de supervisor qui permettra de gérer le processus UWSGI (interpréteur python pour le web)
```
$apt-get install supervisor
```
### Installation de pip(3) et de virtualenvwrapper
```
$apt-get install python3-pip
$pip install virtualenvwrapper
```

* (en «web») 
* ajouter à la fin de .bashrc : 
* `export VIRTUALENVWRAPPER_PYTHON=/usr/bin/python3`
* `source /usr/local/bin/virtualenvwrapper.sh`
* `source .bashrc`

## Installation de l'applicatif
```
$git clone <url_du_repository> <nom_du_projet>
$mkvirtualenv <nom_du_projet> -a <nom_du_projet>
$pip3 install -r requirements.txt
$pip3 install uwsgi
```
Pour tester le site à l'arrache
```
$./manage.py runserver 0.0.0.0:8000
$./manage.py collectstatic
```

Dans settings\.py
```
DEBUG = False
```

Dans le extra_settings.py:

```
ALLOWED_HOST = ["XXX.XXX.XXX.XXX","nom_de_domaine.tld"]
```

Synchro de media depuis le PC de dev:
```
scp -r media web@<IP_DU_SERVEUR>:/home/web/<nom_du_projet>
```

### Création de <nom_du_projet>.ini pour la conf supervisor/uwsgi
```
[uwsgi]
chdir = /home/web/<nom_du_projet>/
socket = /home/web/<nom_du_projet>/<nom_du_projet>.sock
wsgi-file = /home/web/<nom_du_projet>/<app_de_coordination>/wsgi.py
processes = 4
virtualenv = /home/web/.virtualenvs/<nom_du_projet>/
master = true
uid = web
gid = www-data
chmod-socket = 666

ignore-sigpipe = true
ignore-write-errors = true
disable-write-exception = true
```

### Création de la conf nginx
```
upstream django-<nom_du_projet> {
    server unix:///home/web/<nom_du_projet>/<nom_du_projet>.sock;
}

server {
    server_name <nom_de_domaine>;
    charset utf-8;

    access_log /home/web/<nom_du_projet>/nginx.access.log;
    error_log  /home/web/<nom_du_projet>/nginx.error.log;


    client_max_body_size 75M;
    sendfileon;
    keepalive_timeout  0;
    proxy_buffering on;
    proxy_buffer_size 8k;
    proxy_buffers 2048 8k;


    location /media  {
        alias /home/web/<nom_du_projet>/media;
    }

    location /static {
        alias /home/web/<nom_du_projet>/static;
    }

    location / {
        uwsgi_pass  django-<nom_du_projet>;
        include /etc/nginx/uwsgi_params;
    }
}
```

### Conf supervisor : <nom_du_projet>.conf
```
[program:<nom_du_projet>]
user = web
command=/home/web/.virtualenvs/<nom_du_projet>/bin/uwsgi /home/web/<nom_du_projet>/<nom_du_projet>.ini
autostart=true
autorestart=true
stderr_logfile = /home/web/<nom_du_projet>/uwsgi-err.log
stdout_logfile = /home/web/<nom_du_projet>/uwsgi-out.log
stopsignal=INT
```

### Activation des configs (en «root»)

Dans /etc/nginx/sites-enabled:
```
ln -s /home/web/<nom_du_projet>/<nom_de_domaine.tld>
```

Dans /etc/supervisor/conf.d/
```
ln -s /home/web/<nom_du_projet>/<nom_du_projet>.conf
```
###  Redémarrage des services
```
$service nginx restart
$supervisorctl reload
```







