# Debian Server Set Up for Django Instruction

In this guide we will set up clean Debian server for Python and Django projects. We will configure secure SSH connection, install from Debian repositories and from sources all needed packages and ware it together for working Debian Django server.

[Youtube video guide (in Russian)](https://www.youtube.com/watch?v=FLiKTJqyyvs)

## Create user, setup SSH

Connect through SSH to remote Debian server and create user 'www' if not exists
```
sudo adduser www ; \
sudo usermod -aG sudo www
```

Switch to www:
```
su - www
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
sudo apt-get install -y zsh tree redis-server nginx zlib1g-dev libbz2-dev libreadline-dev llvm libncurses5-dev libncursesw5-dev xz-utils tk-dev liblzma-dev python3-dev python-pil python3-lxml python-libxml2 libffi-dev libssl-dev python-dev-is-python2 gnumeric libsqlite3-dev libpq-dev libxml2-dev libxslt1-dev libjpeg-dev libfreetype6-dev libcurl4-openssl-dev supervisor libwebp-dev webp certbot python3-certbot-nginx pgbouncer
```

Install [oh-my-zsh](https://github.com/robbyrussell/oh-my-zsh):

```
sh -c "$(curl -fsSL https://raw.githubusercontent.com/robbyrussell/oh-my-zsh/master/tools/install.sh)"
```

## Install python 3.10

mkdir ~/code

Build from source python 3.10, install with prefix to ~/.python folder:

```
wget https://www.python.org/ftp/python/3.10.8/Python-3.10.8.tgz ; \
tar xvf Python-3.10.* ; \
cd Python-3.10.8 ; \
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

cd
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

## PGBOUNCER for PostgreSQL
Install pgbouncer:

```
sudo apt-get install pgbouncer
```

Change settings:
```
sudo nano /etc/pgbouncer/pgbouncer.ini
	# paste after [databases]
	* = host=localhost port=5432
	# search "pool_mode"
	pool_mode = transaction
	# search "max_client_conn"
	max_client_conn = 1000
	# search "auth_type"
	auth_type = md5
	# search "admin_users"
	admin_users = dbms
```

Get md5 hash:

```
echo "md5"$(echo -n 'Passworddbms' | md5sum | awk '{print $1}')
```

Edit this:

```
sudo nano /etc/pgbouncer/userlist.txt
	"dbms" "yourhash"
```

Restart PGBouncer:

```
sudo service pgbouncer restart
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

vim /etc/supervisor/conf.d/supervisor.salesbeat.conf
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

        access_log /var/log/nginx/x.log;

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
sudo cp -r /etc/letsencrypt/archive/your_cert_name /home/yourpath
```

Now you must to concatinate Fullchain and Privatekey files to one smth.pem file and upload to Selectel Cloud Storage SSL Certificates page.

```
sudo chmod 777 your_cert_name/

cd /home/yourpath/your_cert_name

cat fullchain1.pem privkey1.pem > your_cert.pem
```


===================================================


## How to Automate PostgreSQL Database Backups in Linux
April 24, 2020 by Omelchenko Alexander
The solutions that are mentioned in this article work for any Linux distribution: Ubuntu, Red Hat, Open SUSE, CentOS, Debian, Oracle Linux, etc. However, the package installation commands may differ slightly.

The question of regular backups eventually comes up for any database management system. PostgreSQL is no exception. As a rule, the pg_dump utility, which is part of the PostgreSQL client, is used to create a hot backup.


In all the examples below, the postgres user will be utilized for working with pg_dump. You can specify a different user by using pg_dump -U <username>, and the password can be transferred with the environment variable PGPASSWORD=<password>. If you plan to connect to a non-local server, use the -h parameter to specify the host name.

**Minimal solution**
To perform a backup with this utility, just call it and specify the destination file:

```
su - postgres
```
```
pg_dump -U postgres db_name | gzip > backup_file.gz
```
You can add a cron job to run backups regularly. To this end, follow the steps below:

Create a backup storage folder
```
sudo mkdir -p /backups/postgresql
```
Provide the postgres user permissions to the directory
```
sudo chown postgres /backups/postgresql
```
Set the PostgreSQL user
```
su - postgres
```
Open crontab by running the command
```
crontab -e
```
Add this line at the end, replacing db_name with the name of your database
```
0 0 * * 0 pg_dump -U postgres db_name | gzip > /backups/postgresql/db_name.gz
```
This way, a backup will be performed on your server at midnight on Saturday. This is the most basic solution to a backup task.

Run the following commands to restore the database.

Set the PostgreSQL user
```
su - postgresl
```
Recreate the database if it exists
```
psql -c "drop database t1" && psql -c "create database t1;"
```
Restore the database from the archive
```
gunzip -c /backups/postgresql/db_name.gz | psql db_name
```


**Extended solution**
A good solution should at least encompass the following aspects:

• Creating a backup with a unique name
• File compression
• Encrypting the compressed file
• Receiving email notifications concerning backup results
• Deleting old backups
The official PostgreSQL wiki offers a good script for solving these tasks. However, the disadvantage of this script is the lack of email notifications.

To create backups with email notifications, you can use the script provided below. But before that, you need to install postfix mailutils. These are the steps to do this in Ubuntu:

```
sudo apt-get update
```
```
sudo apt-get install postfix mailutils
```
Also, please note that this script must be run by the postgres user. To log in under this user, run the command:

```
su - postgres
```


The most basic method for running the bash script for regular backups is shown below. In the beginning of the script, specify the backup storage directory, email notification address, and backup storage period.


```
#!/bin/sh

# parse database name and email recipient from arguments
database=$1 # first argument
email=$2 # second argument

# Database name
db_name=$database

# Backup storage directory
backup_folder=~/backups/$database

# Notification email address
recipient_email=$email

# Number of days to store the backup
keep_day=10

sqlfile=$backup_folder/$database-$(date +%d-%m-%Y_%H-%M-%S).sql
zipfile=$backup_folder/$database-$(date +%d-%m-%Y_%H-%M-%S).zip

#create backup folder
mkdir -p "$backup_folder"

# Create a backup

if pg_dump $db_name > $sqlfile ; then
   echo "Sql dump created for $database"
else
   echo "pg_dump return non-zero code" | mailx -s "No backup was created for $database!" $recipient_email
   exit
fi

# Compress backup
if gzip -c $sqlfile > $zipfile; then
   echo "The backup was successfully compressed for $database"
else
   echo "Error compressing backup" | mailx -s "Backup was not created for $database!" $recipient_email
   exit
fi

rm $sqlfile
echo $zipfile | mailx -s "Backup was successfully created for $database" $recipient_email

# Delete old backups
find $backup_folder -mtime +$keep_day -delete
```


You can schedule this script to run regularly as shown here, but instead of calling pg_dump, specify the path to the script.

And don't forget to add user to **mail** group:
```
sudo usermod -aG mail youruser
```

Also add `.mailrc` file with following parameters:
```
set smtp-use-starttls
set ssl-verify=ignore
set smtp=smtp://smtp.yandex.ru:465
set smtp-auth=yourmail@yandex.ru
set smtp-auth-user=yourmail@yandex.ru
set smtp-auth-password=2adv1q31v8a7
set from="Mail (My Server)"
```



=== BONUS ===


## How to upload/sync folder of backups with SELECTEL CLOUD STORAGE using CyberDuck


**Install CyberDuck**

Add repositories to sources.list

```
echo 'deb https://s3.amazonaws.com/repo.deb.cyberduck.io nightly main' | sudo tee -a /etc/apt/sources.list
echo 'deb https://s3.amazonaws.com/repo.deb.cyberduck.io stable main' | sudo tee -a /etc/apt/sources.list
```

Add key

```
sudo apt-key adv --keyserver keyserver.ubuntu.com --recv-keys FE7097963FEFBE72
```

Run this

```
sudo apt-get update
sudo apt-get install duck
```

Login as postgres

```
su - postgres
```	

Create folder

```
mkdir scripts
cd scripts
```

Create myscript.sh

```
SWIFT_USERNAME=123456_cyberduck                 # cloud user username
SWIFT_PASSWORD=hY21313311                       # cloud user pwd
SWIFT_AUTH_URL=auth.selcdn.ru                   # default do not touch
BACKUP_PATH=MyContainer/pgbackup/              # /<container_name>/<your_path>
LOCAL_PATH=/var/lib/postgresql/backups/         # your local folder for sync

# ENTER first input and say no with printf '\nn\n' then run sync cmd
# We use absolute path of duck package to run it because crontab may not found it - /opt/duck/bin/duck
printf '\nn\n' | /opt/duck/bin/duck --synchronize swift://$SWIFT_USERNAME@$SWIFT_AUTH_URL/$BACKUP_PATH $LOCAL_PATH -p $SWIFT_PASSWORD -q
```

Run chmod
```
chmod +x myscript.sh
```

Test it
```
. myscript.sh
```

Edit crontab
```
crontab -e
```

Add to crontab

For example at 23:30 everyday 

```
30 23 * * * (/bin/date && /var/lib/postgresql/scripts/myscript.sh) >> /var/lib/postgresql/scripts/myscript.log 2>&1
```

That's all!

Additionally:

Post about this - https://habr.com/ru/company/selectel/blog/254053/

CyberDuck profile config file - https://static.selectel.ru/kb/selectel-storage.cyberduckprofile (Optional. I didn't have any problem without this) 
