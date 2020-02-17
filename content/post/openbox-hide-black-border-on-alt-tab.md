---
author: "ZEN"
title: "Як приховати чорну рамку при перемиканні вікон в Openbox"
date: 2015-06-24T22:51:00+02:00
tags:
  - "linux"
  - "openbox"
categories:
  - "Desktop"
type: "post"
archives: "2015"
---

Всюди користуюсь менеджером вікон Openbox, та вирішив розібратись як вимкнути надокучливі чорні рамки вікон, коли перемикаю їх по Alt+Tab. Як виявилося, це дуже просто зробити.

<!--more-->

Ось, подивіться в останнє як це виглядає:
![screenshot 1](/images/2015/openbox/openbox-black-borders-window.png#center "Openbox black border window")

А тепер відкривайте в улюбленому текстовому редакторі файл `~/.config/openbox/rc.xml`, та шукайте наступну секцію:
{{< highlight html >}}
<keybind key="A-Tab">
  <action name="NextWindow"/>
</keybind>
{{< /highlight >}}

І приведіть її до наступного вигляду
{{< highlight html >}}
<keybind key="A-Tab">
  <action name="NextWindow">
    <bar>no</bar>
  </action>
</keybind>
{{< /highlight >}}

Виконайте в терміналі команду `openbox --reconfigure` та насолоджуйтесь результатом.
