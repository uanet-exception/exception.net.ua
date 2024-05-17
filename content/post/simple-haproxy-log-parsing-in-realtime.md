---
author: "ZEN"
title: "Простий парсер логів HAProxy у реальному часі"
date: 2020-03-11T15:42:32+02:00
tags:
  - "linux"
  - "python"
  - "tail"
  - "tailf"
  - "haproxy"
categories:
  - "Development"
type: "post"
archives: "2020"
---

Виникло бажання написати Python-скрипт, який буде слідкувати за новими рядками у лог-файлі HAProxy та виводити потрібні мені дані у термінал. За бажанням, дані можна відправити як метрики у StatsD, InfluxDB, Elasticsearch або просто записувати в якусь базу даних, але це вже звісно ідея поза цим прикладом.

<!--more-->
Тож як ми це будемо робити? Писати свою реалізацію `tail -f -F` на Python з нуля? Занадто складно... (Сирцевий код програми tail складає 1752 рядок на C). Залишається скористатися старим добрим `subprocess`. Звісно, pip запропонувал мені деякі готові модулі які виконують те ж саме, але я вирішив трохи переписати код [pytailf](https://bitbucket.org/angry_elf/pytailf/src/default/tailf/__init__.py). Отже у нас в коді буде клас TailF.

Далі нам потрібно вичитувати рядки з stdout/stderr, що поверне нам TailF, та парсити їх. Для цього напишемо функцію `main`, яка буде приймати аргументом шлях до лог-файлу. Насправді, все досить просто. Ось весь сирцевий код:
{{< highlight python >}}
#!/usr/bin/env python3

import os
import re
import time
import fcntl
import select
import signal
import logging
import argparse
import subprocess

signal.signal(signal.SIGINT, lambda _, frame: exit(130))


class TailF:
    def __init__(self, filename):
        self.process = subprocess.Popen(
            ["tail", "-f", "-F", "-n 0"] + [filename],
            stdin=subprocess.PIPE,
            stdout=subprocess.PIPE,
            stderr=subprocess.PIPE)

        # set non-blocking mode for file
        fl = fcntl.fcntl(self.process.stdout, fcntl.F_GETFL)
        fcntl.fcntl(self.process.stdout, fcntl.F_SETFL, fl | os.O_NONBLOCK)

        fl = fcntl.fcntl(self.process.stderr, fcntl.F_GETFL)
        fcntl.fcntl(self.process.stderr, fcntl.F_SETFL, fl | os.O_NONBLOCK)

    def xreadlines(self):
        buf = ''
        while True:
            reads, writes, errors = select.select(
                [self.process.stdout, self.process.stderr], [],
                [self.process.stdout, self.process.stderr], 0.1)

            if self.process.stdout in reads:
                buf += self.process.stdout.read().decode()
                lines = buf.split('\n')

                if lines[-1] == '':
                    # whole line received
                    buf = ''
                else:
                    buf = lines[-1]
                lines = lines[:-1]

                if lines:
                    for line in lines:
                        yield (line, None)

            if self.process.stderr in reads:
                stderr_input = self.process.stderr.read().decode()
                yield (None, stderr_input)


def main(log_file):
    pattern = r'^\S+ \d+ \d+:\d+:\d+ (?P<host>\w+) \S+ (?P<client_ip_port>\S+) \S+ \S+ \S+ \S+ (?P<status>\d+) (?P<bytes_read>\d+) \S+ \S+ \S+ \S+ \S+ \"(?P<http_method>\w+) (?P<path>\S+) \S+\"$'  # noqa: E501
    tail = TailF(log_file)
    for stdout, stderr in tail.xreadlines():
        if stderr:
            logging.error(stderr.rstrip())

        if stdout:
            match = re.match(pattern, stdout)
            if not match:
                logging.error("Can't match regex: " + stdout)
                continue

            data = match.groupdict()
            data['path'] = data.get('path', '').split('?', 1)[0]
            logging.info("{host} {client_ip_port} {status} {path}".format(**data))  # noqa: E501


if __name__ == '__main__':
    def extant_file(x):
        if not os.path.exists(x):
            raise argparse.ArgumentTypeError(
                "File {0} does not exist".format(x))
        return x

    parser = argparse.ArgumentParser()
    parser.add_argument(
        "-f", "--file", help="log file to parse",
        type=extant_file, required=True)
    args = parser.parse_args()

    logging.Formatter.converter = time.gmtime
    format_str = ' '.join([
          '[%(asctime)s]', '[%s]' % 'log-parser',
          '[%(levelname)s]', '%(message)s'
      ])
    logging.basicConfig(level=logging.INFO, format=format_str)
    main(args.file)

{{< /highlight >}}

Ну і приклад того, як сценарій працює:
{{< highlight html >}}
$ ./haproxy-logparser.py -f /var/log/haproxy.log
[2020-03-11 15:39:37,041] [log-parser] [INFO] node2 212.45.XXX.XXX:31948 401 /wp-admin.php
[2020-03-11 15:39:37,041] [log-parser] [INFO] node2 34.248.XX.XX:45111 200 /index.php
[2020-03-11 15:39:37,041] [log-parser] [INFO] node2 212.45.XXX.XXX:32374 401 /wp-admin.php
[2020-03-11 15:39:37,042] [log-parser] [INFO] node2 212.45.XXX.XXX:18544 401 /wp-admin.php
[2020-03-11 15:39:37,042] [log-parser] [INFO] node2 34.248.XX.XX:36210 200 /index.php
[2020-03-11 15:39:37,042] [log-parser] [INFO] node2 212.45.XX.XXX:33682 401 /wp-admin.php
[2020-03-11 15:39:37,042] [log-parser] [INFO] node2 212.45.XXX.XX:19726 401 /wp-admin.php
[2020-03-11 15:39:37,042] [log-parser] [INFO] node2 34.248.XXX.XX:14518 200 /index.php
[2020-03-11 15:39:37,042] [log-parser] [INFO] node2 34.248.XXX.XXX:41169 200 /index.php
[2020-03-11 15:39:37,043] [log-parser] [INFO] node2 34.248.XX.XXX:15166 200 /index.php
{{< /highlight >}}

І ще у випадку ротації лог-файлу haproxy:
{{< highlight html >}}
[2020-03-11 15:49:07,569] [log-parser] [ERROR] tail: '/tmp/haproxy.log' has become inaccessible: No such file or directory
[2020-03-11 15:49:09,834] [log-parser] [ERROR] tail: '/tmp/haproxy.log' has appeared;  following new file
[2020-03-11 15:49:18,678] [log-parser] [INFO] node2 34.254.XXX.XX:20434 200 /index.php
{{< /highlight >}}
