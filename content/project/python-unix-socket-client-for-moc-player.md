---
author: "ZEN"
title: "Імплементація unix socket клієнта до плеєра MOC"
# description: ""
date: 2020-05-23T17:53:01+03:00
lastmod: 2020-05-23T17:53:01+03:00
tags:
  - "linux"
  - "mocp"
  - "python3"
  - "py-mocp"
categories:
  - "Development"
  - "Desktop"
draft: false
type: "project"
archives: "2020"
---

py-mocp - це python-модуль з імплементацію протоколу консольного плеєра MOC через unix socket. Окрім програмних інтерфейсів, модуль містить в собі сценарій mocp-notify.py, що видає сповіщення при зміні треку.

<!--more-->

Інсталюється модуль дуже просто, для цього використовуємо команду pip3:

{{< highlight html>}}
user@localhost:~$ sudo pip3 install py-mocp
{{< /highlight >}}

До речі, сценарій mocp-notify.py автоматично під'єднується до MOC, якщо роботу останнього було перервано. То ж можна його додати до автозапуску й час від часу отримувати сповіщення від системи:

![mocp-notify.py OSD](/images/2020/py-mocp-0.1rc6/mocp-notify.py.gif#center)

А ось приклад того, як можна відправляти команди до MOC через unix socket використовуючи програмний інтерфейс:
{{< youtube 3DN--W86BO8 >}}

Але майте на увазі, що на відео лише простий приклад. Рекомендую зазирнути у [сирцевий код](https://gitlab.com/zen-tools/py-mocp/-/blob/c589e6fbca46f94809e8da5e9da834ec558449fe/mocp/__init__.py#L500), тому що для нормальної роботи з MOC, клієнт повинен не тільки відправляти команди, але ще й зчитувати відповіді.

Ось і все, сподіваюсь, що це надихне кого-небудь на створення GUI чи WebUI (можна було б на Raspberry PI слухати музику), або взагалі реалізувати API для мобільного додатку.

Додаткова інформація:</br>
PYPI: [https://pypi.org/project/py-mocp/](https://pypi.org/project/py-mocp/)</br>
GitLab: [https://gitlab.com/zen-tools/py-mocp/](https://gitlab.com/zen-tools/py-mocp/)
