---
author: "ZEN"
title: "Примусове зупинення процесу з викликом коду 0"
date: 2018-02-01T13:34:00+02:00
tags:
  - "linux"
  - "posix"
  - "gdb"
categories:
  - "Administration"
type: "post"
archives: "2018"
---

Склалась ситуація, коли потрібно перезавантажити Jenkins та не хочеться, що б він відправив сповіщення у Slack неначе дочірній процес аварійно завершився. Відповідь на це питання я знайшов на
[serverfault](https://serverfault.com/questions/390846/kill-a-process-and-force-it-to-return-0-in-linux/515412#515412).

<!--more-->

Ми або запускаємо gdb та використовуючи команду `call exit(0)` зупиняємо процес.

{{< highlight html >}}
# gdb -p <process id>
....
.... Gdb output clipped
(gdb) call exit(0)

Program exited normally.
{{< /highlight >}}

Або робимо все те ж саме, але однією командою:

{{< highlight html >}}
gdb --batch --eval-command 'call exit(0)' --pid <process id>
{{< /highlight >}}
