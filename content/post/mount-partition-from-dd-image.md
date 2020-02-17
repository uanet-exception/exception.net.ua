---
author: "ZEN"
title: "Монтування розділу диска з образу отриманого за допомогою dd"
date: 2014-09-15T21:48:00+02:00
tags:
  - "linux"
  - "dd"
  - "ddrescue"
  - "gddrescue"
  - "fdisk"
  - "fsck"
categories:
  - "Administration"
type: "post"
archives: "2014"
---

Ситуація наступна - ви маєте на руках носій з даними, але виправити помилки на файловій системі не виходить штатними засобами ОС через апаратні помилки. Ви зробили образ диску за допомогою dd чи ddrescue. І ось тут виникає питання як виправити помилки та вилучити хоч якісь дані з образу диска?

<!--more-->

Раджу спершу зробити ще одну копію образу, бо якщо щось піде не так, ви завжди зможете зробити копію з копії та спробувати інші поради з інтернету по відновленню даних. Отже, до роботи!

Спочатку виправляємо усі помилки ФС за допомогою fsck:
{{< highlight html >}}
# fsck USB.img
{{< /highlight >}}

Та за допомогою fdisk заглядаємо у таблицю розділів, щоб зрозуміти де саме знаходиться розділ чи розділи:
{{< highlight html >}}
# fdisk -l USB.img

Disk USB.img: 978 MiB, 1025506304 bytes, 2002942 sectors
Units: sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes
Disklabel type: dos
Disk identifier: 0x0001df33

Device     Boot Start     End Sectors  Size Id Type
USB.img1   *     2048 2000895 1998848  976M  b W95 FAT32
{{< /highlight >}}

Нас цікавлять розмір сектору диска - 512, та старт розділу - 2048. Отже, пробуємо змонтувати розділ:

{{< highlight html >}}
# mkdir /tmp/USB
# mount -o loop,offset=$((2048*512)) USB.img /tmp/USB
{{< /highlight >}}

І як в наслідок, ви зможете скопіювати файли з директорії /tmp/USB та відмонтувати розділ після всіх операцій:
{{< highlight html >}}
# umount /tmp/USB
# rmdir /tmp/USB
{{< /highlight >}}

Ось і все.
