#### nginx install

```
sudo apt update
sudo apt install nginx
sudo ufw allow 'Nginx HTTP'
systemctl status nginx
sudo systemctl reload nginx

sudo mkdir -p /var/www/example.com/html
sudo chown -R $USER:$USER /var/www/example.com/html
sudo chmod -R 755 /var/www/example.com
sudo nano /etc/nginx/sites-available/example.com
sudo ln -s /etc/nginx/sites-available/example.com /etc/nginx/sites-enabled/
sudo systemctl restart nginx
```

#### example.com conf

```
upstream django {
    server unix:///var/www/dcna/dcna.sock fail_timeout=600s; 
}

server {
    listen 80;
	client_max_body_size 100M;
    server_name dcna.computationalbiology.org;
	
	client_body_timeout   600;
	client_header_timeout 600;

    location /static {
        alias   /var/www/dcna/static;
    }
	
	location  /uploads {
        alias /var/www/dcna/uploads;
    }

    location / {
		proxy_connect_timeout   600;
		proxy_send_timeout      600;
		proxy_read_timeout      600;
		keepalive_timeout     600;
		uwsgi_read_timeout 18000;
		uwsgi_pass  django;
        include     /etc/nginx/uwsgi_params;
   }

    listen 443 ssl; # managed by Certbot
    ssl_certificate /etc/letsencrypt/live/computationalbiology.org/fullchain.pem; # managed by Certbot
    ssl_certificate_key /etc/letsencrypt/live/computationalbiology.org/privkey.pem; # managed by Certbot
    include /etc/letsencrypt/options-ssl-nginx.conf; # managed by Certbot
    ssl_dhparam /etc/letsencrypt/ssl-dhparams.pem; # managed by Certbot
}

```

#### uWsgi

```
pip3 install uwsgi
nano conf/uwsgi_params
uwsgi  --uid www-data --gid www-data --ini conf/uwsgi.ini
(sudo pkill -f uwsgi -9)

sudo mkdir -p env/vassals
sudo ln -s conf/uwsgi.ini env/vassals/
uwsgi --emperor /var/www/dcna/env/vassals 
```

#### uwsgi.ini

```
[uwsgi]
# full path to Django project's root directory
chdir            = /var/www/dcna/
# Django's wsgi file
module           = config.wsgi
# full path to python virtual env
home             = /var/www/dcna/env
# enable uwsgi master process
master          = true
# maximum number of worker processes
processes       = 10
# the socket (use the full path to be safe
socket          = /var/www/dcna/dcna.sock

# socket permissions
chmod-socket    = 666
# clear environment on exit
vacuum          = true
# daemonize uwsgi and write messages into given log
daemonize       = /var/www/dcna/logs/uwsgi.log

harakiri = 1200
harakiri-verbose = true
post-buffering = 1
socket-timeout = 600
#max-worker-lifetime = 35
```

#### django static file settings

```
#STATIC_DIR = os.path.join(BASE_DIR, 'static')
STATIC_URL = '/static/'

STATICFILES_DIRS = [
    os.path.join(BASE_DIR, "static"),
    '/var/www/dcna/static/',
]

MEDIA_URL = '/uploads/'
MEDIA_ROOT = os.path.join(BASE_DIR, 'uploads')
```


#### gunicorn.conf (no need)

```
command = '/var/www/dcna/env/bin/gunicorn'
pythonpath = '/var/www/dcna/env/bin'
#bind = ['159.65.204.133:8000']
bind = 'unix:/var/www/dcna/myproject.sock'
workers = 5
accesslog ='/var/www/dcna/logs/access.log'
errorlog ='/var/www/dcna/logs/error.log'
timeout= 1200
worker_class = 'gevent'
log_level = 'info'
```


zip -r backup.zip . -x "env/*"
