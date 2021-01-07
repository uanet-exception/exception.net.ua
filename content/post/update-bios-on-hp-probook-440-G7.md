---
author: "ZEN"
title: "Оновлення BIOS на HP ProBook 440 G7"
date: 2021-01-07T14:22:00+02:00
tags:
  - "linux"
  - "efi"
  - "hp"
  - "bios"
  - "hp proobok 440 G7"
categories:
  - "Hardware"  
type: "post"
archives: "2021"
---

Раніше я розповідав про не зовсім тривіальний процес [оновлення BIOS на старому лептопі HP ProBook 4540s]({{< ref "/post/update-bios-on-hp-probook-4540s.md" >}} "Оновлення BIOS на HP ProBook 4540s"). І як виявилось, для нових лептопів HP цей процес лише трохи відрізняється. Тож сьогодні я розповім вам як оновити BIOS на HP ProBook 440 G7.

<!--more-->

Для початку, поглянемо яка поточна версія BIOS:
{{< highlight html >}}
$ sudo dmidecode -s bios-version
[sudo] password for zen:
S71 Ver. 01.07.02
{{< /highlight >}}

Та йдемо на сторінку завантаження [драйверів та ПО](https://support.hp.com/us-en/drivers/selfservice/hp-probook-440-g7-notebook-pc/29090063) HP, де нам повідомлять, що не вдалось нічого знайти для нашого лептопа та запропонують обрати самому під яку ОС нас цікавить програмне забезпечення. Що ж, тицяємо на ланку "Try manually selecting your operating system" та обираємо останню версію ОС Windows. Сторінка автоматично оновиться і у нас з'явиться можливість завантажити BIOS у форматі exe-файла - в моєму випадку це sp110825.exe версії 01.07.02 Rev.A.

До речі, ви можете помітити, що у мене версія BIOS збігається з тією, що пропонує сайт, тобто мені не обов'язково оновлювати BIOS. Але цікавий момент полягає в тому, що лептоп дозволяє оновлювати BIOS на ту ж саму версію, що може бути корисним у випадку однієї [апаратної проблеми](https://bugs.launchpad.net/ubuntu/+bug/1841039)...

Що ж, BIOS завантажили, залишилось витягти його з exe-файлу та підготуватись до процесу оновлення. Для цього використовуємо наступні команди в терміналі:

{{< highlight bash >}}
$ mkdir -p workdir/HP/{BiosUpdate,DEVFW}
$ mkdir -p workdir/HP/BIOS/{Current,New,Previous}
$ mv sp110825.exe workdir/
$ cd workdir/
$ 7z x sp110825.exe
$ cp *.bin HP/DEVFW/firmware.bin
$ sudo mv HP /boot/efi/EFI/
{{< /highlight >}}

* Якщо ваш лептоп з якоїсь причини працює не в режимі EFI, ви можете скопіювати директорію HP на USB носій з файловою системою FAT32.

Після цього перезавантажуємо ноутбук, по клавіші F1 потрапляємо до System Information, з якої по клавіші Esc ми повинні перейти до Startup Menu де нас цікавить лише самий останній пункт - "Update System and Supported Device Firmware". Лептоп буде перезавантажено десь з три рази у процесі оновлення, то ж не лякайтесь та не вимикайте його до завершення процесу.

Якщо все пройшло вдало та ОС завантажилась, то ви вже знаєте як перевірити поточну версію BIOS:
{{< highlight html >}}
$ sudo dmidecode -s bios-version
[sudo] password for zen:
S71 Ver. 01.07.02
{{< /highlight >}}

Щодо директорії `/boot/efi/EFI/HP`, не поспішайте її видаляти, тому, що після процесу оновлення в неї можна знайти копію попередньої версії BIOS:
{{< highlight html >}}
$ sudo ls /boot/efi/EFI/HP/BIOS/Previous
S71_010601.bin
{{< /highlight >}}
Тобто ви можете повернутись на стару версію, якщо нова має якісь вади.
