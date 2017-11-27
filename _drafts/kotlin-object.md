---
layout: post
title:  "Kotlin - это плохо. Object keyword"
lang: ru_RU
---

Продолжаем цикл статей про [`Kotlin`](https://kotlinlang.org/):

 - [Расширения - синтаксический сахар над Utility классами]({{ site.url }}/2017/11/14/kotlin-is-bad.html)
 - [Делегаты]({{ site.url }}/2017/11/23/kotlin-delegates.html)
 - object keyword

Вы часто используете [синглтоны](https://en.wikipedia.org/wiki/Singleton_pattern)? Если так, то вы еще студент... 
или вы пишете на современном JVM языке. Потому что:
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