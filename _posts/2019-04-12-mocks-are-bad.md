---
layout: post
title: "Моки испортят ваши тесты"
lang: ru
image: /assets/imgs/mocks-are-bad/title.png
description: "Почему популярные mocking-фреймврорки портят ваши тесты, и как можно писать тесты без моков."
tags: 
  - testing
  - mocks
  - fakes
---

Вы часто используете [Mockito](https://site.mockito.org/) или другие похожие фреймворки для написания 
[unit-тестов](https://en.wikipedia.org/wiki/Unit_testing)? Если это так, то я знаю, как сделать ваши тесты качественнее - 
никогда не используйте моки! 

![Wheel meme]({{ site.url }}{{ page.image }})

<!--more-->

## Что такое Mock
Для начала нужно разобраться с терминами, что такое моки? Моки - это объекты, созданные с помощью 
[`Mockito.mock`{:.language-java}{:.highlight}](https://static.javadoc.io/org.mockito/mockito-core/2.10.0/org/mockito/Mockito.html):
```java
final Email email = Mockito.mock(Email.class);
```
Получившийся объект реализует `Email`{:.language-java}{:.highlight} интерфейс, но все его методы еще ничего не делают, однако у нас есть возможность
**определить поведение** для его методов:
```java
Mockito.when(
    email.html()
).thenReturn(
    "<p>Hello, Fakes!</p>"
);
```
Теперь можно использовать мок почты для печати на html странице, например:
```java
Assert.assertThat(
    new EmailPage(email).html(), 
    IsEqual.equalTo(
        "<html><body><p>Hello, Fakes!</p></body><html>"
    )
);
```

Иными словами, моки - это объекты, которые генерируются внутри теста, конфигурируются под конкретное поведение и не
более того, и вне теста не могут быть использованы. Если мок использовать как реальный объект взамен настоящему вне 
тестов, то программа не будет работать.

Это значит, что когда моки используются при написании тестов, то приходится думать о том коде, который находится внутри
тестируемого объекта, потому что надо корректно сконфигурировать мок. Такой подход называется 
[white-box testing](https://en.wikipedia.org/wiki/White-box_testing). 
В примере выше при тестировании объекта `EmailPage`{:.language-java}{:.highlight} пришлось держать в голове реализацию метода `EmailPage.html`{:.language-java}{:.highlight}: 
`Email.html`{:.language-java}{:.highlight} вызывается внутри него, значит, нужно определить поведение для этого метода у мока `Email`{:.language-java}{:.highlight}.

И почему же это плохо?

Если изменить реализацию тестируемого метода `EmailPage.html`{:.language-java}{:.highlight}, то придется изменять и содержание теста, придется 
изменить настройку мока, а, может, даже создать еще пару моков, потому что текущее поведения мока `Email`{:.language-java}{:.highlight} было 
определено для старой реализации `EmailPage.html`{:.language-java}{:.highlight}. Старый тест был кем-то написан, на него было потрачено время и,
деньги и вот теперь изменена реализация объекта, но не изменено его поведение, и тест перестал работать, и придется
тратить еще время и деньги на переписывание старого теста. 

> [white-box testing](https://en.wikipedia.org/wiki/White-box_testing)
это плохо, потому что при написании тестов вектор мышления сдвигается в сторону реализации тестируемого объекта, а не в 
сторону его поведения.

## Фейки
Тест выше можно переписать так, чтобы он тестировал поведение, а не реализацию:
```java
Assert.assertThat(
    new EmailPage(new FakeEmail("<p>Hello, Fakes</p>")).html(), 
    IsEqual.equalTo(
        "<html><body><p>Hello, Fakes!</p></body><html>"
    )
);
```
Появился новый класс - `FakeEmail`{:.language-java}{:.highlight} - это легковесная простая реализация интерфейса `Email`{:.language-java}{:.highlight}, которая выглядит примерно 
так:
```java
class FakeEmail implements Email {
    private String content;
    FakeEmail(String content) {
        this.content = content;
    }
    @Override
    String html() {
        return content;
    }
    ...
}
```
Этот объект можно использовать в тестах, он не использует сеть, не использует базу данных, не майнит крипту... Более
того, этот объект можно использовать в коде, ведь он полноценный, и программа будет работать, если все реализации 
`Email`{:.language-java}{:.highlight} заменить на `FakeEmail`{:.language-java}{:.highlight}. Объекты таких классов я называю фейками, и они помогают тестировать другие объекты,
проверяя поведение, а не реализацию.

## Еще пример
Хорошо, но ведь когда явное поведение скрыто, то, кажется, без моков не обойтись. Например:
```java
class FilterRepository {
    private final CloudFile file; // requires network
    ...
    public void updatePredicate(final String predicate) { ... }
}
```
В этом случае для проверки корректности метода `FilterRepository.updatePredicate`{:.language-java}{:.highlight} есть стандартный подход - 
`Mockito.verify`{:.language-java}{:.highlight} для того, чтобы узнать, что некоторый метод объекта `CloudFile`{:.language-java}{:.highlight} был вызван:
```java
@Test
public void updateName() {
    final CloudFile file = Mockito.mock(CloudFile.class);
    final FilterRepository repo = new FilterRepository(file);
    repo.updatePredicate("Cloud");
    Mockito.verify(
        file, 
        Mockito.times(1)
    ).write(ArgumentMatchers.any(String.class));
}
```
Видите? Тут пришлось использовать `any`{:.language-java}{:.highlight}, потому что однозначно не ясно, как `FilterRepository`{:.language-java}{:.highlight} обновляет `predicate` в
облачном хранилище, или же эта часть подвержена частым изменениям. И снова, тестируется то, как `FilterRepository`{:.language-java}{:.highlight} 
взаимодействует со своими зависимостями. Однако, даже в этом случае возможно тестировать поведение объекта:
```java
@Test
public void updateName() {
    final CloudFile file = new FakeCloudFile();
    final FilterRepository repo = new FilterRepository(file);
    repo.updatePredicate("Cloud");
    Assert.assertThat(
        file.content(),
        StringContains.containsString("Cloud")
    )
}
```
Класс `FakeCloudFile`{:.language-java}{:.highlight} - дешевый аналог `CloudFile`{:.language-java}{:.highlight}, реализованный, например, `in-memory` способом, и он позволяет 
тестировать поведение репозитория, а не его реализацию.

---
Моки делают ваши тесты неадаптивными к изменениям в коде, моки заставляют думать о 
реализации, тогда как в тестах нужно думать только о поведении и о взаимодействии объектов (ведь это позволяет писать 
поддерживаемые тесты, которые не надо переписывать, если изменяется реализация объектов), моки относительно дорогой 
ресурс (рефлексия все-таки).

Фейки позволяют сосредоточиться во время написания тестов на поведении объекта, они не 
зависят от реализации, фейки быстрее, поскольку там нет рефлексии, тесты на фейках легче читать и понимать, 
что именно тестируется, какие именно входные данные и как узнать об изменениях в поведении. 

---
Похожие мысли:
- [When Writing Unit Tests, Don’t Use Mocks][when-writing-unit-tests-dont-use-mocks] 
by Seth Ammons
- [Mocking is a Code Smell][mocking-is-a-code-smell] by Eric Elliott

[when-writing-unit-tests-dont-use-mocks]: https://sendgrid.com/blog/when-writing-unit-tests-dont-use-mocks/
[mocking-is-a-code-smell]: https://medium.com/javascript-scene/mocking-is-a-code-smell-944a70c90a6a