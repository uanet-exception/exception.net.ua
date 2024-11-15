---
author: "ZEN"
title: "Доступ до робочого комп'ютера за NAT по ssh"
# description: "Історія про те, як побудувати доступ до робочого Linux-комп'ютера через інший сервер."
date: 2020-05-18T01:55:00+03:00
lastmod: 2020-05-18T01:55:00+03:00
tags:
  - "ssh"
  - "forwarding"
  - "tunneling"
  - "network"
  - "systemd"
categories:
  - "Desktop"
  - "Administration"
menu:
  - menu.main:
    - parent: "post"
draft: false
type: "post"
archives: "2020"
---

Інструкція з того як побудувати ssh-тунель з сервера до робочого Linux-комп'ютера. Маст хев, якщо часто працюєш з дому й забуваєш зробити git push.

<!--more-->

Отже, маємо наступну схему:

{{< highlight html >}}
┌────────────┐       ┌──────────────────────┐       ┌─────────────┐
│ Робочий ПК │ <---> │ your.api.example.com │ <---> │ Домашній ПК │
└────────────┘       └──────────────────────┘       └─────────────┘
{{< /highlight >}}

Певно, що ssh сервіс вже встановлено на сервері your.api.example.com, то ж я відразу перейду до цікавого - відкриваємо термінал та встановлюємо на робочий ПК ssh сервер. Без цього зворотний ssh-тунель не буде працювати.

{{< highlight html >}}
$ sudo apt-get install openssh-server --no-install-recommends
{{< /highlight >}}

Далі нам потрібно згенерувати безпарольний ssh-ключ, який буде використовувати сервіс systemd. По задуму сервіс повинен автоматично під'єднатися до your.api.example.com у разі перебоїв у мережі. Тож виконуємо наступні команди для створення ключа з ім'ям nopwd та додамо публічний ключ на your.api.example.com сервер.

{{< highlight html >}}
$ ssh-keygen -N '' -C work-pc-tunnel -f ~/.ssh/nopwd
$ cat ~/.ssh/nopwd.pub | ssh root@your.api.example.com "cat >>  ~/.ssh/authorized_keys"
$ # Перевіряємо, чи ключ працює нормально
$ ssh root@your.api.example.com -i ~/.ssh/nopwd whoami
root
{{< /highlight >}}

Тепер у нас є все для створення сервісу systemd, тож створюємо /etc/systemd/system/backdoor.service файл з наступним змістом:

{{< highlight html >}}
[Unit]
Description=SSH Tunnel to your.api.example.com
After=network.target

[Service]
Type=simple
ExecStart=/usr/bin/ssh \
    -o StrictHostKeyChecking=no -o ExitOnForwardFailure=yes \
    -o TCPKeepAlive=yes -o ConnectTimeout=10 \
    -o ServerAliveInterval=300 -o ServerAliveCountMax=2 \
    -n -N -R 2222:localhost:22 root@your.api.example.com -i /home/user/.ssh/nopwd
ExecReload=/bin/kill -HUP $MAINPID
KillMode=process
Restart=always
RestartSec=10s
StartLimitIntervalSec=0

[Install]
WantedBy=default.target
{{< /highlight >}}

Активуємо сервіс та перевіряємо його статус:

{{< highlight html >}}
$ sudo systemctl daemon-reload
$ sudo systemctl enable backdoor.service
$ sudo systemctl start backdoor.service
$ sudo systemctl status backdoor.service | head -n3
● backdoor.service - SSH Tunnel to your.api.example.com
     Loaded: loaded (/etc/systemd/system/backdoor.service; enabled; vendor preset: enabled)
     Active: active (running) since Sun 2020-05-17 23:41:13 EEST; 28s ago
{{< /highlight >}}

Отже, все гаразд, переходимо до домашнього ПК, підключаємося до your.api.example.com, а з нього вже до робочого ПК:

{{< highlight html >}}
$ ssh root@your.api.example.com
root@vps-00001:~# ssh user@localhost -p 2222
user@work:~$
{{< /highlight >}}

Або все те ж саме, але однією командою:

{{< highlight html >}}
$ ssh root@your.api.example.com -t ssh user@localhost -p 2222
user@work:~$
{{< /highlight >}}

Ось і все, щасливого хакінгу!
