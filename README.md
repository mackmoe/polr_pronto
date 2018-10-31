<img src="https://i.imgur.com/ckI6GTu.png" width="350px" alt="Polr Logo" />


:aerial_tramway: A modern, minimalist, and lightweight URL shortener.



### Polr, Pronto! - Standalone server

This isn't a fork or spin of the actual [Polr](https://github.com/cydrobolt/polr) app. I felt like the instructions on [how to install Polr](http://docs.polrproject.org/en/latest/user-guide/installation/) could use a lot more details on how to install it from end-to-end. 

The list below is a minimal step-by-step list of commands to run, to get Polr up and running locally on Ubuntu 16.04LTS and later versions. I will work on getting this written into a script in the future, for even easier setup.

For now, to get Polr up running, Pronto! Use the following commands (step-by-step) in the clode block below. When you finish running all of the cmds (without any errors) everything should be ready. Open your browser, and point it to either the servername used in the nginx config:


```

DATABASE INSTALL & SETUP - (Note: # == being-ran-as-root VS. $ == being-ran-as-sudo-user)
---
# apt update && apt install -y curl vim git mysql-server  <--- * Password is optional here because we'll set that in the next cmd
# usermod -d /var/lib/mysql/ mysql
# /etc/init.d/mysql start
# mysql_secure_installation
# mysql 
mysql> create database polr_db;
mysql> grant all privileges on polr_db.* to 'polr_dbu'@'localhost' identified by 'secret-password';
mysql> flush privileges;
mysql> exit;
# mysql -e "show databases;" && mysql -e "select User from mysql.user;"
 - You should see the new db and db user you created from the shell in the output from the last cmd that was ran -


WEBSERVER INSTALL AND SETUP
---
# apt install -y unzip nginx php-fpm php-mysql php-mbstring php-geoip php-json php-tokenizer php-curl php-cli php-mcrypt php-dom libzend-framework-zendx-php
# sed -i 's/;cgi.fix_pathinfo=1/cgi.fix_pathinfo=0/' /etc/php/7.0/fpm/php.ini
# vim /etc/nginx/sites-available/default  <-- * copy and paste the just the config block between the '--' symbols *
--
upstream php {
    server unix:/var/run/php-fpm.sock;
    server 127.0.0.1:9000;
}
server {
        listen 80 default_server;
        listen [::]:80 default_server;
        root /var/www/polr/public;
        
        # Add index.php to the list if you are using PHP
        index index.php index.html index.htm;
        
        server_name _;   <--- * You will need to use another valuse here - but it cannot be empty!
        
        location / {
                try_files $uri $uri/ /index.php$is_args$args;
        }
        # pass the PHP scripts to FastCGI server listening on 127.0.0.1:9000
        location ~ \.php$ {
                try_files $uri =404;
                include /etc/nginx/fastcgi_params;
        # With php7.0-cgi alone:
        #       fastcgi_pass 127.0.0.1:9000;
        # With php7.0-fpm:
                fastcgi_pass unix:/run/php/php7.0-fpm.sock;
        # Added extra parameters for polr:
                fastcgi_index   index.php;
                fastcgi_param   SCRIPT_FILENAME $document_root$fastcgi_script_name;
                fastcgi_param   HTTP_HOST       $server_name;
        }
}
--


APP INSTALL AND SETUP
---
# git clone https://github.com/cydrobolt/polr.git --depth=1 /var/www/polr
# cp /var/www/polr/.env.setup /var/www/polr/.env
# rm -f /var/www/polr/composer.lock
# cd /var/www/polr
# curl -sS https://getcomposer.org/installer | php
# php composer.phar install --no-dev -o
# chown www-data: -R /var/www/polr
# find /var/www/polr -exec chmod 775 '{}' \;
# systemctl restart nginx
# systemctl enable nginx mysql
# systemctl status nginx mysql php7.0-fpm

```


#### Bear in mind that you'll need to use a real string for the following variables:
 - $tld_or_ip (in Dockerfile)
 - $secret-password
 - http://192.168.1.X/setup
 - http://your-server.dom/setup


### Polr, Pronto! - With Docker

The polr docker app stack has to be ran with docker compose. However, if you would just like to use the web container -or- for testing purposes you can use the following command to start and run it. Then you can open your favorite browser and navigate to the Polr env Config page http://localhost/setup to verify the status of the container.

```

docker run -e 'WEBROOT=/var/www/html/public' --name polr_webapp -d -p 80:80 mackmoe/polr_docker

```



### Polr, Pronto! - With Docker Build

To run the docker container service for Polr; 

Using docker build:

```

git clone https://github.com/mackmoe/polr_docker.git
cd polr_docker
git clone https://github.com/cydrobolt/polr.git --depth=1 src
rm -rf src/.git && cp src/.env.setup src/.env
docker build . -t <user>/<image>:<version>
docker run -e 'WEBROOT=/var/www/html/public' --name polr_webapp -d -p 80:80 <image-id>

```

All you really need for this to work with docker, is a localhost file or control over how dns finds the url of the container. It works really well out of the box in this way instead of localhost, it's really more of a concept than something to use practically. 

Just update the server_name in conf/nginx-site.conf to whatever you want (or have control of DNS over), and after the container image finishes building successfully, run the container for the first time with the environment flag that points the server_name of the URL you're using for the polr_app. 

For your more custom build settings, automation, etc... beyond the scope of this README - See the docs @ https://gitlab.com/ric_harvey/nginx-php-fpm/tree/master



### Polr, Pronto! - With docker-compose

To run the docker container service for Polr; 

Using docker-compose:

```

git clone https://github.com/mackmoe/polr_pronto.git 
cd polr_pronto
docker pull mackmoe/polr_docker
docker-compose -f docker-compose.yml up

```
Note: You may also use 'docker-compose -f docker-compose.yml up -d' if you prefer


In another screen:

```
$ docker ps -a
CONTAINER ID        IMAGE                        COMMAND                  CREATED             STATUS                       PORTS                                   NAMES
58b8e09e4024        mackmoe/polr_docker:latest   "docker-php-entrypoi…"   2 minutes ago       Up 2 minutes                 443/tcp, 0.0.0.0:80->80/tcp, 9000/tcp   polr_docker_webstck_1
d3425ecff48c        mysql/mysql-server:5.7       "/entrypoint.sh mysq…"   2 minutes ago       Up 2 minutes (healthy)       3306/tcp, 33060/tcp                     polr_docker_db_1
--
$ docker inspect polr_docker_db_1 | grep -i hostname
        "HostnamePath": "/var/lib/docker/containers/d3425ecff48c24788da563e8411d3298932d0d358aa9d839206a64aa13411b6c/hostname",
            "Hostname": "d3425ecff48c",  <--- *This is the hostname you'll use for the DeeBee (database hostname on the http://localhost/setup page)
```

#### Default DB Info

database host: (see instructions and cmd output above)
database port: 3306
database username: polr_dbu
database password: secret-password
database name: polr_dbu

More details still in the wworks..... 👌
