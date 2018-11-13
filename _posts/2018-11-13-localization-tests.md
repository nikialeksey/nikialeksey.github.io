---
layout: post
title:  "Тесты для текстов"
image: /assets/imgs/desk.jpg
lang: ru_RU
tags:
  - android
  - localization
  - hunspell
  - tests
---

Очень часто я получаю баги в трекинговой системе на орфографические ошибки 
(кстати, а вы получаете такие баги?). И вот в этой статье я опишу, как мне удалось снизить 
количество орфографических ошибок в строковых ресурсах. Поехали!

![Desk by Greg Montani]({{ site.url }}/assets/imgs/desk.jpg)

<!--more-->

## Hunspell

На самом деле идея очень простая - чтобы не получать баги на орфографические ошибки в текстах,
нужно написать **тесты на проверку орфографии** текстов в строковых ресурсах. Осталось только найти
надежную и быструю библиотеку проверки орфографии... И такая нашлась! Встречайте - 
[Hunspell](http://hunspell.github.io/) - популярная `spell-checking` библиотека, используется
в [Google Chrome](https://www.google.com/chrome/), [LibreOffice](https://www.libreoffice.org/),
[Apple OS X](https://wikipedia.org/wiki/MacOS) и еще много где. Правда нам нужна возможность 
запуска из Java, и такой способ тоже нашелся - 
[HunspellBridJ](https://github.com/thomas-joiner/HunspellBridJ). Теперь можно в тестах,
написанных на Java, проверять корректность орфографии в словах.

## Как?

Для того, чтобы использовать [`hunspell`](http://hunspell.github.io/) в тестах, я реализовал 
библиотеку-обертку [`arspell`](https://github.com/nikialeksey/arspell). Воспользоваться ей можно так:
```gradle
dependencies {
    testImplementation 'com.nikialeksey:arspell:0.0.2'
}
```

```kotlin
class ResourcesTest {
    @Test
    fun enSpell() {
        val errors = HunspellCheck(
            Hunspell(
                "./src/test/assets/en_US/index.dic",
                "./src/test/assets/en_US/index.aff"
            ),
            AndroidStrings(File("./src/main/res/values-en/strings.xml"))
        ).check()
        Assert.assertTrue(ErrorMessage(errors).asString(), errors.isEmpty())
    }
}
```

Словари для конкретного языка можно взять [тут](https://github.com/wooorm/dictionaries). 
Конечно же, там словари достаточно общие. Это значит, что придется добавлять слова в словари,
и, чтобы иметь такую возможность, необходимо ознакомиться с 
[документацией по `hunspell`](https://www.systutorials.com/docs/linux/man/4-hunspell/).

---

Не получать баги за офрографию можно, достаточно лишь писать unit-тесты на локализацию!

