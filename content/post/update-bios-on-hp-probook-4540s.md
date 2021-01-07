---
author: "ZEN"
title: "Оновлення BIOS на HP ProBook 4540s"
date: 2019-12-16T23:23:00+02:00
tags:
  - "linux"
  - "efi"
  - "hp"
  - "bios"
  - "hp probook 4540s"
categories:
  - "Hardware"  
type: "post"
archives: "2019"
---

Зазвичай, процес оновлення біос не є складним. Але компанія HP зробили з цього виклик для лінукс користувачів. Бо коли ви відкриваєте офіційну сторінку підтримки HP ProBook 4540s та сайт виявляє, що ви користуєтеся лінукс, то вам буде пропоновано інсталяцію лише застарілої версії біос - F.01 (Липень 5, 2012). Але, якщо ви відреагуєте на пропозицію вибрати іншу ОС, та вкажете останній реліз віндовс, то вам буде пропоновано іншу версію - F.68 (Травень 14, 2019).

Мабуть, відповідальна за інсталятор для лінукс людина вже досить давно звільнилася з HP, але будемо вдячні йому або їй за tgz-архів. Тому що саме зміст tgz-архіву підштовхнув мене розпакувати exe-файл для віндовс та винайти спосіб як оновити біос ноутбука використовуючи розділ UEFI.

<!--more-->

До речі, якщо не впевнені, чи взагалі вам потрібно оновлювати біос, то перевірте поточну версію наступною командою:
{{< highlight html >}}
$ sudo dmidecode -s bios-version
[sudo] password for zen:
68IRR Ver. F.64
{{< /highlight >}}
Та й йдемо далі.

Завантажуємо exe-файл з біосом ось [тут](https://support.hp.com/us-en/drivers/selfservice/hp-probook-4540s-notebook-pc/5229455) (у моєму випадку я оновлював біос до версії F.68 - файл sp96086.exe). Та використовуємо наступні команди в терміналі:

{{< highlight bash >}}
$ mkdir -p workdir/HP/{BIOSUpdate,tmp}
$ mkdir -p workdir/HP/BIOS/{Current,New,Previous}
$ mv ~/sp96086.exe workdir/
$ cd workdir/tmp
$ 7z x ../sp96086.exe
$ 7z x ROM.CAB
$ BIOS_NAME=$(head -n1 ver.txt | awk '{print $2}')
$ echo $BIOS_NAME
68IRR
$ cp Rom.bin ../HP/BIOS/New/$BIOS_NAME.BIN
$ cp Rom.sig ../HP/BIOS/New/$BIOS_NAME.sig
$ cp efibios.sig ../HP/BIOS/New/$BIOS_NAME.s12
$ cp ver.sig ver.txt ../HP/BIOS/New/
$ ls ../HP/BIOS/New/
$ 68SRR.BIN  68SRR.s12  68SRR.sig  ver.sig  ver.txt
$ cp BIOSUpdate/* ../HP/BIOSUpdate/
$ cd ..
$ rm -rf tmp
$ sudo mv HP /boot/efi/EFI/
{{< /highlight >}}

Після цього перезавантажуємо ноутбук, по F10 отримуємо доступ до біосу та вибираємо функцію оновлення. Буде ще одне перезавантаження за яким піде процес оновлення біос.

Якщо все пройшло вдало, то після завантаження в ОС, перевіряємо чи версія біос оновилась:
{{< highlight html >}}
$ sudo dmidecode -s bios-version
[sudo] password for zen:
68IRR Ver. F.68
{{< /highlight >}}

А також не забуваємо видалити вже непотрібні файли з директорії EFI:
{{< highlight html >}}
$ sudo rm -r /boot/efi/EFI/HP
{{< /highlight >}}
