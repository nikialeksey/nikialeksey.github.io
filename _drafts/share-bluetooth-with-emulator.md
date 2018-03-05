---
layout: post
title: "Sharing bluetooth devices with android emulator"
lang: ru_RU
---

Всем прекрасно известны ограничения [Android эмулятора](https://developer.android.com/studio/run/emulator.html):
- Bluetooth
- NFC
- SD card insert/eject
- Device-attached headphones
- USB

Но, что делать, если у вас приложение работает с `bluetooth` устройством и вы хотите его отлаживать на эмуляторе,
чтобы не тратить время на процесс передачи и установки `*.apk` файла? Выход есть: 
[VMware Workstation Player](https://www.vmware.com/products/workstation-player.html)
 
Вы спросите, а почему не бесплатный [VirtualBox](https://www.virtualbox.org/)? Все просто: [VirtualBox](https://www.virtualbox.org/)
не умеет на момент написания этой статьи шарить `bluetooth` устройства между гостевой машиной и хостом. С использованием
[VirtualBox](https://www.virtualbox.org/) можно пробросить только `usb` в гостевую систему, и, если у вас есть `usb`
с `bluetooth`, то у вас все получится, но мы хотим оставить порты `usb` свободными на вашем компьютере, и поэтому
будем ~~качать-с-любого-знакомого-торрента~~ покупать [VMware Workstation Player](https://www.vmware.com/products/workstation-player.html).

Итак, на самом деле схема простая:
- Установить `Workstation Player`
- Скачать и установить гостевую [Android x86](http://www.android-x86.org/)
- Настроить [Sharing Bluetooth devices with a virtual machine](https://kb.vmware.com/s/article/2005315)

После этого, все устройства, что видит по bluetooth ваш компьютер, будет видеть Android в `Workstation Player`. 
Осталось сделать так, чтобы команда `adb devices` показывала наш эмулятор в списке. Можно воспользоваться, например,
приложением [WiFi ADB - Debug Over Air](https://play.google.com/store/apps/details?id=com.ttxapps.wifiadb).

Все, после этого я стал значительно меньше тратить времени на запуск и отладку после сборки `*.apk`'шки. Если у вас
та же проблема, если вам, как и мне надоело дебажить взаимодействие с bluetooth переферией своего приложения,
попробуйте этот путь. Даже [Geny Motion](https://www.genymotion.com/) не сможет шарить bluetooth устройства! 

