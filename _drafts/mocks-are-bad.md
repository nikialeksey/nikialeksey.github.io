---
layout: post
title: "Mocks are bad. Don't use it"
lang: en
image: 
description: "Why most popular testing frameworks spoil your tests and how we can live without mocks."
tags: 
  - testing
  - mocks
  - fakes
---

Do you use [Mockito](https://site.mockito.org/) often or some similar frameworks for writing your tests?
So, I know how to improve your tests. 

<!--more-->

Well, first we have to know what is the mocks. Simply, mocks are objects created by `Mockito.mock`:
```java
final Email email = Mockito.mock(Email.class);
```
The resulting object implements `Email` interface but his methods do nothing yet. We can define behavior for his 
methods:
```java
Mockito.when(
    email.html()
).thenReturn(
    "<p>Hello, Fakes!</p>"
);
```
And then we can test object which using email for printing, for example:
```java
Assert.assertThat(
    new EmailPage(email).html(), 
    IsEqual.equalTo(
        "<html><body><p>Hello, Fakes!</p></body><html>"
    )
);
```

So, if we use mocks, then we have to think about implementation of testing object in writing tests - 
so-called [white-box testing](https://en.wikipedia.org/wiki/White-box_testing). 
In the sample above in testing of `EmailPage` we needed knowledge about implementation of the method `EmailPage#html`: 
`Email#html` is calling in it, and we had to define this behavior for `Email` mock.
The bad thing is: if you change implementation of `EmailPage#html`, then you have to change test, because the behavior
of defined for old implementation. In the nutshell, white-box testing is bad, because in writing tests you 
need to think about how to test objects, not about how they are implemented.

Ok, what about mocks for testing object with hidden behavior?
```java
class FilterRepository {
    private final CloudFile file;
    
    ...
    
    public void updateName(final String name) { ... }
}
```
In this case most common pattern for assert - `Mockito.verify` to verify that some method of `CloudFile` was called:
```java
@Test
public void updateName() {
    final CloudFile file = Mockito.mock(CloudFile.class);
    final FilterRepository repo = new FilterRepository(file);
    repo.updateName("Cloud");
    Mockito.verify(file, Mockito.times(1)).write(ArgumentMatchers.any(String.class));
}
```
See it? We had to add `any` because we don't really know how `FilterRepository` update `name` in some cloud 
persistence, or that part is changing often. What's wrong? We need to know in test how `FilterRepository` work with
him dependencies. But we must think about behavior in tests:
```java
@Test
public void updateName() {
    final CloudFile file = new FakeCloudFile();
    final FilterRepository repo = new FilterRepository(file);
    repo.updateName("Cloud");
    Assert.assertThat(
        file.content(),
        StringContains.containsString("Cloud")
    )
}
```
Fake `FakeCloudFile` is a cheap in-memory implementation of `CloudFile`, and it allow us to test behavior and
barely think about implementation of testing object.

Пример сложного взаимодействия в одном методе, когда подготовка моков занимает бОльшую часть строк в тесте. Показать, 
что проще (понятнее и поддерживаемее) было бы создать и подготовить несколько фейков, чем следить за 100500 настройками
моково, которые придется переписывать при каждом изменении реализации без изменения контракта. (поискать тесты на 
ShiftsImpl)

Подготовить сравнительный тест и найти пример того, что с моками скорость выполнения тестов падает примерно в 10 раз

Вывод примерно такой: моки полезны при слабой связности внутри объекта (кто не знает, слабая связность - это плохо, 
так как слабая связность является нарушением целостности, первый принцип из SOLID), моки заставляют думать о 
реализации, тогда как в тестах нужно думать только о поведении и о взаимодействии объектов, моки относительно дорогой 
ресурс (рефлексия все-таки); фейки позволяют сосредоточится во время написания тестов на поведении объекта, они не 
зависят от реализации (придется меньше переписывать тесты, а, значит, они будут дольше работать - тут можно вспомнить 
пример про тесты AliPay, когда они были выкинуты из за того, что поменялось АПИ и было тяжело разобраться в тестах,
чтобы их адаптировать, проще было написать новые - и это вызволо проблемы в продакшене), фейки быстрее, поскольку там 
нет рефлексии, тесты на фейках легче читать и понимать, что именно тестируется, какие именно входные данные и как 
узнать об изменениях в поведении.

