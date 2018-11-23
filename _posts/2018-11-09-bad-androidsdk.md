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
О ней и пойдет сегодня речь. И основной вопрос, который мне хочется задать 
инженерам из [Google](https://www.google.com): почему вы не 
измените архитектуру SDK? Ведь вы же знаете, что наследование классов - это плохо,
что фрагменты уже много лет [никого не устраивают своим жизненным циклом](https://medium.com/square-corner-blog/advocating-against-android-fragments-81fd0b462c97), что 26'000+ 
строк во `View.java` - это чересчур. Поддержка проектов, написанных на этом `SDK`
страдает именно потому, что `SDK` написано в процедурном стиле. 

Разберем на примере. Практически у каждого Android-разработчика был момент, когда
он в процессе отладки попадает в `View.java` и не может определить проблему. 
Почему так происходит? Вот некоторые причины:

## 26'000+ строк
О чем тут говорить. Это много. Попадая в исходный код такого большого класса, 
конечно же, программист заблудится. Кому-нибудь понятно, зачем в классе `View`
находится метод с названием [`resolveTextAlignment`](https://android.googlesource.com/platform/frameworks/base/+/oreo-release/core/java/android/view/View.java#23967)
? Или переменная [`private static final boolean DBG = false;`](https://android.googlesource.com/platform/frameworks/base/+/oreo-release/core/java/android/view/View.java#773)
? Мне кажется, даже контрибьюторы этого класса боятся вносить туда изменения. Их 
страх оправдан, ведь вероятность непредвиденных ошибок большая в таком огромном 
`scope` этого файла.

## 52% комментариев
Конечно же, нужно больше комментариев, чтобы такой код стал более понятным:
```java
public int mPrivateFlags;
int mPrivateFlags2;
int mPrivateFlags3;
```
Мне кажется, 52% - это мало для такого большого класса. Глядя на код выше мне 
еще не ясно, что же может храниться в этих переменных, и их [комментарий](https://android.googlesource.com/platform/frameworks/base/+/oreo-release/core/java/android/view/View.java#3695) 
мне не помогает. Нужно больше. Если серьезно, то в таком классе запутывает все,
даже комментарии.

## 630 строк комментарий к классу
Это важный пункт. Обычно комментарий к классу содержит описание поведения 
объекта этого класса, возможно, варианты использования. Факт, что тут 630
строк, говорит только о том, что это целая статья, перенасыщенная вариантами
кастомизации и использования `View.java` - хотя бы нарушение принципа единственной 
ответственности.

## 3'700+ строк до первого конструктора
Что же там такого может быть? Описания полей, описания значений, которые 
могут принимать эти поля, описания колбеков, статическая инициализация, и, 
конечно же, куча комментариев. Почему важно то, что этот блок занимает 14% кода 
от всего класса? Потому что с этим так или иначе ведется работа в методах,
некоторые статические поля имеют модификатор доступа `public` и доступны извне.
Мало того, что они доступны, нас заставляют их использовать. Например, если нам 
нужно, чтобы ОС сама смогла заполнить день истечения срока действия платежной 
карты, то для этого `View.java` специально приберегла статическую переменную:
[`AUTOFILL_HINT_CREDIT_CARD_EXPIRATION_DAY`](https://android.googlesource.com/platform/frameworks/base/+/oreo-release/core/java/android/view/View.java#1119)

Со стороны все это выглядит не как объект, а как [пространство имен](https://en.wikipedia.org/wiki/Namespace).
[Процедурное программирование](https://en.wikipedia.org/wiki/Procedural_programming) - вот что нам дал [Google](https://www.google.com).
Как это не печально, большинство современных Android-программ не знают, что такое
объектно-ориентированное программирование, они так и застряли в 70-х. Вы 
думаете, один `View.java` такой большой? Неа:

- RecyclerView.java ~12’000
- TextView.java ~11’000
- ViewGroup.java ~9’000
- Activity.java ~8’000
- Fragment.java ~3’000
- PopupWindow.java ~3’000

Но, даже если проблема была бы только во `View.java`, а все остальные вьюхи были
бы написаны "нормально" (размером меньше, чем 300 строк), то нужно вспомнить,
что эти все остальные наследуются от `View.java`, а наследование `==` 
копирование кусков кода из родительского класса в дочерние, и весь тот `scope`, 
который мы увидели в 26к строчках внезапно поместится в, казалось бы, безобидной 
вьюхе на 300 строк.

Самый главный вопрос: что делать?

Сделать все заново. Сделать заново фреймворк, в котором будут использованы 
принципы ОО программирования, в котором размер файлов не будет превышать 250 
строк, в котором мы не увидим `implementation inheritance`, в котором `scope`
будет соответствовать [I](https://en.wikipedia.org/wiki/Interface_segregation_principle) 
из [SOLID](https://en.wikipedia.org/wiki/SOLID_(object-oriented_design)), в 
котором декларативный подход будет основным для решения задач пользовательского
интерфейса, в котором сделать кастомную вьюху будет обычной задачей, а не 
сверхъестественной, в котором `MaterialDesign` не ограничит простор фантазии 
дизайнера. Можно долго еще рассуждать на тему: "А что можно будет сделать,
если мы будем придерживаться принципа 
[`elegantobjects`](http://www.elegantobjects.org/)?", и я почти уверен, 
результат будет на порядок выше существующего решения, от которого 
[кровь из глаз](https://medium.com/@drinfo/fuck-you-android-framework-ddbb02c4ae48). 

---

Кстати, если вам, как и мне, не нравится класс [`View.java`](https://android.googlesource.com/platform/frameworks/base/+/oreo-release/core/java/android/view/View.java),
то можете прокомментировать [Issue](https://issuetracker.google.com/issues/114273949) про плохую архитектуру
[`View.java`](https://android.googlesource.com/platform/frameworks/base/+/oreo-release/core/java/android/view/View.java).
