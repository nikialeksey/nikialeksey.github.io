---
layout: post
title:  "Плохой Android SDK"
image: /assets/imgs/chirita-the-chicken.jpg
lang: ru_RU
tags:
  - google
  - android
  - androidsdk
  - elegantobjects
  - badandroidsdk
---

Разработка мобильных приложений для платформы [Android](https://www.android.com/) - одно из самых популярных направлений
в области программирования на сегодняшний день. [Google](https://www.google.com) активно развивает эту систему и делает 
огромные шаги вперед в каждой новой версии ОС. Например, в версии 
[Oreo](https://www.android.com/versions/oreo-8-0/) появилось 
[серьезное улучшение](https://source.android.com/devices/architecture/treble) в архитектуре системы - `Project Treble`, 
влияющее на безопасность конечных потребителей. Но я считаю, что [Google](https://www.google.com) мог бы и лучше...

![Chirita The Chicken by analyser]({{ site.url }}/assets/imgs/chirita-the-chicken.jpg)

<!--more-->

[Project Treble](https://source.android.com/devices/architecture/treble) - повторюсь,
это классная архитектурная штука, придуманная инженерами [Google](https://www.google.com),
чего нельзя сказать про [`AndroidSDK`](https://android.googlesource.com/platform/frameworks/base/) - 
это ужасная архитектурная штука, которая заставляет страдать разработчиков приложений.
И основной вопрос, который мне хочется задать инженерам из [Google](https://www.google.com): почему вы не 
измените архитектуру SDK? Ведь вы же знаете, что наследование классов - это плохо,
что фрагменты уже много лет никого не устраивают своим жизненным циклом, что 26'000+ 
строк во `View.java` - это чересчур. Поддержка проектов, написанных на этом `SDK`
страдает именно потому, что `SDK` написано в процедурном стиле. 

Разберем на примере. Практически у каждого Android-разработчика был момент, когда
он в процессе отладки попадает в `View.java` и не может определить проблему. 
Почему так происходит? Мне кажется вот некоторые причины:

##26'000+ строк
О чем тут говорить. Это много. Попадая в исходный код такого большого класса, 
конечно же, программист заблудится. Кому-нибудь понятно, зачем в классе `View`
находится метод с названием [`resolveTextAlignment`](https://android.googlesource.com/platform/frameworks/base/+/oreo-release/core/java/android/view/View.java#23967)
? Или переменная [`private static final boolean DBG = false;`](https://android.googlesource.com/platform/frameworks/base/+/oreo-release/core/java/android/view/View.java#773)
? Мне кажется, даже контрибьюторы этого класса боятся вносить туда изменения. Их 
страх оправдан, ведь вероятность непредвиденных ошибок большая в таком огромном 
`scope` этого файла.

##52% комментариев
Конечно же, нужно больше комментариев, чтобы такой код стал более понятным:
```java
public int mPrivateFlags;
int mPrivateFlags2;
int mPrivateFlags3;
```


##630 строк комментарий к классу
##3'700+ строк до первого конструктора  

[presentation]({{ site.url }}/assets/presentations/badandroidsdk.html)