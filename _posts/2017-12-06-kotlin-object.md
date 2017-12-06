---
layout: post
title:  "Kotlin - это плохо. Object keyword"
image: /assets/imgs/security-light.jpg
lang: ru_RU
---

Продолжаем цикл статей про [`Kotlin`](https://kotlinlang.org/):

 - [Расширения - синтаксический сахар над Utility классами]({{ site.url }}/2017/11/14/kotlin-is-bad.html)
 - [Делегаты]({{ site.url }}/2017/11/23/kotlin-delegates.html)
 - `object` keyword
 
![Security light by Chance Agrella]({{ site.url }}/assets/imgs/security-light.jpg)

Вы часто используете [синглтоны](https://en.wikipedia.org/wiki/Singleton_pattern)? Если так, то вы еще не читали
[вопрос про синглтоны](https://stackoverflow.com/questions/137975/what-is-so-bad-about-singletons)... 
или вы пишете на современном JVM языке. Потому что [`документация к Kotlin`](https://kotlinlang.org/docs/reference/object-declarations.html) 
говорит:
![Singleton is a very useful pattern]({{ site.url }}/assets/imgs/singletone-very-useful.jpg)

Например, [Роберт Мартин](https://en.wikipedia.org/wiki/Robert_C._Martin) 
[подробно объясняет](https://8thlight.com/blog/uncle-bob/2015/06/30/the-little-singleton.html), почему не надо 
использовать этот паттерн. Но разработчики из [JetBrains](https://www.jetbrains.com/) думают иначе, и поэтому синглтоны
в `Kotlin` теперь создавать очень просто:
```java
object A {
    fun hello() {
        println("Hello")
    }
}
fun main(args: Array<String>) {
    A.hello()
}
```

`A` - это синглтон. Декомпилируем?
```java
public final class A {
   public static final A INSTANCE;

   public final void hello() {
      String var1 = "Hello";
      System.out.println(var1);
   }

   private A() {
      INSTANCE = (A)this;
   }

   static {
      new A();
   }
}
```
[Не надо использовать синглтоны в коде](http://copist.ru/books/97things-dev/73). Я не первый, кто это говорит. Однако,
в свежем JVM языке синглтон можно сделать используя, всего-навсего, ключевое слово `object`.

---
Вы все еще думаете, что язык, на котором можно очень просто написать неподдерживаемый код, стоит использовать 
в своем проекте? Подумайте еще раз.