---
author: "ZEN"
title: "Ліхтарик Джека на Raspberry Pi"
date: 2017-10-31T23:54:00+02:00
tags:
  - "linux"
  - "Raspberry Pi"
  - "RPi"
  - "HC-SR501"
  - "leds"
  - "python"
categories:
  - "Development"
  - "Hardware"
type: "project"
archives: "2017"
---

Pumpkin Pi - це жартівний проект, мета якого побудувати ліхтарик Джека, що викрикує випадкові фрази та блимає очіма якщо коло нього хтось рухається.

<!--more-->

Щоб реалізувати цю ідею, я мав не більше 12 годин. То ж деякі речі можна було зробити краще... Але я задоволений результатом та радий поділитися досвідом.

Перш за все, потрібно було приєднати датчик руху (HC-SR501) та світлодіоди до "малинки" (які будуть використовуватися замість "очей"). То ж я зробив ось таку просту схему:

[![Pumpkin Pi принципова схема](/images/2017/pumpkin_pi/datasheet.jpg "Pumpkin Pi принципова схема")](/images/2017/pumpkin_pi/datasheet.jpg)

Наступний етап - написати python-скрипт, який буде реагувати на рух. Вийшов ось такий невеличкий скрипт:
{{< highlight python >}}
#!/usr/bin/env python

import os
import time
import glob
import pygame
import random
import logging
import itertools
import RPi.GPIO as GPIO

log = logging.getLogger(__name__)


class MainApp:
    def __init__(self):
        # Use BCM GPIO references instead of physical pin numbers
        GPIO.setmode(GPIO.BCM)
        GPIO.setwarnings(False)
        # Define GPIO to use on Pi
        self.GPIO_PIR = 14
        self.GPIO_LED = 25
        # Set pin as input
        GPIO.setup(self.GPIO_PIR, GPIO.IN)
        GPIO.setup(self.GPIO_LED, GPIO.OUT)
        GPIO.output(self.GPIO_LED, GPIO.LOW)
        # Music files
        resource_dir_glob = os.path.join(
            os.path.abspath(
                os.path.join(__file__, os.pardir, 'resources', '*.mp3')))
        files = glob.glob(resource_dir_glob)
        assert len(files) > 0, "Can't find music files"
        random.shuffle(files)
        self.music_files = itertools.cycle(files)
        pygame.mixer.init()

    def __del__(self):
        # Reset GPIO settings
        GPIO.cleanup()

    def run(self):
        Current_State = 0
        Previous_State = 0

        # Loop until PIR output is 0
        while GPIO.input(self.GPIO_PIR) == 1:
            Current_State = 0
        # Loop until users quits with CTRL-C
        while True:
            # Read PIR state
            Current_State = GPIO.input(self.GPIO_PIR)
            if Current_State == 1 and Previous_State == 0:
                # PIR is triggered
                log.info("Motion detected!")
                track = self.music_files.next()
                pygame.mixer.music.load(track)
                pygame.mixer.music.play()
                # Blink an LED while the sound playing
                while pygame.mixer.music.get_busy() == True:
                    GPIO.output(self.GPIO_LED, GPIO.HIGH)
                    time.sleep(0.5)
                    GPIO.output(self.GPIO_LED, GPIO.LOW)
                    time.sleep(0.5)
                # Record previous state
                Previous_State = 1
            elif Current_State == 0 and Previous_State == 1:
                # REED has returned to ready state
                log.info("Ready")
                Previous_State = 0
            # Wait for 10 milliseconds
            time.sleep(0.01)


if __name__ == '__main__':
    logging.basicConfig(
        format='[%(asctime)s] [%(levelname)s] %(message)s',
        level=logging.ERROR
    )
    log.setLevel(logging.INFO)

    try:
        MainApp().run()
    except KeyboardInterrupt:
        pass
{{< /highlight >}}

Як ви могли помітити, скрипт будує список mp3-файлів з директорії resources, що знаходиться поряд зі скрипом. То ж не забудьте зробити свою підбірку жахливих фраз. ;)

Ну і на солодке, ось вам відео з тим, як це виглядало у процесі розробки:
{{< youtube z7MeaGWjq40 >}}

Фінальний варіант:
{{< youtube kDaqJOrCwRg >}}

І звісно ж актуальний програмний код на [![GitHub](https://github.com/zen-tools/pumpkin-pi)](https://github.com/zen-tools/pumpkin-pi).
