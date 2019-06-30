## Installation des diverses services usuelles (à faire 1 fois pour toute)

### Création d'un compte «utilisateur» pour gérer l'application web (code source) 
    
    $adduser web
    $usermod -a -G www-data web`

### Installation du serveur web Nginx

    $apt-get install nginx-full

### Installation de supervisor qui permettra de gérer le processus UWSGI (interpréteur python pour le web)

    $apt-get install supervisor

    $apt-get install python3-pip
    $pip install virtualenvwrapper

   * (en «web») 
       * ajouter à la fin de .bashrc : 
           * `export VIRTUALENVWRAPPER\_PYTHON=/usr/bin/python3`
           * `source /usr/local/bin/virtualenvwrapper.sh`
       * `source .bashrc`

## Installation de l'applicatif

    $git clone <url\_du\_repository> <nom\_du\_projet>
    $mkvirtualenv <nom\_du\_projet> -a <nom\_du\_projet>
    $pip3 install -r requirements.txt
    $pip3 install uwsgi

Pour tester le site à l'arrache

    $./manage.py runserver 0.0.0.0:8000
    $./manage.py collectstatic
    
Dans settings.py

    DEBUG = False

Dans le extra\_settings.py:

    ALLOWED\_HOST = ["XXX.XXX.XXX.XXX","nom\_de\_domaine.tld"]

Synchro de media depuis le PC de dev:

    scp -r media web@<IP\_DU\_SERVEUR>:/home/web/<nom\_du\_projet>

### Création de <nom\_du\_projet>.ini pour la conf supervisor/uwsgi

    [uwsgi]
    chdir = /home/web/<nom\_du\_projet>/
    socket = /home/web/<nom\_du\_projet>/<nom\_du\_projet>.sock
    wsgi-file = /home/web/<nom\_du\_projet>/<app\_de\_coordination>/wsgi.py
    processes = 4
    virtualenv = /home/web/.virtualenvs/<nom\_du\_projet>/
    master = true
    uid = web
    gid = www-data
    chmod-socket = 666

    ignore-sigpipe = true
    ignore-write-errors = true
    disable-write-exception = true

### Création de la conf nginx

    upstream django-<nom\_du\_projet> {
        server unix:///home/web/<nom\_du\_projet>/<nom\_du\_projet>.sock;
    }

    server {
        server\_name <nom\_de\_domaine>;
        charset     utf-8;

        access\_log /home/web/<nom\_du\_projet>/nginx.access.log;
        error\_log  /home/web/<nom\_du\_projet>/nginx.error.log;


        client\_max\_body\_size 75M;
        sendfile        on;
        keepalive\_timeout  0;
        proxy\_buffering on;
        proxy\_buffer\_size 8k;
        proxy\_buffers 2048 8k;


        location /media  {
            alias /home/web/<nom\_du\_projet>/media;
        }

        location /static {
            alias /home/web/<nom\_du\_projet>/static;
        }

        location / {
            uwsgi\_pass  django-<nom\_du\_projet>;
            include     /etc/nginx/uwsgi\_params;
        }
    }

### Conf supervisor : <nom\_du\_projet>.conf

    [program:<nom\_du\_projet>]
    user = web
    command=/home/web/.virtualenvs/<nom\_du\_projet>/bin/uwsgi /home/web/<nom\_du\_projet>/<nom\_du\_projet>.ini
    autostart=true
    autorestart=true
    stderr\_logfile = /home/web/<nom\_du\_projet>/uwsgi-err.log
    stdout\_logfile = /home/web/<nom\_du\_projet>/uwsgi-out.log
    stopsignal=INT

### Activation des configs (en «root»)

Dans /etc/nginx/sites-enabled:

    ln -s /home/web/<nom\_du\_projet>/<nom\_de\_domaine.tld>Dans /etc/supervisor/conf.d/

    ln -s /home/web/<nom\_du\_projet>/<nom\_du\_projet>.conf

###  Redémarrage des services
    service nginx restart
    supervisorctl reload







