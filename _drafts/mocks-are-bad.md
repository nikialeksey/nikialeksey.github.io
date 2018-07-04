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
assertThat(
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

