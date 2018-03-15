---
layout: post
title:  "Ревью Elegant Objects vol.1"
image: /assets/imgs/seeking-the-answer.jpg
lang: ru_RU
tags:
  - elegantobjects
  - bookreview
---

После почти 2-ух месяцев чтения книги [Elegant Objects vol.1](http://www.yegor256.com/elegant-objects.html) я готов рассказать 
о том, почему эту книгу стоит прочитать каждому программисту.

> Почему два месяца? Потому что я читал в рабочее время, в промежутки, пока шла сборка Android-приложения, над которым 
я работаю ;)

![Seeking the answer by James Robertson]({{ site.url }}/assets/imgs/seeking-the-answer.jpg)

<!--more-->

Вам нужно прочесть эту книгу чтобы осознать то вранье, которое лилось из всех других IT-книг/статей/конференций про 
хороший код и объектно-ориентированное программирование. Вам следует прочесть эту книгу, потому что там вы найдете 
максимум ответов на вопросы о хорошей архитектуре кода. Вам нужно прочесть эту книгу, потому что David West - автор
книги [Object Thinking](http://davewest.us/product/object-thinking/) - 
[сказал](https://twitter.com/yegor256/status/933428055464398848) о ней следующее: "Книга Егора  демонстрирует то, как 
правильно и элегантно реализовать концепцию объекта в коде; в то время как другие книги показывают, как использовать 
код, чтобы искажать и разлагать концепцию объекта."

<blockquote class="twitter-tweet" data-lang="ru"><p lang="en" dir="ltr">David West, the author of &quot;Object Thinking&quot; reviewed my books today: &quot;Yegor&#39;s books show you how to correctly and elegantly implement the object concept in code; while all other books show you how to use code to warp and corrupt the object concept.&quot; <a href="https://twitter.com/hashtag/elegantobjects?src=hash&amp;ref_src=twsrc%5Etfw">#elegantobjects</a> <a href="https://twitter.com/hashtag/oop?src=hash&amp;ref_src=twsrc%5Etfw">#oop</a></p>&mdash; Yegor Bugayenko (@yegor256) <a href="https://twitter.com/yegor256/status/933428055464398848?ref_src=twsrc%5Etfw">22 ноября 2017 г.</a></blockquote>
<script async src="https://platform.twitter.com/widgets.js" charset="utf-8"></script>

---

Я уже второй день пытаюсь понять код вот этого
[Sync.java](https://github.com/LWJGL/lwjgl/blob/master/src/java/org/lwjgl/opengl/Sync.java) файла из библиотеки 
[LWJGL](https://www.lwjgl.org/). Этот код содержит логику фиксации количества кадров в секунду при отрисовке графики. 
Если вы сами на него взглянете, то ваша реакция будет примерно такой: "Тут ничего не ясно! Надо переписать!..". Код 
сложно понять, и поддерживать, я думаю, его не возможно, потому что в нем использованы:

- статические методы (по сути свой `Sync` - это `Utility`-класс)
```java
class Sync {
    public static void sync(int fps) { ... }
    private static void initialise() { ... }
    private static long getTime() { ... }
    private static class RunningAvg { ... }
}
```
- перегруженные смыслом магические константы
```java
// don't change: 0.9f is exactly right!
private static final float DAMPEN_FACTOR = 0.9f;
```
- методы, которые берут на себя слишком много ответственности (при инициализации метод интересуется платформой, на 
которой он запускается)
```java
private static void initialise() {
    i...
    
    if (osName.startsWith("Win")) {
        // On windows the sleep functions can be highly inaccurate by 
        // over 10ms making in unusable. However it can be forced to 
        // be a bit more accurate by running a separate sleeping daemon
        // thread.
        ...
    }
}
```

Файл состоит всего лишь из 175 строчек, но чтобы понять то, что там написано, нужно потратить много времени и усилий.
Еще сложнее будет исправлять баги в этом коде. Автору 
[Sync.java](https://github.com/LWJGL/lwjgl/blob/master/src/java/org/lwjgl/opengl/Sync.java) достаточно было бы узнать, 
что `Utility`-классы - это противоположность поддерживаемости, что следует делать полноценные объекты, определять их тип
через интерфейсы, уменьшить область использования магических констант (если совсем не убрать их), и вместо одного 
класса со статическими методами стало бы три-четыре класса, но с целевым и прозрачным поведением и декларативным 
описанием. Такие объекты стало бы легче поддерживать и тестировать, их стало бы легче читать.

Книга [Elegant Objects vol.1](http://www.yegor256.com/elegant-objects.html) о том, чтобы писать код так, что программист будет 
тратить меньше времени на его чтение, и, как следствие, дешевле и качественнее вести разработку. Практически на каждой 
странице этой книге вы найдете фразу: "It's all about maintainability", потому что все практические рекомендации 
направлены на повышение уровня поддреживаемости вашего кода. 

Вам рассказывали в университете, что объектно-ориентированное программирование позволило писать большие программы, 
которые можно было бы поддерживать длительное время. Но попробуйте честно ответить на вопрос: как часто вы хотите 
переписать свой код? Чаще, чем раз в месяц? Тратила ли ваша компания деньги на реализацию того, что уже реализовано, с
целью изменение архитектуры, потому что поддерживать текущее слишком дорого? Именно поэтому книгу 
[Elegant Objects vol.1](http://www.yegor256.com/elegant-objects.html) необходимо прочитать каждому программисту.