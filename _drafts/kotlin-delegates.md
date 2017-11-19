---
layout: post
title:  "Kotlin - это плохо. Делегаты"
image: /assets/imgs/hard-work.jpg
lang: ru_RU
---

Продолжаем цикл статей про [`Kotlin`](https://kotlinlang.org/).

В этой статье рассмотрим паттерн [делегат](https://en.wikipedia.org/wiki/Delegation_pattern)
и языковую возможность реализации делегата в Koltin.

![Hard Work by Joachim Bär]({{ site.url}}/assets/imgs/hard-work.jpg)

Паттерн делегат очень простой: один объект передает выполнение своего метода другому объекту. Например, в `Java`
это выглядит так:

```java
interface Startable {
    void start();
}

class Starter implements Startable {
    ...
}
class GasolineCar implements Startable {
    private final Starter starter;
    ...
    @Override
    public void start() {
        starter.start();
    }
}
```
то есть, чтобы завестись, автомобиль передает команду запуска стартеру. Теперь реализуем то же самое на `Kotlin`:
```java
interface Startable {
    fun start()
}
interface Car : Startable
class Starter : Startable {
    ...
}
class GasolineCar(
    private val starter: Starter
) : Startable by starter {
    
}
```
Нет нет, я не забыл реализовать метод `start` у бензинового автомобиля, он там реализован. Разве не видно?
Вот в этой строчке: `) : Startable by starter {`, эта языковая конструкция с использованием ключевого слова `by`
говорит, что реализацию методов интерфейса `Startable` нужно делегировать полю `starter`. При этом декомпилированный 
в `Java` код выглядит точно так же как и пример выше на `Java`.

Что в этом плохого? Ведь теперь тривиальные реализации интерфейсов можно будет не писать.
Вот, какие минусы вижу я:

 - 