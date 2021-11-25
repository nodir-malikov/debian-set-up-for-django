# Debian Server Set Up for Django Instruction

In this guide we will set up clean Debian server for Python and Django projects. We will configure secure SSH connection, install from Debian repositories and from sources all needed packages and ware it together for working Debian Django server.

[Youtube video guide (in Russian)](https://www.youtube.com/watch?v=FLiKTJqyyvs)

## Create user, setup SSH

Connect through SSH to remote Debian server and create user 'www' if not exists
```
sudo adduser www
sudo usermod -aG sudo www
```

Update repositories and install some initial needed packages:

```
sudo apt-get update ; \
sudo apt-get install -y vim mosh tmux htop git curl wget unzip zip gcc build-essential make
```

Configure SSH:

Type this and press only Enter:
```
sudo ssh-keygen
```
Then run this:
```
sudo ssh-keygen
sudo ssh-copy-id www@SERVER_IP
```
Then:
```
sudo vim /etc/ssh/sshd_config
    AllowUsers www
    PermitRootLogin no
    PasswordAuthentication no
```

Restart SSH server, change `www` user password:

```
sudo service ssh restart
sudo passwd www
```

## Init — must-have packages & ZSH

```
sudo apt-get install -y zsh tree redis-server nginx zlib1g-dev libbz2-dev libreadline-dev llvm libncurses5-dev libncursesw5-dev xz-utils tk-dev liblzma-dev python3-dev python-pil python3-lxml python-libxml2 libffi-dev libssl-dev python-dev-is-python2 gnumeric libsqlite3-dev libpq-dev libxml2-dev libxslt1-dev libjpeg-dev libfreetype6-dev libcurl4-openssl-dev supervisor libwebp-dev webp certbot python3-certbot-nginx
```

Install [oh-my-zsh](https://github.com/robbyrussell/oh-my-zsh):

```
sh -c "$(curl -fsSL https://raw.githubusercontent.com/robbyrussell/oh-my-zsh/master/tools/install.sh)"
```

## Install python 3.10

mkdir ~/code

Build from source python 3.10, install with prefix to ~/.python folder:

```
wget https://www.python.org/ftp/python/3.10.0/Python-3.10.0.tgz ; \
tar xvf Python-3.10.* ; \
cd Python-3.10.0 ; \
mkdir ~/.python ; \
./configure --enable-optimizations --prefix=/home/www/.python ; \
make -j8 ; \
sudo make altinstall
```

Configure some needed aliases:

```
vim ~/.zshrc
    alias cls="clear"
    export PATH=$PATH:/home/www/.python/bin
```
Then run `. ~/.zshrc`
Now python3.10 in `/home/www/.python/bin/python3.10`. Update pip:

```
sudo python3.10 -m pip install -U pip
```

Ok, now we can pull our project from Git repository (or create own), create and activate Python virtual environment:

```
cd code
git pull project_git
cd project_dir
python3.10 -m venv venv
. ./env/bin/activate
```

## Install and configure PostgreSQL

Install PostgreSQL 13 and configure locales.

```
wget --quiet -O - https://www.postgresql.org/media/keys/ACCC4CF8.asc | sudo apt-key add - ; \
RELEASE=$(lsb_release -cs) ; \
echo "deb http://apt.postgresql.org/pub/repos/apt/ ${RELEASE}"-pgdg main | sudo tee  /etc/apt/sources.list.d/pgdg.list ; \
sudo apt update ; \
sudo apt -y install postgresql-13 ; \
sudo localedef ru_RU.UTF-8 -i ru_RU -fUTF-8 ; \
export LANGUAGE=ru_RU.UTF-8 ; \
export LANG=ru_RU.UTF-8 ; \
export LC_ALL=ru_RU.UTF-8 ; \
sudo locale-gen ru_RU.UTF-8 ; \
sudo dpkg-reconfigure locales
```

Add [arch=amd64] after "deb" if Ubuntu

Add locales to `/etc/profile`:

```
sudo vim /etc/profile
    export LANGUAGE=ru_RU.UTF-8
    export LANG=ru_RU.UTF-8
    export LC_ALL=ru_RU.UTF-8
```

Change `postges` password, create clear database named `dbms_db`:

```
sudo passwd postgres
su - postgres
export PATH=$PATH:/usr/lib/postgresql/13/bin
createdb --encoding UNICODE dbms_db --username postgres
exit
```

Create `dbms` db user and grand privileges to him:

```
sudo -u postgres psql
postgres=# ...
create user dbms with password 'some_password';
ALTER USER dbms CREATEDB;
grant all privileges on database dbms_db to dbms;
\c dbms_db
GRANT ALL ON ALL TABLES IN SCHEMA public to dbms;
GRANT ALL ON ALL SEQUENCES IN SCHEMA public to dbms;
GRANT ALL ON ALL FUNCTIONS IN SCHEMA public to dbms;
CREATE EXTENSION pg_trgm;
ALTER EXTENSION pg_trgm SET SCHEMA public;
UPDATE pg_opclass SET opcdefault = true WHERE opcname='gin_trgm_ops';
\q
exit
```

Now we can test connection. Create `~/.pgpass` with login and password to db for fast connect:

```
vim ~/.pgpass
	localhost:5432:dbms_db:dbms:some_password
chmod 600 ~/.pgpass
psql -h localhost -U dbms dbms_db
```

Run SQL dump, if you have:

```
psql -h localhost dbms_db dbms  < dump.sql
```

If you need access to db from PG Admin or something like this:

```
nano /etc/postgresql/13/main/postgresql.conf
# change the line - listen_addresses = '*'
nano /etc/postgresql/13/main/pg_hba.conf
# type after - # TYPE  DATABASE        USER            ADDRESS                 METHOD
# this - host    all             nodir           0.0.0.0/0               md5
```

## Install and configure supervisor

Now recommended way is using Systemd instead of supervisor. If you need supervisor — welcome:

```
sudo apt install supervisor

vim /home/www/code/project/bin/start_gunicorn.sh
	#!/bin/bash
	source /home/www/code/project/env/bin/activate
	exec gunicorn  -c "/home/www/code/project/gunicorn_config.py" project.wsgi

chmod +x /home/www/code/project/bin/start_gunicorn.sh

vim project/supervisor.salesbeat.conf
	[program:www_gunicorn]
	command=/home/www/code/project/bin/start_gunicorn.sh
	user=www
	process_name=%(program_name)s
	numprocs=1
	autostart=true
	autorestart=true
	redirect_stderr=true
```

If you need some Gunicorn example config — welcome:

```
command = '/home/www/code/project/env/bin/gunicorn'
pythonpath = '/home/www/code/project/project'
bind = '127.0.0.1:8001'
workers = 3
user = 'www'
limit_request_fields = 32000
limit_request_field_size = 0
raw_env = 'DJANGO_SETTINGS_MODULE=project.settings'
```

NGINX CONFIG EXAMPLE:

```
server {

        access_log /var/log/nginx/xabardor.log;

        root /var/www/html;

        index index.html index.html index.nginx-debian.html;

	server_name x.fullfocus.uz;

        location / {
                proxy_pass http://127.0.0.1:8001;
                proxy_set_header X-Forwarded-Host $server_name;
                proxy_set_header X-Real-IP $remote_addr;
                add_header P3P 'CP="ALL DSP COR PSAa PSDa OUR NOR ONL UNI COM NAV"';
		add_header Access-Control-Allow-Origin *;
        }
	
}
```


INSTALLING SSL for Selectel CDN server domain:

```
sudo certbot certonly --cert-name exampple_cert_name --manual -d yourdomain.com -d www.yourdomain.com --preferred-challenges dns
```

Follow the instructions given by certbot.

For each of the domain specified, create the TXT record with the values generated by certbot in your DNS provider of choice.


Then copy key files to your directory:

```
sudo cp -r /etc/letsencrypt/archive/yourdomain /home/yourpath
```

Now you must to concatinate Fullchain and Privatekey files to one smth.pem file and upload to Selectel Cloud Storage SSL Certificates page.
