---
author: "ZEN"
title: "Своє власне інтернет-радіо"
description: ""
date: 2024-05-17T14:57:52+03:00
lastmod: 2024-05-17T14:57:52+03:00
tags:
  - "mixxx"
  - "icecast"
  - "liquidsoap"
  - "webradio"
  - "selfhost"
  - "nginx"
  - "docker"
categories:
  - "Other"
type: "post"
archives: "2024"
---

В сучасному світі люди більше надають перевагу стрімінговим сервісам, тоді як мені більше до вподоби слухати музику із власної колекції. До того ж це надає впевненості у тому, що завтра чи післязавтра стрімінговий сервіс не видалить треки з моїх плейлистів без мого відома, і що в рекомендаціях не вилізе щось таке, що зіпсує мені настрій. Сьогодні ми розберемось із тим, як створити свій власний стрімінговий сервіс на своєму сервері, який буде стрімити випадкові треки із вашої колекції 24/7, і при цьому за вами залишиться можливість у будь-який момент під'єднатись та стрімити музику під свій настрій.

<!--more-->

Спочатку, розберемось з архітектурою. У вас має бути вже зареєстрований свій домен (чи піддомен, як наприклад, `radio.example.com` далі у статті), згенерований SSL сертифікат, встановлений Nginx та docker з docker-compose, щоб ми змогли налаштувати всі сервіси за наступною схемою:

![Блок схема інтернет-радіо](/images/2024/webradio/webradio-schema.png#center)

Як видно на схемі, у нас є вебсервер Nginx, який виступає в ролі проксі сервера, та ізольовані від зовнішньої мережі docker контейнери - icecast, liquidsoap та webplayer. Контейнер з назвою webplayer відповідатиме за вебінтерфейс та деяку статистику, яку він витягуватиме з icecast, тоді як icecast окрім статистики відповідатиме за аудіо потік, котрий йому буде стрімити liquidsoap. До речі, на схемі liquidsoap має відкритий назовні порт, але якщо ви не хочете буди ді-джеєм, то можна буде потім в docker-compose.yml закрити порт 8888. Наразі ж почнемо з налаштування Nginx, створимо спочатку окремий файл для налаштування віртуального хоста:


```bash
$ sudo touch /etc/nginx/sites-available/radio.conf
$ sudo ln -s /etc/nginx/sites-available/radio.conf /etc/nginx/sites-enabled/
```

І запишемо в нього наступний зміст, тільки не забудьте замінити `radio.example.com` на ім’я свого домену:

```nginx
server {
    listen 80;
    server_name radio.example.com;

    access_log /var/log/nginx/radio.example.com-access.log;
    error_log /var/log/nginx/radio.example.com-error.log;

    proxy_set_header Host $host;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header X-Forwarded-Host $host;
    proxy_set_header X-Forwarded-Server $host;
    proxy_set_header X-Real-IP $remote_addr;

    location /status-json.xsl {
        access_log off;
        log_not_found off;
        proxy_pass http://127.0.0.1:8000/status-json.xsl;
    }

    location /live.mp3.m3u {
        proxy_pass http://127.0.0.1:8080;
    }

    location /live.mp3.xspf {
        proxy_pass http://127.0.0.1:8080;
    }

    location /live.mp3 {
        proxy_pass http://127.0.0.1:8000/live.mp3;
    }

    location /playlist.json {
        access_log off;
        log_not_found off;
        proxy_pass http://127.0.0.1:8080;
    }

    location / {
        return 301 https://$host$request_uri;
    }
}

server {
    listen 443 ssl;
    server_name radio.example.com;

    # ssl certs.
    ssl_certificate     /etc/letsencrypt/live/example.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/example.com/privkey.pem;

    access_log /var/log/nginx/radio.example.com-access.log;
    error_log /var/log/nginx/radio.example.com-error.log;

    proxy_set_header Host $host;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header X-Forwarded-Host $host;
    proxy_set_header X-Forwarded-Server $host;
    proxy_set_header X-Real-IP $remote_addr;

    location /status-json.xsl {
        access_log off;
        log_not_found off;
        proxy_pass http://127.0.0.1:8000/status-json.xsl;
    }

    location /live.mp3.m3u {
        proxy_pass http://127.0.0.1:8080;
    }

    location /live.mp3.xspf {
        proxy_pass http://127.0.0.1:8080;
    }

    location /live.mp3 {
        proxy_pass http://127.0.0.1:8000/live.mp3;
    }

    location /playlist.json {
        access_log off;
        log_not_found off;
        proxy_pass http://127.0.0.1:8080;
    }

    location / {
        proxy_pass http://127.0.0.1:8080;
    }
}
```

Перевіряємо чи не містить конфіг Nginx помилок і перезавантажуємо його:

```bash
$ nginx -t
nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
nginx: configuration file /etc/nginx/nginx.conf test is successful
$ sudo service nginx reload
$
```

Переходимо до налаштування docker-контейнерів. Ви можете обрати будь-яку свою директорію, де у вас буде зберігатись `docker-compose.yml` файл, я ж для прикладу буду використовувати директорію `~/dockerized-apps/radio`. Отже, спочатку створимо цю директорію і поки що пусті, але надалі потрібні, допоміжні файли та директорії.

```bash
~$ mkdir -p ~/dockerized-apps/radio
~$ cd ~/dockerized-apps/radio
~/dockerized-apps/radio$ mkdir -p data/{music,static}
~/dockerized-apps/radio$ touch data/{script.liq,icestats_history.json}
~/dockerized-apps/radio$ touch {.env,docker-compose.yml}
```

В результаті, отримаємо ось таку структуру проєкту:

```bash
~$ tree -a ~/dockerized-apps/radio
~/dockerized-apps/radio
├── data
│   ├── icestats_history.json
│   ├── music
│   ├── script.liq
│   └── static
├── docker-compose.yml
└── .env

4 directories, 4 files
```

Тепер залишилося відредагувати файли `docker-compose.yml`, `.env` та `data/script.liq`. Почнемо з останнього, як я вже раніше згадував, `liquidsoap` у нас займатиметься тим, що буде випадково обирати треки із теки `music` для стріму 24/7, і до нього ж ви, можливо, будете підключатись, коли захочете вийти в прямий етер. Якщо захочете навчити своє радіо новим трюкам, то саме цей файл вам доведеться вивчати та редагувати в майбутньому. Наразі ж відкриємо його в улюбленому редакторі та скопіюємо наступний код:

```bash
#!/usr/bin/liquidsoap
# Allow root
settings.init.allow_root.set(true)

settings.server.telnet.set(true)
settings.server.telnet.port.set(1234)
settings.server.telnet.bind_addr.set("0.0.0.0")
settings.harbor.bind_addrs.set(["0.0.0.0"])
# settings.log.level.set(4)

# Set environments if empty
if getenv("ICECAST_SOURCE_PASSWORD") == "" then
    setenv("ICECAST_SOURCE_PASSWORD", "hackme")
end
if getenv("ICECAST_STREAM_BITRATE") == "" then
    setenv("ICECAST_STREAM_BITRATE", "128")
end
if getenv("STREAM_NAME") == "" then
    setenv("STREAM_NAME", "Radio")
end
if getenv("STREAM_DESC") == "" then
    setenv("STREAM_DESC", "Our selection of music")
end
if getenv("STREAM_MOUNTPOINT") == "" then
    setenv("STREAM_MOUNTPOINT", "live")
end
if getenv("DJ_USERNAME") == "" then
    setenv("DJ_USERNAME", "source")
end
if getenv("DJ_PASSWORD") == "" then
    setenv("DJ_PASSWORD", "hackme")
end
if getenv("DJ_MOUNTPOINT") == "" then
    setenv("DJ_MOUNTPOINT", "/dj-on-air-live")
end

headers = ref([])

def on_connect(new_headers) =
    headers := new_headers
end

def add_dj_metadata(metadata) =
    headers = !headers
    name = headers["Ice-Name"]
    desc = headers["Ice-Description"]

    if (name != "" and desc != "") then
        list.add(("title", "#{name} (#{desc})"), metadata)
    elsif (name != "") then
        list.add(("title", "#{name}"), metadata)
    elsif (desc != "") then
        list.add(("title", "#{desc}"), metadata)
    else
        list.add(("title", "Live DJ Connected"), metadata)
    end
end

live = input.harbor(
    getenv("DJ_MOUNTPOINT"),
    icy=true,
    replay_metadata=true,
    max=2048.,
    port=8888,
    buffer=0.0,
    on_connect=on_connect,
    user=getenv("DJ_USERNAME"),
    password=getenv("DJ_PASSWORD")
)
live = map_metadata(add_dj_metadata, live)

autodj = crossfade(
    duration=3.0,
    smart=true,
    blank.eat(
        start_blank=true,
        max_blank=1.0,
        threshold=-45.0,
        playlist(
            mode="randomize",
            reload=1,
            reload_mode="rounds",
            "/music"
        )
    )
)

radio = mksafe(fallback(track_sensitive=false, [live, autodj]))

# Output
output.icecast(
    %mp3(
        bitrate=int_of_string(getenv("ICECAST_STREAM_BITRATE")),
        id3v2=true
    ),
    name=getenv("STREAM_NAME"),
    description=getenv("STREAM_DESC"),
    url=getenv("STREAM_URL"),
    mount=getenv("STREAM_MOUNTPOINT"),
    password=getenv("ICECAST_SOURCE_PASSWORD"),
    host="icecast",
    genre=getenv("STREAM_GENRE"),
    port=8000,
    encoding="UTF-8",
    radio
)
```

В коді ви могли помітити змінні середовища `ICECAST_SOURCE_PASSWORD` та `DJ_PASSWORD` із дефолтним значенням 'hackme'. Не переживайте, ви їх, а також інші параметри, зараз зміните в `.env` файлі:

```bash
# Web player config
SITE_URL=https://radio.example.com/
SITE_TITLE=Example Radio
SITE_DESCRIPTION=Best Variety Rock Hits + Live Shows
ICECAST_STREAM_URL=https://radio.example.com/live.mp3
ICECAST_STATUS_JSON_URL=https://radio.example.com/status-json.xsl
FEDIVERSE_URL=
GITHUB_URL=
XMPP_URL=

# Liquidsoap config
STREAM_NAME=Radio example.com
STREAM_DESC=Best Variety Rock Hits + Live Shows
STREAM_URL=http://radio.example.com
STREAM_MOUNTPOINT=live.mp3
STREAM_GENRE=Rock
DJ_USERNAME=source
DJ_PASSWORD=ADJPassword
DJ_MOUNTPOINT=dj-on-air-live

# Icecast config
ICECAST_PORT=8000
ICECAST_LOCATION=Ukraine
ICECAST_SOURCE_PASSWORD=ASourcePassword
ICECAST_RELAY_PASSWORD=APasswordForRelaysIGuess
ICECAST_ADMIN_PASSWORD=AnAdminPassword
ICECAST_ADMIN_USERNAME=emanresu
ICECAST_ADMIN_EMAIL=icemaster@localhost
ICECAST_HOSTNAME=localhost
ICECAST_MAX_SOURCES=1
ICECAST_MAX_CLIENTS=1000
ICECAST_CHARSET=UTF-8
```

Обов'язково замініть `radio.example.com` на ім'я хосту, який вказали в конфігурації Nginx, а також згенеруйте та вкажіть свої паролі в змінних `DJ_PASSWORD`, `ICECAST_SOURCE_PASSWORD`, `ICECAST_RELAY_PASSWORD` та `ICECAST_ADMIN_PASSWORD`. І звісно ж вкажіть назву та опис радіо в змінних `SITE_TITLE`, `SITE_DESCRIPTION`, `STREAM_NAME` та `STREAM_DESC`.

Залишився тільки `docker-compose.yml` файл, відкриваємо та записуємо в нього наступне:

```yml
version: '3'

services:
  webplayer:
    container_name: radio-webplayer
    image: "ghcr.io/uanet-exception/radio.social.net.ua:v1.0.0"
    restart: always
    ports:
      - "127.0.0.1:8080:8080"
    environment:
      - SITE_URL
      - SITE_TITLE
      - SITE_DESCRIPTION
      - ICECAST_STREAM_URL
      - ICECAST_STATUS_JSON_URL
      - FEDIVERSE_URL
      - GITHUB_URL
      - XMPP_URL
    depends_on:
      - liquidsoap
    volumes:
      - ./data/icestats_history.json:/opt/radio/icestats_history.json
      # - ./data/static/wallpaper.webp:/opt/radio/app/static/background.webp
      # - ./data/static/favicon.png:/opt/radio/app/static/favicon.png
      # - ./data/static/mic.jpg:/opt/radio/app/static/mic.jpg
      # - ./data/static/css/style.css:/opt/radio/app/static/css/style.css
      # - ./data/static/css/themes/default.css:/opt/radio/app/static/css/themes/default.css

  liquidsoap:
    container_name: radio-liquidsoap
    image: "ghcr.io/savonet/liquidsoap:v2.1.4"
    restart: always
    command: ["/script.liq"]
    ports:
      - "127.0.0.1:${LIQUIDSOAP_PORT:-1234}:1234"
      - "8888:8888"
    environment:
      - ICECAST_STREAM_BITRATE
      - ICECAST_SOURCE_PASSWORD
      - STREAM_NAME
      - STREAM_DESC
      - STREAM_URL
      - STREAM_GENRE
      - STREAM_MOUNTPOINT
      - DJ_USERNAME
      - DJ_PASSWORD
      - DJ_MOUNTPOINT
    depends_on:
      - icecast
    volumes:
      - ./data/music:/music:ro
      - ./data/script.liq:/script.liq:ro

  icecast:
    container_name: radio-icecast
    image: "ghcr.io/uanet-exception/icecast:v2.4.4"
    restart: always
    ports:
      - "127.0.0.1:${ICECAST_PORT:-8000}:8000"
    environment:
      - ICECAST_SOURCE_PASSWORD
      - ICECAST_RELAY_PASSWORD
      - ICECAST_ADMIN_PASSWORD
      - ICECAST_ADMIN_USERNAME
      - ICECAST_ADMIN_EMAIL
      - ICECAST_LOCATION
      - ICECAST_HOSTNAME
      - ICECAST_MAX_CLIENTS
      - ICECAST_MAX_SOURCES
      - ICECAST_CHARSET
```

Поки що нічого редагувати в ньому не треба, але ви могли вже помітити закоментовані рядки в секції для `webplayer`. Це вже вам буде самостійна робота, якщо захочете змінити дефолтний вигляд веб плеєра. Звісно ж, якщо будуть труднощі, то не соромтесь ставити ваші питання у коментарях.

Тепер, коли ми закінчили з конфігурацією, заливаємо будь-яким зручним способом свої улюблені аудіофайли в директорію `data/music`, і підіймаємо контейнери:

```plaintext
~/dockerized-apps/radio$ docker-compose up -d
Creating network "radio_default" with the default driver
Pulling icecast (ghcr.io/uanet-exception/icecast:v2.4.4)...
v2.4.4: Pulling from uanet-exception/icecast
...
6ceee0e1ef2d: Pull complete
Digest: sha256:a36956c457439d1ff46ab9c9d344b15e84086dea003d481a47513c02771439c3
Status: Downloaded newer image for ghcr.io/uanet-exception/icecast:v2.4.4
Pulling liquidsoap (ghcr.io/savonet/liquidsoap:v2.1.0)...
v2.1.0: Pulling from savonet/liquidsoap
...
1f7f75df6d97: Pull complete
Digest: sha256:50ca328d12324d57cce21ab9fef42adc8ea20f5e520da54a03b7d0a1227bfd99
Status: Downloaded newer image for ghcr.io/savonet/liquidsoap:v2.1.0
Pulling webplayer (ghcr.io/uanet-exception/radio.social.net.ua:v1.0.0)...
v1.0.0: Pulling from uanet-exception/radio.social.net.ua
...
06e1ddc3ed0a: Pull complete
Digest: sha256:133e11746b2dc227d86cb9fed6cba251748e6e74b5618384d9d1ce359606e944
Status: Downloaded newer image for ghcr.io/uanet-exception/radio.social.net.ua:v1.0.0
Creating radio-icecast ... done
Creating radio-liquidsoap ... done
Creating radio-webplayer  ... done
```

Перевіряємо, чи всі вони в статусі Up:

```plaintext
~/dockerized-apps/radio$ docker-compose ps
     Name                    Command               State                                 Ports                               
------------------------------------------------------------------------------------------------------------------------------
radio-icecast      /entrypoint.sh /bin/sh -c  ...   Up      127.0.0.1:8000->8000/tcp                                          
radio-liquidsoap   /usr/bin/tini -- /usr/bin/ ...   Up      127.0.0.1:1234->1234/tcp, 0.0.0.0:8888->8888/tcp,:::8888->8888/tcp
radio-webplayer    /usr/bin/supervisord -n -c ...   Up      127.0.0.1:8080->8080/tcp
```

І відкриваємо сторінку плеєра в браузері, тицяємо на кнопку Play, перевіряємо чи є звук. Вже на цьому етапі, з можливою затримкою в 5-10 секунд, на вебсторінці має бути видно поточний трек. Зі зміною треків почне також заповнюватись історія попередніх треків.

Щодо власних стрімів на радіо, то ви можете, як і я, використовувати програму Mixxx, і відповідно в Options->Live Broadcasting налаштувати Server connection подібним чином:

![Mixxx Preferences](/images/2024/webradio/mixxx_preferences.png#center)

Як можна здогадатись, деякі із параметрів на скріншоті ми раніше вказували в `.env` файлі в змінних `DJ_USERNAME`, `DJ_PASSWORD` та `DJ_MOUNTPOINT`. Також, звернутіть увагу на Metadata->Format, як бачите, ви можете додати свій власний префікс до назви пісні, аби підкреслити, що за мікрофоном зараз сидить живий ді-джей.

Про Mixxx детальніше розписувати не буду, це тема для окремої статті, до того ж ви можете користуватись іншою програмою, навіть стрімити голим ffmpeg із термінала. А зараз просто насолоджуйтесь результатом, та діліться із друзями своєю інтернет-радіостанцією (бажано разом із цією інструкцією 😉). Також, не соромтесь додавати своє радіо на сторінці проєкту [radio-browser.info](https://www.radio-browser.info/) (кнопка New station в меню), це зробить його доступним в купі мобільних додатків, які використовують цей сайт для пошуку онлайн стрімів.
