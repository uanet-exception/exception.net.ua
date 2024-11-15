---
author: "ZEN"
title: "Пряма трансляція з авторизацією за допомогою Nginx-RTMP"
# description: ""
date: 2021-09-16T15:19:52+03:00
# lastmod: 2021-09-16T15:19:52+03:00
tags:
  - "ffmpeg"
  - "ffplay"
  - "nginx"
  - "rtmp"
  - "linux"
categories:
  - "Other"
  - "Administration"
draft: false
type: "post"
archives: "2021"
---

Вирішив погратися з трансляцією відео через власний сервер. Серед наявних рішень я спочатку розглядав можливість встановлення додаткового програмного забезпечення на кшталт [Owncast](https://owncast.online/) та [rtsp-simple-server](https://hub.docker.com/r/aler9/rtsp-simple-server), але в результаті вирішив просто довстановити модуль rtmp до nginx. До того ж, мені не потрібен вебінтерфейс для переглядання трансляції.

<!--more-->

Отже, передусім встановлюємо пакети nginx-full та модуль rtmp:
{{< highlight html >}}
root@vps-00001:~# apt-get install nginx-full libnginx-mod-rtmp
...
{{< / highlight >}}

Наступним кроком відкриваємо у своєму улюбленому текстовому редакторі файл `/etc/nginx/nginx.conf` та додаємо туди конфігурацію двох примітивних віртуальних веб серверів для перевірки авторизації, а також додаємо налаштування RTMP серверу:

{{< highlight nginx "linenos=table,hl_lines=4-22 25-41,linenostart=72" >}}
 	include /etc/nginx/conf.d/*.conf;
 	include /etc/nginx/sites-enabled/*;

	server {
		listen localhost:9999;
		location /rtmp-auth-on-publish {
			if ($arg_access_token = "publishonlySECRETpassword") {
				return 201;
			}
			return 404;
		}
	}

	server {
		listen localhost:9090;
		location /rtmp-auth-on-play {
			if ($arg_access_token = "watchonlySECRETpassword") {
				return 201;
			}
			return 404;
		}
	}
}

 # RTMP configuration
 rtmp {
     server {
         listen 1935;
         ping 30s;
         notify_method get;
         chunk_size 4000;

         application stream {
             on_publish http://localhost:9999/rtmp-auth-on-publish;
             on_play http://localhost:9090/rtmp-auth-on-play;
             live on;
             meta copy;
             record off;
         }
     }
}

 #mail {

{{< / highlight >}}

Зверніть увагу на рядки 78 та 88, замість **`publishonlySECRETpassword`** та **`watchonlySECRETpassword`** ви повинні вказати свої паролі. Рекомендую згенерувати нові з допомогою [random.org/passwords](https://www.random.org/passwords/?num=8&len=18&format=html&rnd=new).

Далі перевіряємо, що файл налаштувань nginx не містить помилок:

{{< highlight html >}}
root@vps-00001:~# nginx -t
nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
nginx: configuration file /etc/nginx/nginx.conf test is successful
{{< / highlight >}}

І якщо все гаразд, даємо nginx команду перечитати налаштування:

{{< highlight html >}}
root@vps-00001:~# service nginx reload
{{< / highlight >}}

На цьому з налаштуванням nginx завершено, настав час перевірити чи все працює. Візьмемо для прикладу якийсь файл з відео та за допомогою `ffmpeg` розпочнемо трансляцію на сервер:

{{< highlight html >}}
$ ffmpeg -re -i ~/Videos/AntiTrust.avi \
  -vcodec libx264 -preset fast -crf 30 \
  -acodec aac -ab 128k -ar 44100 -strict experimental \
  -f flv rtmp://142.250.201.206/stream/test?access_token=publishonlySECRETpassword
{{< / highlight >}}

Зверніть увагу на те, що замість ip адреси **`142.250.201.206`** ви повинні вказати адрес свого серверу і свій новий згенерований пароль. До речі, у прикладі вище в посиланні вказана назва трансляції **test**, тобто ви можете запускати безліч трансляцій з різними назвами.

Отже, якщо `ffmpeg` не видав помилки, переходимо до завершальної частини - перевірки трансляції. Відкриваємо посилання на трансляцію з відповідним паролем, наприклад, в плеєрі vlc та перевіряємо чи є картинка та звук:

{{< highlight html >}}
$ vlc rtmp://142.250.201.206/stream/test?access_token=watchonlySECRETpassword
{{< / highlight >}}

Можна також одночасно транслювати відео з вебкамери та переглядати зі звуком локально. Це може бути корисним, коли ви, наприклад, захочете транслювати музику зі свого вінілового програвача:

{{< highlight html >}}
$ ffmpeg -re -f v4l2 -i /dev/video0 \
  -f alsa -i default -acodec libmp3lame \
  -map 0 -c:v libx264 -preset ultrafast -crf 23 \
  -map 1:0 -c:a aac \
  -f tee "[f=flv]rtmp://142.250.201.206/stream/test?access_token=publishonlySECRETpassword|[f=nut]pipe:" \
  | ffplay -
{{< / highlight >}}

Але замість **alsa -i default** потрібно буде вказати який самий вхідний канал використовувати для вхідного звукового потоку.
