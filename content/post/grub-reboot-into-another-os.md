---
author: "ZEN"
title: "GRUB2: Перезавантажити систему у іншу ОС з меню завантаження"
# description: ""
date: 2021-05-05T00:37:00+03:00
# lastmod: 2021-05-05T00:37:00+03:00
tags:
  - "awk"
  - "bash"
  - "dialog"
  - "sudo"
  - "grub"
  - "grub-reboot"
  - "linux"
categories:
- "Desktop"
- "Administration"
draft: false
type: "post"
archives: "2021"
---
Уявіть, що ви встановили Linux на комп'ютері поряд з іншою операційною системою. Ось вас вітає після перезавантаження GRUB2 та пропонує обрати в яку систему ви бажаєте завантажитись і... тут на вас чекає неприємний момент - в залежності від положення Місяця на орбіті клавіатура може працювати, а може і не працювати. Клавіатура під'єднана через PS/2 порт і налаштування у BIOS не допомагають розв'язати цю проблему.

<!--more-->

Саме така прикра ситуація сталася з моїм знайомим. Звісно, що дивитись як людина змушена миритись з такою проблемою я не міг. Пригадавши, що колись давно я бачив в KDE3 на OpenSUSE 10 функцію, яка дозволяла із головного меню перезавантажитись у Windows, я трохи повивчав інформацію про `grub-reboot` та написав наступний скрипт:

{{< highlight plaintext >}}
#!/usr/bin/env bash

readarray -t OPTIONS < <(awk -F"'" '$1=="menuentry " || $1=="submenu " {print i++ "\n\"" $2"\""};' /boot/grub/grub.cfg);

TERMINAL=$(tty);
CHOICE=$(dialog --clear --menu "Choose OS to boot at next startup:" 0 0 0 "${OPTIONS[@]}" 2>&1 >$TERMINAL) || exit 0;
sudo grub-reboot $CHOICE;
sudo reboot;
{{< /highlight >}}

Скрипт досить простий. За допомогою `awk` ми отримуємо список ОС з файлу `/boot/grub/grub.cfg` та записуємо  його в змінну `OPTIONS`. Далі за допомогою `dialog` пропонуємо обрати ОС і в разі відмови виконуємо команду `exit 0`. Якщо ж користувач обрав щось зі списку, то за допомогою `grub-reboot` буде створений тимчасовий файл, який буде використаний GRUB2 для обрання іншого пункту меню після завантаження. Команда `reboot` звісно що відправить комп'ютер перезавантажуватись. При запуску із термінала виглядає це якось так:

![screenshot 1](/images/2021/grub-reboot-into-another-os/grub-reboot-menu.png#center "grub-reboot menu")

Аби не запускати кожного разу скрипт вручну із термінала, я вирішив створити відповідний значок на робочому столі.

**Увага, тут та надалі `/home/user` потрібно буде замінювати на домашню директорію вашого користувача.**

Щоб було зручніше, я перемістив скрипт за наступним шляхом `/home/user/.local/bin/grub-once.sh` та скориставшись командою `sudo visudo` додав у файл `/etc/sudoers` наступний рядок аби можна було запускати скрипт без запиту пароля:

{{< highlight plaintext >}}
user ALL=NOPASSWD:/home/user/.local/bin/grub-once.sh
{{< /highlight >}}

Далі на робочому столі був створений файл `grub-once.desktop` з наступним змістом:

{{< highlight plaintext >}}
[Desktop Entry]
Name=Grub reboot chooser
GenericName=Grub reboot chooser
Comment=Choose OS to boot at next startup
Exec=sudo /home/user/.local/bin/grub-once.sh
Terminal=true
Type=Application
Icon=/home/user/.local/share/icons/hicolor/256x256/grub-once.png
Categories=Utility;
StartupNotify=false
{{< /highlight >}}

Тепер просто тицнувши на значок на робочому столі буде автоматично відкриватись ваш улюблений термінал, в якому буде запущено скрипт. Іконку для `desktop` файлу вже самі оберете яка вам подобається. 🙃
