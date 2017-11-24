---
layout: post
title:  "Kotlin - это плохо. Делегаты"
image: /assets/imgs/hard-work.jpg
lang: ru_RU
---

Продолжаем цикл статей про [`Kotlin`](https://kotlinlang.org/):

 - [Kotlin - это плохо. Расширения - синтаксический сахар над Utility классами]({{ site.url }}//2017/11/14/kotlin-is-bad.html)
 - Kotlin - это плохо. Делегаты

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
Вот в этой строчке: `) : Startable by starter {`, эта [языковая конструкция](https://kotlinlang.org/docs/reference/delegation.html) 
с использованием ключевого слова `by` говорит, что реализацию методов интерфейса `Startable` 
нужно делегировать полю `starter`. При этом декомпилированный в `Java` код выглядит так же как и 
пример выше на `Java`.

Что в этом плохого? Ведь теперь тривиальные реализации интерфейсов можно будет не писать.
Вот, какой минус вижу я:
**Теряется контроль за зоной ответственности объекта, так как непосредственные реализации будут скрыты от взгляда 
программиста. На ревью сложно оценить количество методов у класса, который использует 
[Class Delegation](https://kotlinlang.org/docs/reference/delegation.html)**

> На ревью сложно оценить количество методов у класса, который использует 
  [Class Delegation](https://kotlinlang.org/docs/reference/delegation.html)

Допустим, вы читаете чужой код, видите там строчку:
```java
gasolineCar.start()
```
И, вам надо узнать, что происходит в методе `start`. Вы знаете, что этот метод вызывается у объекта класса 
`GasolineCar`, поэтому вы открываете код этого класса и ... там нет метода `start`. П - паника. Конечно, через пару 
часов вы увидите, что там [Class Delegation](https://kotlinlang.org/docs/reference/delegation.html), и метод просто
скрыт, и вы потеряете больше времени на понимание происходящего, чем если бы там был делегат здорового человека.
![Делегат здорового человека - курильщика]({{ site.url }}/assets/imgs/health-delegate.jpg)

---
Используя делегаты в `Koltin` вы обрекаете себя на более высокую стоимость разработки.