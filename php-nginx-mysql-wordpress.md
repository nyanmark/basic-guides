# How to install Wordpress (php-nginx-mysql) on Debian 11

The first step you want to perform on your server is to update the APT repository to it has a list of the latest files.
```
su - root
apt-get update
```
You can now either setup sudo with the commands below for a normal user i.e. joe or run all the commands as root without sudo
```
apt-get install sudo
/usr/sbin/usermod -aG sudo YOUR_USER
su - YOUR_USER
```
Next install iptables and iptables-persistent we will be using those to configure the firewall later on. We will remove nftables which is the default.
```
sudo apt-get remove --auto-remove nftables
sudo apt-get purge nftables
sudo apt-get install iptables iptables-persistent
```
Once we have done that we can move onto installing the database, for this guide we will be using mariadb.
```
sudo apt install mariadb-server
```
When mariadb has installed we can configure our database.
```
sudo mariadb
```
Once inside mysql cli run the following commands.
```
CREATE DATABASE wordpress;
CREATE USER 'wordpress'@'%' identified by 'StrongPassword';
GRANT ALL PRIVILEGES ON wordpress.* TO 'wordpress'@'%';
FLUSH PRIVILEGES;
QUIT;
```
The database should now be created, if you want to allow external connections edit the file /etc/mysql/mariadb.conf.d/50-server.cnf using nano or vi.
```
Replace bind-address = 127.0.0.1 with
bind-address = 0.0.0.0
```
Now you can verify that MariaDB is running on 0.0.0.0 by running commands below.
```
sudo systemctl restart mysql
ss -tln
```
To install PHP on your instance run these commands.
```
sudo apt-get install php php-{fpm,pear,cgi,common,zip,mbstring,net-socket,gd,xml-util,mysql,bcmath}
```
If you want to run your PHP on a seperate node to your webserver edit the file /etc/php/7.4/fpm/pool.d/www.conf replace 7.4 with the php version.
```
Replace listen = /var/run/php7.4-fpm.sock with
listen = 0.0.0.0:9000
```
Finally restart the php-fpm service and verify port 9000 is running on 0.0.0.0.
```
sudo systemctl restart php7.4-fpm
ss -tln
```
Now you should be ready for wordpress itself. First you will need to install nginx.
```
sudo apt-get install nginx
```
Now we can install wordpress.
```
wget https://wordpress.org/latest.tar.gz
tar xvf latest.tar.gz
sudo mv wordpress /var/www/html/wordpress
cd /var/www/html/wordpress
sudo cp wp-config-sample.php wp-config.php
```
We are now ready to edit the wordpress configuration, edit the file wp-config.php with nano or vi.
```
define('DB_NAME', 'wordpress');
define('DB_USER', 'wordpress');
define('DB_HOST', 'IP_ADDRESS'); #EDIT THE IP TO YOUR OWN!
define('DB_PASSWORD', 'StrongPassword');
```
You can now set the file permissions using chown. Remember if running nginx and php on seperate servers both servers need to contain the files.
```
sudo chown -R www-data:www-data /var/www/html/wordpress
```
Finally configure nginx, first delete the default config and create a new config for the site.
```
sudo rm /etc/nginx/sites-enabled/default
sudo nano /etc/nginx/sites-enabled/wordpress.conf
```
Fill the file with the following configuration.
```
server {
    listen 80;
    server_name _;
    index index.php index.html index.htm;
    root /var/www/html/wordpress;
    server_tokens off;
    client_max_body_size 75M;

    # logging
    access_log /var/log/nginx/wordpress.access.log;
    error_log  /var/log/nginx/wordpress.error.log;

    # some security headers ( optional )
    add_header X-Frame-Options "SAMEORIGIN" always;
    add_header X-XSS-Protection "1; mode=block" always;
    add_header X-Content-Type-Options "nosniff" always;
    add_header Referrer-Policy "no-referrer-when-downgrade" always;
    add_header Content-Security-Policy "default-src * data: 'unsafe-eval' 'unsafe-inline'" always;

    location / {
        try_files $uri $uri/ /index.php$is_args$args;
    }

    location ~ \.php$ {
        try_files $uri = 404;
        fastcgi_split_path_info ^(.+\.php)(/.+)$;
        fastcgi_pass PHP-SERVER-IP:9000; #REMEMBER TO EDIT ME TO YOUR SERVER IP!
        fastcgi_index index.php;
        include fastcgi_params;
        fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
        fastcgi_param PATH_INFO $fastcgi_path_info;
    }

    location ~ /\.ht {
        deny all;
    }

    location = /favicon.ico {
        log_not_found off; access_log off;
    }

    location = /favicon.svg {
        log_not_found off; access_log off;
    }

    location = /robots.txt {
        log_not_found off; access_log off; allow all;
    }

    location ~* \.(css|gif|ico|jpeg|jpg|js|png)$ {
        expires max;
        log_not_found off;
    }
}
```
Now you can now check wether your configuration is working and restart nginx using.
```
sudo nginx -t
sudo systemctl restart nginx
```
Visit your webserver IP to test wether it is working once confirmed to be working setup your iptables.
```
sudo iptables -A INPUT -m state --state ESTABLISHED,RELATED -j ACCEPT
# ^ Requited on all servers 
sudo iptables -A INPUT -p tcp --dport 80 -j ACCEPT 
# ^ Required for nginx
sudo iptables -A INPUT -p tcp --dport 443 -j ACCEPT
# ^ Required for nginx
sudo iptables -A INPUT -p tcp --dport 9000 -j ACCEPT
# ^ Required for php
sudo iptables -A INPUT -p tcp --dport 3306 -j ACCEPT
# ^ Required for mysql
sudo iptables -P INPUT DROP
# ^ Requited on all servers 
```
You can also set up OUTPUT and FORWARDING policies however, that is outside the scope of this guide. Save the iptables rules.
```
sudo -s -H
iptables-save > /etc/iptables/rules.v4
```
