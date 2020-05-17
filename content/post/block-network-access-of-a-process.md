---
author: "ZEN"
title: "Ізолюємо процес від доступу до мережі"
# description: ""
date: 2020-05-07T11:40:51+03:00
lastmod: 2020-05-07T11:40:51+03:00
tags:
  - "linux"
  - "bash"
  - "network"
  - "jail"
categories:
  - "Development"
draft: false
type: "post"
archives: "2020"
---
Якщо у вас виникало бажання ізолювати програму від доступу до мережі, то це досить просто.

<!--more-->

Насправді існує [безліч варіантів](https://unix.stackexchange.com/questions/68956/block-network-access-of-a-process) як це зробити, але для себе я вибрав один, той що не потребує інсталяції додаткових утиліт.

Спочатку, треба обрати ім'я для користувальницького простору імен мережі та створити його. Нехай ім'я буде jail:
{{< highlight html>}}
$ sudo ip netns add jail
{{< /highlight >}}

Тепер ми можемо запустити `ping -c 1 1.1.1.1` с правами користувача user:
{{< highlight html>}}
$ sudo ip netns exec jail su user -c 'whoami; ping -c 1 1.1.1.1'
user
connect: Network is unreachable
{{< /highlight >}}

Як бачите, сеанс bash не має доступу до мережі. Якщо вам більше не потрібен простір імен мережі jail, то ви завжди можете його видалити наступною командую:
{{< highlight html>}}
$ sudo ip netns delete jail
{{< /highlight >}}
