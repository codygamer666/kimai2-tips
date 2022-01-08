# INSTALL PRE-REQUISITES

sudo apt-get update
sudo apt-get upgrade
sudo apt-get install git unzip curl vim
sudo apt-get install php-fpm php-cli php-common php-json php-opcache php-readline php-xml php-zip php-intl php-gd php-mbstring php-mysql php-curl
sudo apt-get install mariadb-server mariadb-client
sudo apt-get install nginx
sudo apt-get install composer

# CREATE DATABASE AND DATABASE USER

sudo mysql -u root -p
CREATE DATABASE kimai;
GRANT ALL ON kimai.* TO 'kimai'@'localhost' IDENTIFIED BY 'toor';
exit

# INSTALL KIMAI
sudo su
cd /var/www/
git clone -b 1.12 --depth 1 https://github.com/kevinpapst/kimai2.git
cd kimai2/
composer install --no-dev --optimize-autoloader

# CONFIGURE THE DB CONNECTION IN THE .env file
sudo nano /var/www/kimai2/.env

DATABASE_URL=mysql://kimai:toor@127.0.0.1:3306/kimai

# RUN KIMAI INSTALLER
sudo bin/console kimai:install -n

# CONFIGURE FILE PERMISSIONS (**use fix permissions at the bottom insted)
sudo chown -R :www-data .
sudo chmod -R g+r .
sudo chmod -R g+rw var/
sudo chmod -R g+rw public/avatars/

# CONFIGURE NGINX
unlink /etc/nginx/sites-enabled/default
sudo nano /etc/nginx/sites-available/kimai2

paste:

server {
    listen 80 default_server;
    listen [::]:80 default_server;
    server_name www.kimai.local;
    root /var/www/kimai2/public;
    index index.php;

    access_log off;
    log_not_found off;

    location ~ /\.ht {
        deny all;
    }

    location / {
        try_files $uri /index.php$is_args$args;
    }

    location ~ ^/index\.php(/|$) {
        fastcgi_pass unix:/run/php/php7.4-fpm.sock;
        fastcgi_split_path_info ^(.+\.php)(/.*)$;
        include fastcgi.conf;
        fastcgi_param PHP_ADMIN_VALUE "open_basedir=$document_root/..:/tmp/";
        internal;
    }

    location ~ \.php$ {
        return 404;
    }
}

#endpaste#

ln -s /etc/nginx/sites-available/kimai2 /etc/nginx/sites-enabled/kimai2
nginx -t && service nginx reload

# CREATE THE FIRST USER
sudo bin/console kimai:create-user admin your@email.com ROLE_SUPER_ADMIN

pass: yourpassword

# OPTIONAL SSL (Let's Encrypt)
sudo apt-get install certbot python3-certbot-nginx
sudo certbot --nginx

# OPTIONAL SSL (Self Signed)
coming soon


# INSTALLATION COMPLETED. OPEN BROWSER AND NAVIGATE TO http://yourip/kimai/


######################################## Fix Permissions ####################################
sudo nano /var/www/kimai2/cache.sh
#paste#

#!/bin/bash

if [[ ! -d "var/" || ! -d "var/cache/prod/" ]];
then
 echo "Cache directory does not exist at: var/cache/prod/"
 exit 1
fi

if [[ ! -f "bin/console" ]];
then
 echo "Kimai console does not exist at: bin/console"
 exit 1
fi

rm -r var/cache/prod/*
bin/console kimai:reload --env=prod
chgrp -R www-data .
chmod -R g+r .
chmod -R 775 var/
chmod -R g+rw public/avatars/

#endpaste#
Run the script ./cache.sh
