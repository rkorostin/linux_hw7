# Операционные системы и виртуализация (Linux)
## Запуск веб-приложения из контейнеров
### Установить в виртуальную машину или VDS Docker, настроить набор контейнеров через docker compose по инструкции по ссылке: https://www.digitalocean.com/community/tutorials/how-to-install-wordpress-with-docker-compose-ru. Часть с настройкой certbot и HTTPS опустить, если у вас нет настоящего домена и белого IP. 
1. Установка docker https://docs.docker.com/engine/install/ubuntu/
 - ~$ sudo docker -v
 ```
Docker version 23.0.4, build f480fb1
```
 - ~$ sudo apt install docker-compose
 - ~$ sudo docker-compose -v
```
docker-compose version 1.29.2, build unknown
```
2. Установка WordPress с помощью Docker Compose https://www.digitalocean.com/community/tutorials/how-to-install-wordpress-with-docker-compose-ru
 - ~$ mkdir wordpress && cd wordpress
 - ~/wordpress$ mkdir nginx-conf
 - ~/wordpress$ vim nginx-conf/nginx.conf
 ```
 server {
        listen 80;
        listen [::]:80;

        server_name example.com www.example.com;

        index index.php index.html index.htm;

        root /var/www/html;

        location ~ /.well-known/acme-challenge {
                allow all;
                root /var/www/html;
        }

        location / {
                try_files $uri $uri/ /index.php$is_args$args;
        }

        location ~ \.php$ {
                try_files $uri =404;
                fastcgi_split_path_info ^(.+\.php)(/.+)$;
                fastcgi_pass wordpress:9000;
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
        location = /robots.txt {
                log_not_found off; access_log off; allow all;
        }
        location ~* \.(css|gif|ico|jpeg|jpg|js|png)$ {
                expires max;
                log_not_found off;
        }
}
```
 - ~/wordpress$ vim .env
```
MYSQL_ROOT_PASSWORD=your_root_password
MYSQL_USER=your_wordpress_database_user
MYSQL_PASSWORD=your_wordpress_database_password
```
 - ~/wordpress$ vim docker-compose.yml
```
---
version: '3'

services:
  db:
    image: mysql:8.0
    container_name: db
    restart: unless-stopped
    env_file: .env
    environment:
      - MYSQL_DATABASE=wordpress
    volumes:
      - dbdata:/var/lib/mysql
    command: '--default-authentication-plugin=mysql_native_password'
    networks:
      - app-network

  wordpress:
    depends_on:
      - db
    image: wordpress:5.1.1-fpm-alpine
    container_name: wordpress
    restart: unless-stopped
    env_file: .env
    environment:
      - WORDPRESS_DB_HOST=db:3306
      - WORDPRESS_DB_USER=$MYSQL_USER
      - WORDPRESS_DB_PASSWORD=$MYSQL_PASSWORD
      - WORDPRESS_DB_NAME=wordpress
    volumes:
      - wordpress:/var/www/html
    networks:
      - app-network

  webserver:
    depends_on:
      - wordpress
    image: nginx:1.15.12-alpine
    container_name: webserver
    restart: unless-stopped
    ports:
      - "80:80"
    volumes:
      - wordpress:/var/www/html
      - ./nginx-conf:/etc/nginx/conf.d
    networks:
      - app-network

volumes:
  wordpress:
  dbdata:

networks:
  app-network:
    driver: bridge
```
Проверка yaml
 - ~/wordpress$ yamllint docker-compose.yml

Проверка, что docker чист, без образов, без контейнеров
 - ~/wordpress$ sudo docker images
```
REPOSITORY   TAG       IMAGE ID   CREATED   SIZE
```
 - ~/wordpress$ sudo docker ps -a
```
CONTAINER ID   IMAGE     COMMAND   CREATED   STATUS    PORTS     NAMES
```
 - ~/wordpress$ sudo docker-compose up -d
 - ~/wordpress$ sudo docker-compose ps -a
 ```
  Name                 Command               State                Ports
-------------------------------------------------------------------------------------
db          docker-entrypoint.sh --def ...   Up      3306/tcp, 33060/tcp
webserver   nginx -g daemon off;             Up      0.0.0.0:80->80/tcp,:::80->80/tcp
wordpress   docker-entrypoint.sh php-fpm     Up      9000/tcp
```
3. Захожу на **localhost**

![wp-admin](https://github.com/rkorostin/Images/blob/main/linux_hw7_1.png)
### Запустить два контейнера, связанные одной сетью (используя документацию). Первый контейнер БД (например, образ mariadb:10.8), второй контейнер — phpmyadmin. Получить доступ к БД в первом контейнере через второй контейнер (веб-интерфейс phpmyadmin).
1. Создать сеть для контейнеров:
 - ~/wordpress$ docker network create mynetwork
 - ~/wordpress$ sudo docker network ls
```
NETWORK ID     NAME                    DRIVER    SCOPE
32dae66977ae   bridge                  bridge    local
88d51e2220f5   host                    host      local
c974ba64cdaa   mynetwork               bridge    local
838f3c0925f4   none                    null      local
bb66ebcf34a1   wordpress_app-network   bridge    local
```
2. Создать контейнер с БД с привязкой к сети **mynetwork**:
 - sudo docker run --name mariadb -e MYSQL_ROOT_PASSWORD=password111 --network mynetwork -d mariadb:10.8
3. Создать контейнер с phpmyadmin с привязкой к сети **mynetwork**:
 - ~/wordpress$ sudo docker run --name phpmyadmin -d --network mynetwork -e PMA_HOST=mariadb -p 8080:80 phpmyadmin
4. Зайти на **localhost:8080/**
![phpmyadmin](https://github.com/rkorostin/Images/blob/main/linux_hw7_2.png)

## Итого:
- :~/wordpress$ sudo docker ps -a #Какие контейнеры есть и активны
```
CONTAINER ID   IMAGE                        COMMAND                  CREATED          STATUS          PORTS                                   NAMES
d600f6baad98   phpmyadmin                   "/docker-entrypoint.…"   4 minutes ago    Up 4 minutes    0.0.0.0:8080->80/tcp, :::8080->80/tcp   phpmyadmin
862ad2283924   mariadb:10.8                 "docker-entrypoint.s…"   6 minutes ago    Up 6 minutes    3306/tcp                                mariadb
19299cba3426   nginx:1.15.12-alpine         "nginx -g 'daemon of…"   23 minutes ago   Up 17 minutes   0.0.0.0:80->80/tcp, :::80->80/tcp       webserver
d5de0526ba74   wordpress:5.1.1-fpm-alpine   "docker-entrypoint.s…"   23 minutes ago   Up 17 minutes   9000/tcp                                wordpress
c80653e29980   mysql:8.0                    "docker-entrypoint.s…"   23 minutes ago   Up 17 minutes   3306/tcp, 33060/tcp                     db
```
 - ~/wordpress$ sudo docker-compose stop # тушим контейнеры из первого задания, т.е. выключаем wp-admin
```
Stopping webserver ... done
Stopping wordpress ... done
Stopping db        ... done
```
 - :~/wordpress$ sudo docker ps # Смотрим какие контейнеры активны
```
CONTAINER ID   IMAGE          COMMAND                  CREATED         STATUS         PORTS                                   NAMES
d600f6baad98   phpmyadmin     "/docker-entrypoint.…"   5 minutes ago   Up 5 minutes   0.0.0.0:8080->80/tcp, :::8080->80/tcp   phpmyadmin
862ad2283924   mariadb:10.8   "docker-entrypoint.s…"   8 minutes ago   Up 8 minutes   3306/tcp                                mariadb
```
Убеждаемся, что по сети **mynetwork** работают контейнеры **phpmyadmin*** и **mariadb**
 - :~/wordpress$ sudo docker network inspect mynetwork
```
[
    {
        "Name": "mynetwork",
        "Id": "c974ba64cdaa421a7eb2e5cbbf0aa6ca415aa19e122256f863dc81a731710beb",
        "Created": "2023-04-18T21:32:27.92422894+03:00",
        "Scope": "local",
        "Driver": "bridge",
        "EnableIPv6": false,
        "IPAM": {
            "Driver": "default",
            "Options": {},
            "Config": [
                {
                    "Subnet": "172.19.0.0/16",
                    "Gateway": "172.19.0.1"
                }
            ]
        },
        "Internal": false,
        "Attachable": false,
        "Ingress": false,
        "ConfigFrom": {
            "Network": ""
        },
        "ConfigOnly": false,
        "Containers": {
            "862ad2283924a594fc6c26a00123b31984acbb66f9436a62edfc58508d891d81": {
                "Name": "mariadb",
                "EndpointID": "1001b9baab335fcdebf285346fd4b84699f51e05ad877c0ac8e8b2eb3ed2c722",
                "MacAddress": "02:42:ac:13:00:02",
                "IPv4Address": "172.19.0.2/16",
                "IPv6Address": ""
            },
            "d600f6baad983058496150e4e6e96f7527e74c41e4d06c35d22abe94cf784ac0": {
                "Name": "phpmyadmin",
                "EndpointID": "4776456c3e73790b0de72a01f4b0a209677a8ec5bfc6575358bd8d2bdd2971d6",
                "MacAddress": "02:42:ac:13:00:03",
                "IPv4Address": "172.19.0.3/16",
                "IPv6Address": ""
            }
        },
        "Options": {},
        "Labels": {}
    }
]
```



