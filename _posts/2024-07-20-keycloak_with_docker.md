---
layout: post
title: "Running Keycloak with docker"
date: 2024-07-20 12:00:00 -0000
author: "TimDude"
categories: ssh_poc docker keycloak
tags: keycloak docker
---

OpenLDAP is already up and running. Time to add Keycloak and a Database.
The external database is not reall necessary for a lab, but it makes Backup a lot easier.
Besides, with docker it is really easy and you will get very close to a production setup.

## Addig Keycloak an Postgresql to our docker-compose.yml
Edit /opt/docker/ssh_poc/docker-compose.yml
~~~
version: '3.8'

services:
  openldap:
    image: osixia/openldap
    container_name: openldap
    ports:
      - "389:389"
    volumes:
      - openldap_config:/etc/ldap/slapd.d
      - openldap_db:/var/lib/ldap
    environment:
      LDAP_ADMIN_PASSWORD: admin
      
  postgres:
    image: postgres:13
    container_name: postgres
    environment:
      POSTGRES_USER: keycloak
      POSTGRES_PASSWORD: keycloak
      POSTGRES_DB: keycloak
    ports:
      - "5432:5432"
    volumes:
      - postgres_data:/var/lib/postgresql/data
      
  keycloak:
    image: quay.io/keycloak/keycloak
    container_name: keycloak
    ports:
      - "8080:8080"
      - "8443:8443"
    volumes:
      - keycloak_data:/opt/keycloak/data
    environment:
      KEYCLOAK_ADMIN: admin
      KEYCLOAK_ADMIN_PASSWORD: admin
      KC_DB: postgres
      KC_DB_URL: jdbc:postgresql://keycloak-postgres:5432/keycloak
      KC_DB_USERNAME: keycloak
      KC_DB_PASSWORD: keycloak
      KC_HOSTNAME: https://auth.fogcity.de
      KC_PROXY_HEADERS: xforwarded
    depends_on:
      - postgres
      - openldap
    command: start-dev

volumes:
  openldap_config:
  openldap_db:
  postgres_data:
  keycloak_data:
~~~

## Start our Lab :)
Run the container
~~~
systemctl restart docker-compose-ssh_poc.service
~~~

## Get a SSL Certificatee for our Nginx Vhost
We are terminating SSL at the Nginx Proxy running on the host OS
Setup a minimal Nginx Vhost to get a certificate.
~~~
export SITE="auth.fogcity.de"
cat <<EOF > /etc/nginx/sites-available/${SITE}
server {
    listen 80;
    server_name ${SITE};
}
EOF

ln -sv /etc/nginx/sites-available/${SITE} /etc/nginx/sites-enabled/
systemctl restart nginx
sudo certbot --nginx -d ${SITE}
~~~


## Add proxy settings to the Vhost
Certbot has alreday added the SSL Parameters, we wiil now add the location statements for Keyclaok
```
server {

    server_name auth.fogcity.de;

    listen 443 ssl; # managed by Certbot
    ssl_certificate /etc/letsencrypt/live/auth.fogcity.de/fullchain.pem; # managed by Certbot
    ssl_certificate_key /etc/letsencrypt/live/auth.fogcity.de/privkey.pem; # managed by Certbot
    include /etc/letsencrypt/options-ssl-nginx.conf; # managed by Certbot
    ssl_dhparam /etc/letsencrypt/ssl-dhparams.pem; # managed by Certbot

    location / {
        proxy_pass http://localhost:8080;  # Forward to Keycloak HTTP port
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }

}
server {
    if ($host = auth.fogcity.de) {
        return 301 https://$host$request_uri;
    } # managed by Certbot

    listen 80;

    server_name auth.fogcity.de;
    return 404; # managed by Certbot
}
```
