# Ubuntu Web Server Setup

### Switch to the root user

`sudo su`

### Install Node

`curl -sL https://deb.nodesource.com/setup_12.x | sudo -E bash -`

`sudo apt install nodejs`

### Confirm the Installation

`node --version`

### Install git (Ubuntu 18.x installed already)

`sudo apt install git`

### Confirm the Installation

`git --version`

### Clone your project

- Your project should place into `/var/www/example.com/html/`
- Open your project folder and `npm install`

### Startup your application

- `node app.js` (or whatever your file name)
- Check your application with public DNS / public IP

## PM2

PM2 that has a built in load balancer. This will also facilitate in solving the issue where we need to always keep running our app in the CLI.

`sudo npm i pm2 -g`

`pm2 start app.js (or whatever your file name)`

### Setup Firewall

- `sudo ufw enable`
- `sudo ufw status`
- `sudo ufw allow ssh` (Port 22)
- `sudo ufw allow http` (Port 80)
- `sudo ufw allow https` (Port 443)

### Install nginx

`sudo apt install nginx`

### Map a Domain To a Service Running on Your VPS with nginx

- `sudo mkdir /etc/nginx/sites-available`
- `sudo mkdir /etc/nginx/sites-enabled`
- `cd /etc/nginx/sites-available`
- `touch example.com`

#### Main Domain

```
server {
    listen 80;
    listen [::]:80;
    server_name example.com www.example.com;
    return 301 https://$server_name$request_uri;
}

server {
    listen 443 ssl http2 default_server;
    listen [::]:443 ssl http2 default_server;
    server_name example.com www.example.com;
    location / {
        root /var/www/example.com/html;
        index index.html index.htm index.nginx-debian.html
        proxy_pass http://localhost:3000;
    }
    ssl_certificate /etc/letsencrypt/live/example.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/example.com/privkey.pem;
    ssl_session_timeout 1h;
    ssl_prefer_server_ciphers on;
    ssl_session_cache shared:SSL:5m;
    ssl_protocols TLSv1 TLSv1.1 TLSv1.2;
    add_header Strict-Transport-Security “max-age=15768000” always;
    ssl_ciphers EECDH+CHACHA20:EECDH+AES128:RSA+AES128:EECDH+AES256:RSA+AES256:EECDH+3DES:RSA+3DES:!MD5;
}
```

#### Sub-Domain

```

server {
    listen 80;
    listen [::]:80;
    server_name sample.example.com www.sample.example.com;
    return 301 https://$server_name$request_uri;
}

server {
    listen 443 ssl;
    listen [::]:443 ssl;

    server_name sample.example.com www.sample.example.com;

    location / {
        root /var/www/sample.example.com/html/;
#       index index.html index.htm index.nginx-debian.html (Do not need when you are running Node.js)
        proxy_pass http://localhost:3000;
#       proxy_http_version 1.1;  (Do not need when you are running Node.js)
#       proxy_set_header Upgrade $http_upgrade;  (Do not need when you are running Node.js)
#       proxy_set_header Connection 'upgrade';  (Do not need when you are running Node.js)
#       proxy_set_header Host $host;  (Do not need when you are running Node.js)
        proxy_cache_bypass $http_upgrade;
    }

    try_files $uri $uri / =404;

    ssl_certificate /etc/letsencrypt/live/example.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/example.com/privkey.pem;
    ssl_session_timeout 1h;
    ssl_prefer_server_ciphers on;
    ssl_session_cache shared:SSL:5m;
    ssl_protocols TLSv1 TLSv1.1 TLSv1.2;
    add_header Strict-Transport-Security “max-age=15768000” always;
    ssl_ciphers EECDH+CHACHA20:EECDH+AES128:RSA+AES128:EECDH+AES256:RSA+AES256:EECDH+3DES:RSA+3DES:!MD5;
}

access_log /var/log/nginx/sample.example.com.access.log;
error_log /var/log/nginx/sample.example.com.error.log;

```

### Creating symbolic links from these files to the sites-enabled directory. The Nginx reads this from during startup.

`sudo ln -s /etc/nginx/sites-available/example.com /etc/nginx/sites-enabled/`

```
# Check NGINX config
sudo nginx -t

# Restart NGINX
sudo service nginx restart
```

### Update DNS in your DNS panel

## Installing SSL/TLS Certificates on AWS EC2 with Ubuntu and Nginx Configuration

### Security Group

- HTTP: TCP 80 0.0.0.0/0, ::/0
- HTTPS: TCP 443 0.0.0.0/0, ::/0
- SSH: TCP 22 0.0.0.0/0, ::/0

### Setting up configuration in Route 53

Now, you need to create two basic records which it points your domain to the EC2. Let say your IP is 198.51.100.234 , you need to create an “A” record : A 300 198.51.100.234 and also create the second one with the AAAA 3600 198.51.100.234 , and set up the name with www. Both records can ensure your server to redirect from example.com to www.example.com.

So now you log in to Ubuntu, and you should be able to run those commands below to install Certificates.

````
sudo wget http://nginx.org/keys/nginx_signing.key
sudo apt-key add nginx_signing.key
cd /etc/apt
echo -e "deb http://nginx.org/packages/ubuntu xenial nginx \ndeb-src http://nginx.org/packages/ubuntu xenial nginx" | sudo tee -a sources.list
````
The Nginx will be started, or you can check with your IP address to confirm.

````
sudo apt-get update 
sudo apt-get install software-properties-common
sudo add-apt-repository ppa:certbot/certbot
sudo apt-get update
sudo apt-get install python-certbot-nginx
````

Don’t forget to update the domain name in Nginx, and go to the path

````
sudo nano /etc/nginx/conf.d/default.conf
````

Replace server_name from localhost to yourdomainname.com and the www domain as well www.yourdomainname.com

````
sudo certbot --nginx -d yourdomainname.com -d www.yourdomainname.com
````

If there is no error, you should be able to find all your SSL’s file by navigating to this directory and enter 2 for the redirect.

````
cd /etc/letsencrypts/live/yourdomainname.com
ls
````

The list of files will be:

````
cert.pem
chain.pem
fullchain.pem
privkey.pem
````

Next return back to Nginx config directory:

````
cd /etc/nginx/conf.d
````

Basically, there is a default Nginx file, but you don’t want to overwrite it. Just update the name of that file as a backup file:

````
sudo mv default.conf default.conf.bak
sudo touch serverUbuntu.conf
sudo nano serverUbuntu.conf
````
Right now the main part to take care of is to replace youdomainname.com with your actual domain

````
server {
    listen 80;
    listen [::]:80;
    server_name yourdomainname.com www.yourdomainname.com;
    return 301 https://$server_name$request_uri;
}
server {
    listen 443 ssl http2 default_server;
    listen [::]:443 ssl http2 default_server;
    server_name yourdomainname.com www.yourdomainname.com;
    location / {
        proxy_pass http://localhost:3000;
    }
    ssl_certificate /etc/letsencrypt/live/yourdomainname.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/yourdomainname.com/privkey.pem;
    ssl_session_timeout 1h;
    ssl_prefer_server_ciphers on;
    ssl_session_cache shared:SSL:5m;
    ssl_protocols TLSv1 TLSv1.1 TLSv1.2;
    add_header Strict-Transport-Security “max-age=15768000” always;
    ssl_ciphers EECDH+CHACHA20:EECDH+AES128:RSA+AES128:EECDH+AES256:RSA+AES256:EECDH+3DES:RSA+3DES:!MD5;
}
````

````
sudo nginx -s reload
````
Now your server will be shown the secure connect

### Get Wildcard SSL

````
certbot certonly --manual --preferred-challenges dns --server https://acme-v02.api.letsencrypt.org/directory --manual-public-ip-logging-ok -d '*.<your.domain>' -d <your.domain>
````

## Automatically Renew SSL Certificates (Optional)

Set this task to run automatically once per month using a cron-job:
````
sudo crontab -e
````
Add the following lines to the end of the crontab file:
````
0 0 1 * * /opt/letsencrypt/letsencrypt-auto renew
0 0 1 * * cd /opt/letsencrypt && git pull
````