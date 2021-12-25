---
author: "ZEN"
title: "Паралельне виконання команд на кожному сервері у кластері"
# description: ""
date: 2021-12-25T13:31:23+02:00
lastmod: 2021-12-25T13:31:23+02:00
tags:
  - "perl5"
  - "bash"
  - "ssh"
  - "onall"
  - "linux"
categories:
  - "Software"
  - "Administration"
draft: false
type: "post"
archives: "2021"
---

`onall` - це скрипт на `Perl5`, котрий фактично є зручною обгорткою над ssh, який дозволяє зручно копіювати файл, виконувати команди та скрипти на віддалених серверах, список котрих можна передати onall як у вигляді текстового файлу, так і просто через [стандартний потік введення (stdin)](https://uk.wikipedia.org/wiki/%D0%A1%D1%82%D0%B0%D0%BD%D0%B4%D0%B0%D1%80%D1%82%D0%BD%D1%96_%D0%BF%D0%BE%D1%82%D0%BE%D0%BA%D0%B8#%D0%A1%D1%82%D0%B0%D0%BD%D0%B4%D0%B0%D1%80%D1%82%D0%BD%D0%B5_%D0%B2%D0%B2%D0%B5%D0%B4%D0%B5%D0%BD%D0%BD%D1%8F).

<!--more-->

Сам по собі скрипт невеличкий і не має додаткових залежностей окрім встановленого клієнта ssh та інтерпретатора perl5 (на момент написання статті perl5 предвстановлений на кожному десктоп дистрибутиві лінукс). Тож якщо у вас вже встановлені пакети `git` та `make` виконуємо наступні команди для інсталяції `onall`:

{{< highlight plaintext >}}
$ git clone https://github.com/ticketmaster/onall.git
$ cd onall
~/onall$ sudo make install
{{< /highlight >}}

І ми готові перейти до практичних прикладів. Якщо просто набрати в терміналі команду `onall`, то побачимо мінімальні приклади використання команди:

{{< highlight plaintext >}}
$ onall

onall v2.11 -- A parallelized system-administration tool.
Usage: onall [<options>] <command>
       onall [<options>] --script <script> <script_args>
       onall [<options>] --copy <file> <dest_path>

ERROR: Nothing to do. Exiting.

{{< /highlight >}}

Припустимо, нам потрібно виконати команду `date` на `root@server1.local`, `root@server2.local` та `root@server3.local`, аби впевнитись, що час налаштовано правильно на всіх трьох серверах. Створюємо файл cluster1.list зі списком серверів та запускаємо команду `onall` використовуючи наступну конструкцію:

{{< highlight plaintext >}}
$ onall -f /tmp/cluster1.list date

3 Target(s):

root@server1.local root@server2.local
root@server3.local

The following command will be executed:
	date
Continue (Y/N):y

root@server2.local: Sat Dec 25 13:03:04 UTC 2021
root@server1.local: Sat Dec 25 13:03:04 UTC 2021
root@server3.local: Sat Dec 25 13:03:04 UTC 2021

{{< /highlight >}}

Оскільки назви серверів побудовані за певним шаблоном, ми можемо згенерувати список серверів за допомогою bash так передати їх на вхід команди `onall` у наступний спосіб:

{{< highlight plaintext >}}
$ echo root@server{1..3}.local | onall -q date
root@server2.local: Sat Dec 25 13:04:37 UTC 2021
root@server3.local: Sat Dec 25 13:04:37 UTC 2021
root@server1.local: Sat Dec 25 13:04:37 UTC 2021
{{< /highlight >}}

Як бачите, я ще додав параметр `-q`, щоб `onall` не перепитував чи хочу я виконати команду на запропонованому списку серверів, тому маємо компактніший вивід.

Аби не додавати до кожного сервера "root@", можна передати `onall` ім'я користувача через параметр `-u`:

{{< highlight plaintext >}}
$ echo server{1..3}.local | onall -u root -q date
server2.local: Sat Dec 25 13:05:19 UTC 2021
server3.local: Sat Dec 25 13:05:19 UTC 2021
server1.local: Sat Dec 25 13:05:19 UTC 2021
{{< /highlight >}}

Ускладнимо завдання, скажімо нам потрібно дізнатися на якому сервері яка материнська плата встановлена, а також яка загальна їх кількість у кластері:  

{{< highlight plaintext >}}
$ echo server{1..3}.local | onall -u root -q 'dmidecode -s baseboard-product-name'
server1.local: X9DRT
server2.local: X9DRT
server3.local: X10DRT-P
$ echo server{1..3}.local | onall -u root -q 'dmidecode -s baseboard-product-name' | awk '{print $2}' | sort | uniq -c
      1 X10DRT-P
      2 X9DRT
{{< /highlight >}}

Як бачите, команду `dmidecode` з додатковими параметрами я взяв в лапки, аби `onall` не намагався розібрати його параметри як свої. Втім, інколи буває ситуація, коли через використання одинарних та подвійних лапок неможливо виконати команду, тому залишається лише створити скрипт, та виконати його на всіх серверах ось таким чином:

{{< highlight plaintext >}}
$ echo server{1..3}.local | onall -u root -q --script cron_status.sh
server1.local: cron is active (running)
server2.local: cron is active (running)
server3.local: cron is active (running)
$ echo server{1..3}.local | onall -u root -q --script cron_status.sh stop
$ echo server{1..3}.local | onall -u root -q --script cron_status.sh
server1.local: cron is not active (stopped)
server2.local: cron is not active (stopped)
server3.local: cron is not active (stopped)
{{< /highlight >}}

У наведеному прикладі містичний скрипт `cron_status.sh` може працювати як з параметрами, так і без них. Яким має бути ваш скрипт і чи повинен він підтримувати якісь параметри - залежить лише від ваших потреб та уяви.

Останній приклад, копіювання файлів. У моїй практиці була ситуація, коли потрібно було синхронізувати файл `/etc/hosts` на 50+ серверах, для цього я використовував параметр `--copy`:

{{< highlight plaintext >}}
$ echo server{1..3}.local | onall -u root -q --copy /etc/hosts /etc/hosts
$
{{< /highlight >}}

На додаток додам, що за замовченням `onall` запускає максимум 20 одночасних ssh клієнтів, що можна змінити параметром `-r`. І він не перевіряє, чи сервери онлайн, що можна виправити додавши параметр `-p`, але треба ще буде довстановити у систему `fping`. Ознайомитись з повним списком параметрів можна виконавши команду: `onall -h`
