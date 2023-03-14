---
title: Ngnix Basics
author: nirajkharel
date: 2022-07-19 20:55:00 +0800
categories: [DevOps, Ngnix]
tags: [DevOps, Nginx]
---

**Installation**
- `sudo apt install -y nginx`

**Check an status**
- `sudo systemctl status nginx`
- If active and running, navigate to http://localhost.

**Nginx Directory**
- `cd /etc/nginx`
- All the configuration file for nginx are available here.
- All the custom configuration file are available on `/etc/nginx/conf.d`

**Creating custom files**
- We have to unlink the default configuration file under `/etc/nginx/sites-enabled`.
- `unlink default`
- Navigate to `/etc/nginx/conf.d`
- Create a new configuration file. The name should be the same as the site which you are going to create.
- `vim example.com.conf`
- Inside the configuration file,
 - server {
 	listen 80 default_server;
 	index index.html index.htm index.php;
 	server_name example.com;
 	root /var/www/example.com;
 }
 - Test the configuration file is correct or not: `nginx -t`

 **Default location of the Nginx**
 - Whe nginx is installed, the files will be available at `/var/www/nginx`
 - Create a new directory `mkdir /var/www/example.com`
 - Navigate inside the directory and create an `index.html` file.
 - Download a github repository under `/var/www/` https://github.com/techbeast-org/nginx-basics
 - Move the files inside it to example.com

 **Reload Nginx**
 - After the configuration file has been changed, we need to reload an nginx.
 - `systemctl reload nginx`

 **Server multiple pages**
 - Navigate to the configuration directory
 - `sudo vim /etc/nginx/conf.d/example.com.conf`
 - The configuration file will be
  - server {
        listen 80 default_server;
        index index.html index.htm index.php;
        server_name example.com;
        root /var/www/example.com;
     location / {
        try_files $uri $uri/ $uri.html =400;
     }
     location /foss {
        try_files $uri /foss.html;
     }
     }
  
**Create a custom error pages**
- The configuration file will be
- server {
        listen 80 default_server;
        index index.html index.htm index.php;
        server_name example.com;
        root /var/www/example.com;
location / {
        try_files $uri $uri/ $uri.html =400;
}
location /foss {
        try_files $uri /foss.html;
}
error_page 400 404 /400.html;
location = /400.html {
        internal;
}
}
- `sudo nginx -t`
- `sudo systemctl reload nginx`

**Create a 50x error pages**
- The configuration file will be
- server {
        listen 80 default_server;
        index index.html index.htm index.php;
        server_name example.com;
        root /var/www/example.com;
location / {
        try_files $uri $uri/ $uri.html =400;
}
location /foss {
        try_files $uri /foss.html;
}
location /500error{
   fastcgi_pass unix:/this/is/error;
}
error_page 400 404 /400.html;
location = /400.html {
        internal;
}
error_page 500 502 503 504 /50x.html;
location = /50x.html {
        internal;
}
}
- `sudo nginx -t`
- `sudo systemctl reload nginx`


**Security Features in NGINX**
- You can restrict access wherever possible by allowing and blocking IP addresses.
- You can protect your sensitive pages by username and password.
- Use SSL certificate to secure your site by encrypting the client-server traffic. (Here we will use self signed certificate)

**Deny the accessibility to a page**
- location /secure{
      try_files $uri /secure.html;
      deny all;
   }

**Make a page password protected**
- `htpasswd -c /etc/nginx/passwords admin`
- Add the configuration on /etc/nginx/conf.d/com.example.com
- location /secure {
   try_files $uri /secure.html;
   auth_basic "Authentication is required here...";
   auth_basic_user_file /etc/nginx/passwords;
}

**Reverse Proxy**
- Act as load balancer
- Protect the origin servers from attacks by hiding its IP addresses
- Cache content for faster performance
- Provide SSL encryption and decryption to and from the server

**Some Load Balancing options**
- Round Robin: The first request will go to server 1, second request will go to server 2 and third request will go to server 3.
- Least connections: The server having the least connection will have the request.
- IP Hash: It is for establishing the session persistence, It will be choosing the connection based on the server IP address. When the server is dead, next server will accept the connection, otherwise, the same server will receive the connection for same IP address.
- The module `upstream` is responsible for performing load balancer.
