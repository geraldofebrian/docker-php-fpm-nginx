# docker-php-fpm-nginx

This repo provides simple configuration for docker with php-fpm and nginx with Virtualhost.

Here are several things you need to consider:
1. You should put **docker-compose.yml** in root directory. These file contains all services you need.
2. Put **dockerfile** file into project's directory (sub-directory from root). This contains the initial configuration for your app to deploy.
3. the directory of all services should be same with directory of nginx service respectively.

See the example of **docker-compose.yml** file below.

```yml

# docker-compose.yml
version: "3.7"

services:
  viewer:
    build:
      context: ./project-directory-1
      dockerfile: './dockerfile'                              # these directory respecting to context directory.
      args:
        port: 9000
        doc_root: '.'
    ports:
      - '9001:9000'
    volumes:
      - './project-directory-1:/path/on/nginx/services'       # e.g /var/www/example-1.test/html
      - './.setup/php/development.ini:/etc/php/php.ini'
    networks:
      - app
    depends_on:
      - nginx
      - mysql

  admin:
    build:
      context: ./project-directory-2
      dockerfile: './dockerfile'                              # these directory respecting to context directory.
      args:
        port: 9000
        doc_root: '.'
    ports:
      - '9001:9000'
    volumes:
      - './project-directory-2:/path/on/nginx/services'       # e.g /var/www/example-2.test/html
      - './.setup/php/development.ini:/etc/php/php.ini'
    networks:
      - app
    depends_on:
      - nginx
      - mysql

  mysql:
    image: mysql:8.0.27
    ports:
      - '3306:3306'
    volumes:
      - 'data-mysql:/var/lib/mysql'
    networks:
      - app
    environment:
      MYSQL_ROOT_PASSWORD: 'your-passwword'
      SERVICE_TAGS: dev
      SERVICE_NAME: mysql

  nginx:
    image: nginx:alpine
    ports:
      - '80:80'
      - '443:443'
    volumes:
      - './project-directory-1:/path/on/nginx/services/project-directory-1'   # e.g /var/www/example-1.test/html
      - './project-directory-2:/path/on/nginx/services/project-directory-2'   # e.g /var/www/example-2.test/html

      - '.setup/nginx/conf/:/etc/nginx/conf.d/'
      - '.setup/nginx/ssl:/etc/nginx/certs'
    networks:
      - app

networks:
  app:
    driver: bridge

volumes:
  data-mysql:
    external: true

```

on the example above, we have 2 app services, 1 database service and main nginx service. mount nginx configuration file into `/etc/nginx/conf.d/` and configure the name of host depend on your virtualhost name.

below is the configuration file named `example-1.test.conf`. 
```conf
server {
    listen 443 ssl;
    server_name example-1.test www.example-1.test;

    root "/var/www/example-1.test/html/public";
    index index.php;

    charset utf-8;

    client_body_buffer_size 10M;
    client_max_body_size 200M;

    # ssl on;
    ssl_certificate /etc/nginx/certs/domain.crt;
    ssl_certificate_key /etc/nginx/certs/domain.key;

    add_header X-Frame-Options "SAMEORIGIN";
    add_header X-Content-Type-Options "nosniff";

    error_page 404 /index.php;

    location = /favicon.ico { access_log off; log_not_found off; }
    location = /robots.txt  { access_log off; log_not_found off; }

    location / {
        try_files $uri $uri/ /index.php?$query_string;
    }

    location ~ \.php$ {
        try_files $uri =404;
        fastcgi_pass viewer:9000;
        fastcgi_index index.php;
        include fastcgi_params;
        fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
        fastcgi_param PATH_INFO $fastcgi_path_info;
    }

    location ~ /\.(?!well-known).* {
        deny all;
    }
}

server {
    listen 80;
    server_name example-1.test www.example-1.test;

    return 302 https://$server_name$request_uri;
}

```
on the `example-1.test.conf` above, do force HTTPS redirect, and pass the **fpm**.  
`fastcgi_pass` is respecting to **your-docker-service-name:port-you-used-in-docker-service**.
