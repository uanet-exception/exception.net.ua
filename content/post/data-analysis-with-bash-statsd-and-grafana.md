---
author: "ZEN"
title: "Аналіз даних за допомогою bash, StatsD та Grafana"
# description: ""
date: 2020-02-19T21:23:16+02:00
lastmod: 2020-02-19T21:23:16+02:00
tags:
  - "bash"
  - "udp"
  - "statsd"
  - "graphite"
  - "grafana"
  - "metrics"
  - "docker"
categories:
  - "Development"
type: "post"
archives: "2020"
draft: false
---
Іноді буває треба зібрати якісь дані та побудувати графік з ними. Графік можна використовувати для аналізу деградації системи або для сповіщень у разі критичних ситуацій. Але почнемо з простого — побудуємо графік зміни температури у реальному часі для міста Одеса (Україна).

<!--more-->

Сподіваюсь, що з docker ви вже знайомі, тому що першим ділом ми повинні запустити контейнер з [Grafana та StatsD](https://github.com/uanet-exception/grafana4-statsd#statsd--graphite--grafana-4). Тож відкриваємо термінал та виконуємо наступну команду:

{{< highlight html>}}
$ docker run --rm -it \
    -p 127.0.0.1:8080:80 \
    -p 127.0.0.1:8125:8125/udp \
    --name=grafana4-statd uanetexception/grafana4-statsd:latest
{{< /highlight >}}

І вже через хвилину вам буде доступна Grafana в браузері за адресою **127.0.0.1:8080**

Щоб увійти у Grafana використовуйте логін та пароль admin. На вас тут буде чекати ось така приборна дошка:
![screenshot 1](/images/2020/grafana-statsd-bash/grafana_ukraine_odesa_temp.png#center "Grafana UI")
Звісно, графіки будуть пусті, бо ми ще не надіслали ніяких даних у StatsD. Тож час цим зайнятись.


Заватажте наступний сценарій bash у н файл з назвою weather-odesa.sh:
{{< highlight bash >}}
#!/usr/bin/env bash

APP="weather";
CITY="Odesa";
STATSD_HOST="127.0.0.1";
STATSD_PORT="8125";

which curl &> /dev/null || {
    echo "[$(date -u +"%d-%m-%Y %H:%M:%S")] [ERROR] curl: command not found" 1>&2;
    exit 1;
}

exec 3<> /dev/udp/$STATSD_HOST/$STATSD_PORT;
trap "exec 3<&-; exec 3>&-; exit 0;" EXIT;

while true; do
    sleep 60s;
    TEMP=$(curl -s wttr.in/$CITY?format="%t") || {
        echo "[$(date -u +"%d-%m-%Y %H:%M:%S")] [ERROR] can't retrieve data" 1>&2;
        echo "$APP.health.error:0|c" >&3;
        echo "$APP.health.error:-1|c" >&3;
        continue;
    };
    TEMP=$(echo $TEMP | sed 's|°.*||');

    echo "$APP.temp.$CITY:0|g" >&3;
    echo "$APP.temp.$CITY:$TEMP|g" >&3;

    echo "$APP.health.ok:1|c" >&3;
done
{{< /highlight >}}

Надайте права на виконання цього сценарію, а потім запустіть
{{< highlight bash >}}
$ chmod +x weather-odesa.sh
$ ./weather-odesa.sh
{{< /highlight >}}

Вже через декілька хвилин ви зможете побачити на приборній дошці поточну температуру.
