---
author: "ZEN"
title: "Єдина точка авторизації для домашнього NAS"
description: ""
date: 2024-11-15T21:00:56+02:00
lastmod: 2024-11-15T21:00:56+02:00
tags:
  - "homelab"
  - "nas"
  - "nginx"
  - "authelia"
  - "openssl"
  - "selfhost"
  - "SSO"
  - "Authentication"
  - "Authorization"
  - "ForwardAuth"
  - "ProxyAuth"
  - "docker"
categories:
  - "Administration"
draft: false
type: "post"
archives: "2024"
images:
  - images/default-cover.png
---

Загорівся бажанням зробити зі старого системника домашній NAS-сервер і сховати всі сервіси за єдиною точкою авторизації. В даній статті ми розглянемо інтеграцію вебсервера Nginx та Authelia із використанням методу авторизації ForwardAuth(ProxyAuth).

<!--more-->

За основу свого NAS я обрав Debian GNU/Linux 12, де, окрім Nginx, всі сервіси будуть запускатись в Docker-контейнері. Тому, за винятком деяких команд, інструкція цілком підійде й для інших Linux-систем. У даній статті буде охоплена лише авторизація по HTTP, яка буде використовувати модуль Nginx auth_request. Тобто Nginx виступатиме в ролі проксі для всіх сервісів, звертаючись на кожному запиті до Authelia для перевірки сесії й за потреби буде перенаправляти користувача на сторінку авторизації. Вебсервер матиме свій самопідписаний сертифікат, а всі сервіси планується запускати в піддиректоріях.

Наступна схема демонструє як все має за задумом працювати:

![Nginx and Authelia Flowchart](/images/2024/sso-nginx-authelia/Nginx_Authelia_Flowchart.png#center)

Отож, почнемо зі встановлення необхідного програмного забезпечення:

```plaintext
~$ sudo apt update
~$ sudo apt install nginx-full docker.io docker-compose openssl
```

І згенеруємо SSL-сертифікат наступною командою:

```plaintext
~$ sudo openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
  -keyout /etc/ssl/private/nginx-selfsigned.key \
  -out /etc/ssl/certs/nginx-selfsigned.crt
```

Команда поставить нам кілька запитань, де важливо для Common Name вказати IP-адресу вашого майбутнього NAS. У моєму випадку це 192.168.1.3:

```plaintext
Country Name (2 letter code) [AU]: UA
State or Province Name (full name) [Some-State]: Odeska Oblast
Locality Name (eg, city) []: Odesa
Organization Name (eg, company) [Internet Widgits Pty Ltd]: Homelab
Organizational Unit Name (eg, section) []: NAS
Common Name (e.g. server FQDN or YOUR name) []: 192.168.1.3
Email Address []: admin@localhost
```

Далі потрібно створити Diffie-Hellman ключ, який буде використовуватися для узгодження ідеальної прямої секретності з клієнтами:

```plaintext
~$ sudo openssl dhparam -out /etc/nginx/dhparam.pem 4096
```

Сертифікат готовий, переходимо до налаштування Nginx.

Щоб ми могли перевикористовувати налаштування SSL директивою `include` замість того, щоб копіювати десятки разів один й той самий текст для різних віртуальних хостів, винесемо ці налаштування в окремі сніпети.

Спочатку створюємо сніпет, де будуть прописані шляхи до сертифікатів:

```plaintext
~$ sudo vim /etc/nginx/snippets/self-signed.conf
```

І записуємо в нього наступний текст:

```plaintext
ssl_certificate /etc/ssl/certs/nginx-selfsigned.crt;
ssl_certificate_key /etc/ssl/private/nginx-selfsigned.key;
```

Далі створюємо окремий сніпет для параметрів SSL, щоб налаштувати Nginx на використання сильного набору шифрів SSL і ввімкнути деякі додаткові функції, які допоможуть забезпечити безпеку вебсервера:

```plaintext
~$ sudo vim /etc/nginx/snippets/ssl-params.conf
```

І відповідно записуємо в нього наступний текст:

```nginx
ssl_protocols TLSv1.3;
ssl_prefer_server_ciphers on;
ssl_dhparam /etc/nginx/dhparam.pem;
ssl_ciphers EECDH+AESGCM:EDH+AESGCM;
ssl_ecdh_curve secp384r1;
ssl_session_timeout  10m;
ssl_session_cache shared:SSL:10m;
ssl_session_tickets off;
ssl_stapling on;
ssl_stapling_verify on;
resolver 8.8.8.8 8.8.4.4 valid=300s;
resolver_timeout 5s;
# Disable strict transport security for now. You can uncomment the following
# line if you understand the implications.
#add_header Strict-Transport-Security "max-age=63072000; includeSubDomains; preload";
add_header X-Frame-Options DENY;
add_header X-Content-Type-Options nosniff;
add_header X-XSS-Protection "1; mode=block";
```

Щоб перевірити наші налаштування SSL, додамо https до віртуального хосту який є в Nginx за замовченням.

Відкриваємо наступний файл:

```plaintext
~$ sudo vim /etc/nginx/sites-enabled/default
```

Замінюємо весь його вміст на наступний:

```nginx
server {
  listen 80 default_server;
  listen [::]:80 default_server;

  server_name _;
  return 301 https://$server_name$request_uri;
}

server {
  listen 443 ssl;
  listen [::]:443 ssl;
  include snippets/self-signed.conf;
  include snippets/ssl-params.conf;

  root /var/www/html;
  index index.html index.htm index.nginx-debian.html;

  server_name _;

  location / {
    try_files $uri $uri/ =404;
  }
}
```

Перезавантажуємо Nginx:
```plaintext
~$ sudo service nginx restart
```

І відкриваємо в браузері http://192.168.1.3. Браузер має перенаправити вас на https://192.168.1.3 і попередити, що перед вами сайт із самопідписаним SSL-сертифікатом, тому погоджуємося на ризик, щоб продовжити. Якщо після цього кроку у вас відкрилася стандартна сторінка Nginx, то ви все зробили правильно.

Переходимо до налаштування Authelia. Ви можете обрати будь-яку свою директорію, де у вас буде зберігатись docker-compose.yml файл, я ж для прикладу буду використовувати директорію `~/dockerized-apps/authelia`.

Отже, спочатку створимо цю директорію і поки що пусті, але надалі потрібні, допоміжні файли та директорії:

```plaintext
~$ mkdir -p ~/dockerized-apps/authelia
~$ cd ~/dockerized-apps/authelia
~/dockerized-apps/authelia$ mkdir config
~/dockerized-apps/authelia$ touch docker-compose.yml config/{configuration,users_database}.yml
```

В результаті, отримаємо ось таку структуру проєкту:

```plaintext
~/dockerized-apps/authelia$ tree -a .
.
├── config
│   ├── configuration.yml
│   └── users_database.yml
└── docker-compose.yml

2 directories, 3 files
```

І тепер наповнюємо змістом yml файли, спочатку `docker-compose.yml` без будь-яких змін:

```yml
version: '3.3'

services:
  authelia:
    image: ghcr.io/authelia/authelia:4.38.8
    container_name: authelia_server
    volumes:
      - ./config:/config
    ports:
      - 127.0.0.1:9091:9091
    restart: unless-stopped
    environment:
      - TZ=Europe/Kyiv

  redis:
    image: redis:alpine
    container_name: authelia_redis
    volumes:
      - ./redis:/data
    restart: unless-stopped
    environment:
      - TZ=Europe/Kyiv
```

Далі `config/configuration.yml`:

```yml
server:
  address: 'tcp://0.0.0.0:9091/authelia'
  endpoints:
    authz:
      auth-request:
        implementation: 'AuthRequest'

log:
  level: info

totp:
  issuer: '192.168.1.3'

identity_validation:
  reset_password:
    jwt_secret: <your-jwt-key>

authentication_backend:
  file:
    path: /config/users_database.yml

access_control:
  default_policy: deny
  rules:
    - domain:
        - "192.168.1.3"
      policy: one_factor

session:
  secret: <your-session-key>
  cookies:
    - name: authelia_session
      domain: '192.168.1.3'
      authelia_url: 'https://192.168.1.3/authelia'
      expiration: 12h
      inactivity: 45m

  redis:
    host: authelia_redis
    port: 6379

regulation:
  max_retries: 3
  find_time: 5m
  ban_time: 15m

storage:
  encryption_key: <your-encryption-key>
  local:
    path: /config/db.sqlite3

notifier:
  filesystem:
    filename: /config/notification.txt
```

Обов'язково замініть в файлі `<your-jwt-key>`, `<your-session-key>` та `<your-encryption-key>` на унікальні значення, які можна отримати наступною командою:

```bash
head /dev/urandom | tr -dc A-Za-z0-9 | head -c64
```

І ще залишився останній yml файл, `config/users_database.yml`:

```yml
###############################################################
#                         Users Database                      #
###############################################################
# This file can be used if you do not have an LDAP set up.
users:
  <username>:
    displayname: "<Your Name>"
    # Generate with docker run authelia/authelia:4.38.8 authelia crypto hash generate argon2 --password="<your-password>"
    password: "<hashed-password>"
    email: <your-email>
    groups:
      - admins
```

Обов'язково замініть у файлі `<username>`, `<Your Name>`, `<your-email>` та `<hashed-password>` на свої значення. Для `<hashed-password>` скористуйтесь наступною командою, щоб отримати хеш:

```plaintext
~$ docker run authelia/authelia:4.38.8 authelia crypto hash generate argon2 --password="<your-password>"
```

Тепер запускаємо контейнери, і перевіряємо їх статус, всі мають бути зі статусом Up:

```plaintext
~/dockerized-apps/authelia$ docker-compose up -d
~/dockerized-apps/authelia$ docker-compose ps
     Name                    Command                  State                Ports          
------------------------------------------------------------------------------------------
authelia_redis    docker-entrypoint.sh redis ...   Up             6379/tcp                
authelia_server   /app/entrypoint.sh               Up (healthy)   127.0.0.1:9091->9091/tcp
```

Повертаємось знову до налаштування Nginx, настав час зробити інтеграцію Nginx з Authelia. Створюємо ще один сніпет `/etc/nginx/authelia_auth.conf`, де один раз вкажемо всі параметри проксі-запиту до Authelia:

```nginx
# Basic Authelia Config
# Send a subsequent request to Authelia to verify if the user is authenticated
# and has the right permissions to access the resource.
auth_request /internal/authelia/authz;
# Set the $(target_url) variable based on the request. It will be used to build the portal
# URL with the correct redirection parameter.
auth_request_set $target_url $scheme://$http_host$request_uri;
# Set the X-Forwarded-User and X-Forwarded-Groups with the headers
# returned by Authelia for the backends which can consume them.
# This is not safe, as the backend must make sure that they come from the
# proxy. In the future, it's gonna be safe to just use OAuth.
auth_request_set $user $upstream_http_remote_user;
auth_request_set $groups $upstream_http_remote_groups;
auth_request_set $name $upstream_http_remote_name;
auth_request_set $email $upstream_http_remote_email;
proxy_set_header Remote-User $user;
proxy_set_header Remote-Groups $groups;
proxy_set_header Remote-Name $name;
proxy_set_header Remote-Email $email;
# If Authelia returns 401, then nginx redirects the user to the login portal.
# If it returns 200, then the request pass through to the backend.
# For other type of errors, nginx will handle them as usual.
error_page 401 =302 https://$http_host/authelia/login/?rd=$target_url;

client_body_buffer_size 128k;
client_max_body_size 0;

proxy_next_upstream error timeout invalid_header http_500 http_502 http_503;

send_timeout 5m;
proxy_read_timeout 360;
proxy_send_timeout 360;
proxy_connect_timeout 360;

proxy_set_header Host $host;
proxy_set_header Upgrade $http_upgrade;
proxy_set_header Connection upgrade;
proxy_set_header Accept-Encoding gzip;
proxy_set_header X-Real-IP $remote_addr;
proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
proxy_set_header X-Forwarded-Proto $scheme;
proxy_set_header X-Forwarded-Host $http_host;
proxy_set_header X-Forwarded-Uri $request_uri;
proxy_set_header X-Forwarded-Ssl on;
proxy_redirect http:// $scheme://;
proxy_http_version 1.1;
proxy_set_header Connection "";
proxy_cache_bypass $cookie_session;
proxy_no_cache $cookie_session;
proxy_buffers 64 512k;
proxy_buffer_size 32k;
proxy_headers_hash_max_size 1024;
proxy_headers_hash_bucket_size 128;
```

І тепер відкриваємо ще раз `/etc/nginx/sites-enabled/default`, і приводимо його вміст до наступного вигляду:

{{< highlight nginx "linenos=table,hl_lines=1-5 26-67 70,linenostart=1" >}}
# Upgrade WebSocket if requested, otherwise use keepalive
map $http_upgrade $connection_upgrade {
    default upgrade;
    ''      '';
}

server {
  listen 80 default_server;
  listen [::]:80 default_server;

  server_name _;
  return 301 https://$server_name$request_uri;
}

server {
  listen 443 ssl;
  listen [::]:443 ssl;
  include snippets/self-signed.conf;
  include snippets/ssl-params.conf;

  root /var/www/html;
  index index.html index.htm index.nginx-debian.html;

  server_name _;

  location /authelia/ {
    proxy_pass http://127.0.0.1:9091;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header X-Forwarded-Proto $scheme;
    proxy_set_header X-Forwarded-Server $host;
    proxy_set_header X-Forwarded-Host $host;
    proxy_set_header Host $host;
  }

  location /internal/authelia/authz {
    internal;
    proxy_pass http://127.0.0.1:9091/authelia/api/authz/auth-request;

    ## Headers
    ## The headers starting with X-* are required.
    proxy_set_header X-Original-Method $request_method;
    proxy_set_header X-Original-URL $scheme://$http_host$request_uri;
    proxy_set_header X-Forwarded-For $remote_addr;
    proxy_set_header Content-Length "";
    proxy_set_header Connection "";

    ## Basic Proxy Configuration
    proxy_pass_request_body off;

    # Timeout if the real server is dead
    proxy_next_upstream error timeout invalid_header http_500 http_502 http_503;

    proxy_redirect http:// $scheme://;
    proxy_http_version 1.1;
    proxy_cache_bypass $cookie_session;
    proxy_no_cache $cookie_session;
    proxy_buffers 4 32k;
    client_body_buffer_size 0;

    ## Advanced Proxy Configuration
    send_timeout 5m;
    proxy_read_timeout 240;
    proxy_send_timeout 240;
    proxy_connect_timeout 240;
  }

  location / {
    try_files $uri $uri/ =404;
    include /etc/nginx/authelia_auth.conf;
  }
}
{{< / highlight >}}

Перезавантажуємо налаштування Nginx:

```plaintext
~$ sudo service nginx reload
```

І знову відкриваємо https://192.168.1.3 у браузері, де вас має зустріти форма логіну. Якщо нічого не наплутали, то після вводу логіну та паролю вас має зустріти вже знайома сторінка Nginx.

На цьому все, більше сервісів для домашнього NAS ви зможете самостійно знайти на сторінці [awesome-selfhosted](https://github.com/awesome-selfhosted/awesome-selfhosted?tab=readme-ov-file#awesome-selfhosted), і деякі [*.subfolder.conf.sample](https://github.com/linuxserver/reverse-proxy-confs/) файли до них.
