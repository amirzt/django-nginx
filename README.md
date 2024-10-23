# nginx
this is a simple step by step guide to deploy a django restframework api on an ubunto vps using nginx and gunicorn without too much explanation and just the steps<br />
back-end : django rest framework<br />
database : mysql<br />
vps : ubunto<br />
using nginx and gunicorn<br />

# set up ubunto
##
    sudo apt update
##
    sudo apt upgrade
##
    sudo apt install python3-venv python3-dev libpq-dev nginx curl


# setup MySQL
##
    sudo apt install mysql-server
##
    sudo systemctl start mysql
login to mysql and create the tables <br />
##
    sudo mysql -u root -p
##
    CREATE DATABASE your_mysql_db_name;
##
    CREATE USER 'your_mysql_user'@'localhost' IDENTIFIED BY 'your_mysql_password';
##
    GRANT ALL PRIVILEGES ON your_mysql_db_name.* TO 'your_mysql_user'@'localhost';
##
    FLUSH PRIVILEGES;
##
    exit


# setup virtual env
make working directory <br />
##
    mkdir api
##
    cd api

##
    python3 -m venv myprojectenv
##
    source myprojectenv/bin/activate
##
    pip install django gunicorn psycopg2-binary djangorestframework markdown django-filter pillow requests
##
    mkdir api
##
    cd api
##
    git init and get your code

# setup django
##
    nano appname/settings.py

add your database in settings<br />
```
DATABASES = {
    'default': {
        'ENGINE': 'django.db.backends.mysql',
        'NAME': 'your_mysql_db_name',
        'USER': 'your_mysql_user',
        'PASSWORD': 'your_mysql_password',
        'HOST': 'localhost',  # or '127.0.0.1'
        'PORT': '3306',
    }
}
```
add your allowedhost
<br />
``` 
ALLOWED_HOSTS = ['server_ip']
```
static files settings <br />
```
MEDIA_ROOT = os.path.join(BASE_DIR, 'media')
MEDIA_URL = '/media/'

STATIC_URL = '/static/'
STATIC_ROOT = os.path.join(BASE_DIR, 'static')
```

run and test

```
python manage.py collectstatic
python manage.py makemigrations
python manage.py migrate
python manage.py runserver 0.0.0.0:8000
```

# setup gunicorn
```
sudo nano /etc/systemd/system/gunicorn.socket
```
put this on the file and save
```
[Unit]
Description=gunicorn socket

[Socket]
ListenStream=/run/gunicorn.sock

[Install]
WantedBy=sockets.target
```
then
```
sudo nano /etc/systemd/system/gunicorn.service
```

put this on the file and save <br />
make sure to change the working directory and execstart url and .wsgi file name based on your project
```
[Service]
User=root
Group=www-data
WorkingDirectory=/root/api/api
ExecStart=/root/api/myprojectenv/bin/gunicorn \
          --access-logfile - \
          --workers 3 \
          --bind unix:/run/gunicorn.sock \
          APP.wsgi:application

[Install]
WantedBy=multi-user.target
```

```
sudo systemctl start gunicorn.socket
sudo systemctl enable gunicorn.socket
```

# setup nginx
```
sudo nano /etc/nginx/sites-available/myproject
```

place this on the file and save <br />
remeber to change addressed and ip based on your setup
```
server {
    listen 80;
    server_name SERVER_IP;

    location = /favicon.ico { access_log off; log_not_found off; }
    location /static/ {
        alias /root/api/api/static/;
    }

    location /media/ {
        autoindex on;
        alias /root/api/api/media/;
    }

    location / {
        include proxy_params;
        proxy_pass http://unix:/run/gunicorn.sock;
    }
}
```

```
sudo ln -s /etc/nginx/sites-available/myproject /etc/nginx/sites-enabled
```

```
sudo nginx -t
```
it must say that everything is ok

```
sudo systemctl daemon-reload
sudo systemctl restart gunicorn
sudo systemctl restart nginx
```
your server should be accessible now but the static files should not load <br />
to load static files you need to change the user that has access to the files. note that this is based on your need. you can set it to root or anything else. search about this yourself. for now we set it as root
```
sudo nano /etc/nginx/nginx.conf
```
change the first line to root and save and restart the nginx <br />
your server should work now

# setup domain
first you need to create A records for your domain using the vps ip <br />
then you can get ssl for the domain and setup the nginx <br />
first open nginx config file and add your domain
```
sudo nano /etc/nginx/sites-available/myproject
```
```
server {
    listen 80;
    server_name domain.com;
...
```
sudo apt install certbot python3-certbot-nginx
```
sudo certbot --nginx -d domain.com
```
this will get ssl certificate and automaticly set it to your nginx <br />
now you need to change your nginx
```
sudo nano /etc/nginx/sites-available/myproject
```
```
server {
    listen 80;
    server_name domain.com;

    client_max_body_size 800M;

    location = /favicon.ico { access_log off; log_not_found off; }
    location /static/ {
        alias /root/api/api/static/;
    }

    location /media/ {
        autoindex on;
        alias /root/api/api/media/;

        add_header 'Access-Control-Allow-Origin' '*' always;
        add_header 'Access-Control-Allow-Methods' 'GET, POST, OPTIONS' always;
        add_header 'Access-Control-Allow-Headers' 'Origin, X-Requested-With, Content-Type, Accept, Authorization' always;
        add_header 'Access-Control-Allow-Credentials' 'true' always;
}

    location / {
        include proxy_params;
        proxy_pass http://unix:/run/gunicorn.sock;

        if ($request_method = 'OPTIONS') {
            add_header 'Access-Control-Allow-Origin' '*' always;
            add_header 'Access-Control-Allow-Methods' 'GET, POST, OPTIONS, DELETE, PUT' always;
            add_header 'Access-Control-Allow-Headers' 'Origin, X-Requested-With, Content-Type, Accept, Authorization' always;
            add_header 'Access-Control-Allow-Credentials' 'true' always;
            add_header 'Access-Control-Max-Age' 1728000 always;
            add_header 'Content-Type' 'text/plain; charset=UTF-8' always;
            add_header 'Content-Length' 0 always;
            return 204;
        }

        add_header 'Access-Control-Allow-Origin' '*' always;
        add_header 'Access-Control-Allow-Methods' 'GET, POST, OPTIONS, DELETE, PUT' always;
        add_header 'Access-Control-Allow-Headers' 'Origin, X-Requested-With, Content-Type, Accept, Authorization' always;
        add_header 'Access-Control-Allow-Credentials' 'true' always;
    }
    listen 443 ssl; # managed by Certbot
    ssl_certificate /etc/letsencrypt/live/domain.com/fullchain.pem; # managed by Certbot
    ssl_certificate_key /etc/letsencrypt/live/domain.com/privkey.pem; # managed by Certbot
    include /etc/letsencrypt/options-ssl-nginx.conf; # managed by Certbot
    ssl_dhparam /etc/letsencrypt/ssl-dhparams.pem; # managed by Certbot

}
server {
    if ($host = domain.com) {
        return 301 https://$host$request_uri;
    } # managed by Certbot


    listen 80;
    server_name domain.com;
    return 404; # managed by Certbot


}
```
now you should set the domain in the setting of django as allowed_host and restart everything


