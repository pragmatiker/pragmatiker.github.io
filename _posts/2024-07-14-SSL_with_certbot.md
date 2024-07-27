---
layout: post
title: "SSL reverse proxy with Certbot & Nginx"
date: 2024-07-14 12:00:00 -0000
author: "TimDude"
categories:
  - IT Sec 
  - ssh certificates
tags: ssl certbot nginx
---

We will install Cerbot to get certifiactes for our lab from letsencrypt.
Also we will use Nginx to terminate SSL for our application running in docker.

## Install Packages on the host OS
We will need Certbot and Nginx.
```
sudo apt install certbot python3-certbot-nginx nginx
```

## Configure Nginx Vhost
Now lets add a Vhost for our first certifiacte
Certbot is really wonderfull, you only need a minimal Vhost config to start, certbot will add SSL parameters for you.

Add a Serverblock /etc/nginx/sites-available/lab.fogcity.de
```
server {
    listen 80;
    
    server_name lab.fogcity.de;
}
```

Activate the Serverblock
```
ln -sv /etc/nginx/sites-available/lab.fogcity.de /etc/nginx/sites-enabled/
```

## Request a Certificate
Now request the certificate, with certbot
Certbot will automatically edit the Nginx Serverblock with the SSL parameters for you.
```
certbot --nginx -d lab.fogcity.de
```
