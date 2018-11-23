---
layout: post
title:  "Kotlin - это плохо. Расширения - синтаксический сахар над Utility классами"
image: /assets/imgs/steps-to-down.jpg
lang: ru_RU
---

Начинается цикл статей, которые будут посвящены языку [`Kotlin`](https://kotlinlang.org/):

 - Расширения - синтаксический сахар над Utility классами
 - [Делегаты]({{ site.url }}/2017/11/23/kotlin-delegates.html)
 - [`object` keyword]({{ site.url }}/2017/12/06/kotlin-object.html)

![Steps going down by Chance Agrella]({{ site.url }}/assets/imgs/steps-to-down.jpg)

<!--more-->

После [анонсирования](https://youtu.be/X1RVYt2QKQE) `Kotlin`'а как официального языка для разработки под 
[Android](https://www.android.com/) все больше и больше разработчиков 
[стали использовать](https://realm.io/realm-report/2017-q4/)этот язык в своих проектах. 

![Kotlin using statistics]({{ site.url }}/assets/imgs/kotlin-using-statistics.jpg)

Это обусловлено, в первую очередь, тем, что `Kotlin` - это что-то свежее, в отличие от
`Java 1.7`. Например, в `Koltin` есть [`lambda`](https://kotlinlang.org/docs/reference/lambdas.html):
```java
val sum = { a, b -> a + b}
println(sum(1, 2)) // 3
```

А еще есть [функции расширения](https://kotlinlang.org/docs/reference/extensions.html), 
[дата классы](https://kotlinlang.org/docs/reference/data-classes.html), 
[делегирование](https://kotlinlang.org/docs/reference/delegation.html), 
[null safety](https://kotlinlang.org/docs/reference/null-safety.html), 
даже [перегрузка операторов](https://kotlinlang.org/docs/reference/operator-overloading.html) и еще много разного и интересного в 
[стандартной библиотеке](https://kotlinlang.org/api/latest/jvm/stdlib/index.html).

С одной стороны, свежий язык, про который многие говорят: "Я больше не буду писать на `Java`, ведь есть `Kotlin`",
а с другой стороны огромная потеря в поддержке кода и в масштабировании. Не верите? Начнем с расширений.

## Расширения
Расширение - это функция, которую можно вписать в любой класс. Например, у класса `java.util.List` явно не хватает 
метода `first`, который возвращал бы первый элемент списка. Как эта проблема решена в `Java`? Там есть 
[`Guava`](https://github.com/google/guava/), в которой полно статических методов, и 
[один из них](https://github.com/google/guava/blob/master/guava/src/com/google/common/collect/Iterables.java#L808-L810)
решает нашу задачу:
```java
final List<String> words = ...
final String first = Iterables.getFirst(words, "");
```
Но теперь, когда у нас есть `Kotlin`, мы можем решить эту задачу с помощью... барабанная дробь...
**статического метода**
```java
fun <T> List<T>.first(): T {
    return this.get(0)
}
```
(такой метод даже в [стандартной библиотеке](https://github.com/JetBrains/kotlin/blob/1.1.3/libraries/stdlib/src/generated/_Collections.kt#L176-L180) есть).
Да, функции расширения в `Koltin` - это просто статические методы. Если метод выше декомпилировать в `Java`, то получится
примерно следующее:
```java
public static final Object first(@NotNull List $receiver) {
    Intrinsics.checkParameterIsNotNull($receiver, "$receiver");
    return $receiver.get(0);
}
```
То есть функции расширения дают возможность использовать утилитарные методы иначе:
```java
val words: List<String> = ...
val first = words.first()
```

Рассмотрим жизненный пример: почтовый клиент. Клиент написан давно, вы работаете над программой две недели и вам дают 
задачу: считать количество слов при отображении письма и выводить это количество в информационной строке. 
Вы находите интерфейс `Post`, и пишите для него метод расширение (в другом файле):
```java
interface Post {
    fun body(): String
}
fun Post.wordsCount(): Int {
    val text = this.body()
    ...
}
```
Начинаете тестировать новый метод и видите, что подсчет слов работает неправильно, он считает на два слова больше, чем
нужно. Оказывается, есть декоратор над `Post` - `HtmlPost`, который форматирует текст письма в соответствии с заданными
настройками цветов, так вот он добавляет два тега - открывающий и закрывающий к телу письма:
```java
class HtmlPost {
    override fun body(): String {
        return "<post> ${origin.body()} </post>"
    }
}
```
Что же делать? Если написать расширение для `HtmlPost` (это все равно, что другой статический метод в `Utility` классе),
то либо будет дублирование кода, потому что уже написан код для подсчета слов, нужно только теги игнорировать, либо
из одного статического метода будет вызываться другой статический метод и появятся проблемы с тестированием.

Другая проблема - постоянный вопрос при написании метода расширения: В каком файле писать `extension` функцию? Еще
похожий вопрос: А, может, уже написана эта функция? А если написана похожая, и не подходит для решения текущей задачи?
Дублировать или пытаться расширять и изменять? Когда дело касается утилитных методов, то все, что было в ООП становится
неприменимым: [декораторы](https://en.wikipedia.org/wiki/Decorator_pattern), 
[фабрики](https://en.wikipedia.org/wiki/Factory_method_pattern), 
[композиции](https://en.wikipedia.org/wiki/Composite_pattern) - это невозможно сделать со статическими методами.

---

Этот пример показывает, что функции расширения в `Kotlin` имеют те же проблемы, что и статические методы в 
`Utility` классах. То есть, другими словами, используя расширения придется больше времени тратить на поддержку кода.
В следующей статье рассмотрим, куда приведет использование 
[делегирования](https://kotlinlang.org/docs/reference/delegation.html).
