---
author: "ihor"
title: "Бот для збору статистики УкрФеді"
date: 2023-02-17T00:06:07+02:00
lastmod: 2023-02-17T00:06:07+02:00
tags:
  - "linux"
  - "bash"
  - "mastodon"
  - "pleroma"
  - "fediverse"
categories:
  - "Development"
  - "Other"
draft: false
type: "project"
archives: "2023"
---

Час від часу, користувачі Fediverse на українських інстансах цікавляться - "Скільки користувачив наразі є в УкрФеді?" і щоб отримати відповідь, потрібно було рахувати вручну кількість користувачів на інстансах, що доволі незручно. Але тепер, щоб дізнатись кількість користувачів в УкрФеді не потрібно рахувати вручну, бо це за вас зробить - `uafedistats`.

<!--more-->

#### Як це працює?

`uafedistats` потрібно попередньо встановити на комп'ютер/сервер та запустити в кроні. 
{{< highlight html>}}
/opt/uafedistats$ echo "*/20  *    * * *   root    bash -c '/opt/uafedistats/uafedistats.sh update &>> /opt/uafedistats/errors.log'" | sudo tee -a /etc/crontab
/opt/uafedistats$ echo "0  *    * * *   root    bash -c '/opt/uafedistats/uafedistats.sh post &>> /opt/uafedistats/errors.log'" | sudo tee -a /etc/crontab
/opt/uafedistats$ sudo service cron reload
{{< /highlight >}}
Далі він сам буде в вибраний вами час робити пост на своїй сторінці в Fediverse

Пости будуть мати вигляд такого плану:
1. Графік
2. Текст над графіком

![uafedistats](/images/2023/uafedistats/uafedistats.png#center)
#### Як цим користуватись?

1. Створіть обліковку та API key.
2. Впевниться в правильності налаштування, після чого чекайте годину(або той час, котрий ви обрали в кроні)

#### Як встановити?

Найпростіше це зробити за допомогою `git clone https://github.com/uanet-exception/uafedistats` після чого почати роботу зі скриптом.

1. Вам потрібен обліковий запис на будь-якому сервері Mastodon/Pleroma, що буде виконувати роль бота.
2. Відредагувати файл конфігурації та базово його налаштувати
{{< highlight html>}}
/opt/uafedistats$ cp main.cfg.example main.cfg
/opt/uafedistats$ vim main.cfg
{{< /highlight >}}
3. Виконати команду `uafedistats.sh update` для того, щоб оновити інформацію про інстанси з релею(в варіанті за замовчуванням це relay.social.net.ua).
4. Виконати команду `uafedistats.sh post` котра зробить пост на вашій обліковці. Якщо ви налаштували все правильно, то пост буде зроблений.

Додаткова інформація: Github: [https://github.com/uanet-exception/uafedistats](https://github.com/uanet-exception/uafedistats)
