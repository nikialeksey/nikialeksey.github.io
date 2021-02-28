---
layout: post
title: "Analyser"
lang: ru
image: /assets/imgs/skeletons.jpg
description: ""
tags:
- oop
- code analysis
---

Статический анализатор обычно помогает поддерживать выбранный стиль кода. Иногда
он находит нетривиальные шаблонные проблемы. Но сегодня посмотрим на то,
как статический анализатор заставляет менять даже архитектуру.

![Videoconference Call by Jack Moreh]({{ site.url }}{{ page.image }})

<!--more-->

Я решал задачу получения текстового контента из объекта, представляющее 
электронное письмо. Оказывается, задача не совсем тривиальная, ведь тело письма 
может состоять из разных частей: вложения, html, альтернативный текст и другие
([RFC-2045](https://tools.ietf.org/html/rfc2045#section-5.1)). Вот ссылка на код,
который [JakartaMail](https://en.wikipedia.org/wiki/Jakarta_Mail) любезно предложил 
мне использовать для этой задачи: 
[How do I find the main message body in a message that has attachments? ](https://javaee.github.io/javamail/FAQ#mainbody)
Я вставил этот метод в свой класс `Mail`, где мне нужно было написать получение
контента из [`MimeMessage`](https://jakarta.ee/specifications/mail/1.6/apidocs/javax/mail/internet/MimeMessage.html):
```java
import javax.mail.MessagingException;
import javax.mail.Multipart;
import javax.mail.Part;
import java.io.IOException;

public final class Mail {

    private boolean textIsHtml = false;

    /**
     * Return the primary text content of the message.
     */
    private String getText(Part p) throws
        MessagingException, IOException {
        if (p.isMimeType("text/*")) {
            String s = (String)p.getContent();
            textIsHtml = p.isMimeType("text/html");
            return s;
        }

        if (p.isMimeType("multipart/alternative")) {
            // prefer html text over plain text
            Multipart mp = (Multipart)p.getContent();
            String text = null;
            for (int i = 0; i < mp.getCount(); i++) {
                Part bp = mp.getBodyPart(i);
                if (bp.isMimeType("text/plain")) {
                    if (text == null)
                        text = getText(bp);
                    continue;
                } else if (bp.isMimeType("text/html")) {
                    String s = getText(bp);
                    if (s != null)
                        return s;
                } else {
                    return getText(bp);
                }
            }
            return text;
        } else if (p.isMimeType("multipart/*")) {
            Multipart mp = (Multipart)p.getContent();
            for (int i = 0; i < mp.getCount(); i++) {
                String s = getText(mp.getBodyPart(i));
                if (s != null)
                    return s;
            }
        }

        return null;
    }
}
```
и запустил статический анализатор iwillfailyou:
```java
> Task :iwillfailyou FAILED

nullfree
Mail.getText(Mail.java:24) > null
Mail.getText(Mail.java:28) > null
Mail.getText(Mail.java:33) > null
Mail.getText(Mail.java:44) > null
Mail.getText(Mail.java:49) > null

allfinal
Mail(Mail.java:8) > textIsHtml = false
Mail.getText(Mail.java:16) > String s = (String) p.getContent()
Mail.getText(Mail.java:23) > Multipart mp = (Multipart) p.getContent()
Mail.getText(Mail.java:24) > String text = null
Mail.getText(Mail.java:25) > int i = 0
Mail.getText(Mail.java:26) > Part bp = mp.getBodyPart(i)
Mail.getText(Mail.java:32) > String s = getText(bp)
Mail.getText(Mail.java:41) > Multipart mp = (Multipart) p.getContent()
Mail.getText(Mail.java:42) > int i = 0
Mail.getText(Mail.java:43) > String s = getText(mp.getBodyPart(i))
Mail.getText(Mail.java:13) > Part p

allpublic
Mail.getText(Mail.java:13) > private 

nomultiplereturn
Mail.getText(Mail.java:13)
```

Гхм... С чего бы начать?...

Тут четыре типа ошибок:
- `NullFree` - нужно писать код без использования `null` (
[Почему?](https://www.yegor256.com/2014/05/13/why-null-is-bad.html)
)
- `AllFinal` - нужно, чтобы везде стояло ключевое слово `final`, чтобы 
поддерживать иммутабельность (
[Как?](https://www.yegor256.com/2016/09/07/gradients-of-immutability.html)  
)
- `AllPublic` - нужно, чтобы все методы и классы были с модификатором доступа 
`public` (
[Почему?](https://www.nikialeksey.com/java/oop/2017/03/31/private-method-should-be-new-class.html)
)
- `NoMultipleReturn` - нужно, чтобы в методах был только один оператор `return` 
(Почему? Потому что много операторов `return` мешают восприятию кода)

Обычно, проще всего решать `AllFinal` ошибки. Это места, которые нужно пометить
ключевым словом `final`. С них и начнем.

Хотя вот прямо сейчас нужно сделать отступление. Если вы нормальный программист,
то "Что, блин, происходит?!" уже звучит в вашей голове. Потому что для чего 
нужно пытаться писать программу без `null`-ов - не понятно, да и возможно ли 
это? А все методы `public` - это зачем? Ведь часто нужно сделать метод, 
необходимый только для одного конкретного класса. А также ссылочки там были на 
[этого противоречивого господина](https://www.yegor256.com/testimonials.html). 
Короче, далее будет еще не одно действие, после которого будет начинаться жжение
под копчиком. Поэтому предлагаю считать все происходящее в этой статье 
[мысленным экспериментом](https://en.wikipedia.org/wiki/Thought_experiment).

## AllFinal
Вернемся к `final`. Где там в самом первом месте у нас `final`?
```java
Mail(Mail.java:8) > textIsHtml = false
```
Да, поле класса, которое будет изменено в зависимости от того, какой тип 
текстового контента нашел алгоритм: если `text/html`, то `textIsHtml == true`, 
если `text/[not html]`, то, соответственно, `textIsHtml == false`. Тут надо 
объяснить, для чего это нужно, если вы не парсили почту в своих программах 
ранее. Текстовый контент письма может быть представлен в разных форматах, чаще 
всего это `text/html` и `text/plain`. Почтовый клиент должен выбрать наилучший
формат отображения текстового контента, исходя из особенностей окружения, и,
возможно, спросив пользователя ([RFC 2046](https://tools.ietf.org/html/rfc2046#section-5.1.4)).
Поэтому, [JavaMail FAQ](https://javaee.github.io/javamail/FAQ#mainbody) решила, 
что приоритетнее `text/html`, ну а если все же `html` не был найден, то нужно 
оставить флажок, чтобы разработчик понимал, что это найден не `html` контент.

Очевидно, если просто поставить `private final textIsHtml`, то это не решит 
проблему. И вот тут на помощь приходит концепция объектов. Задекларируем 
следующее поведение:
```java
public interface TextContent {
    String asString();
}
```
И, как вы уже, наверное, догадались, сделаем две реализации этого интерфейса.
Для не `html` текста (скорее всего там будет просто `text/plain`):
```java
public final class SimpleTextContent implements TextContent {

    private final String text;

    public SimpleTextContent(final String text) {
        this.text = text;
    }

    @Override
    public String asString() {
        return text;
    }
}
```
И для `html` текста:
```java
public final class HtmlTextContent implements TextContent {
    
    private final String text;

    public HtmlTextContent(final String text) {
        this.text = text;
    }

    @Override
    public String asString() {
        return text;
    }
}
```

Да, да, да. Я уже чувствую запах летящего помидора: "Да это же две одинаковые 
реализации!". Верно, реализации пока одинаковые. Достаточно того, что у нас есть 
две реализации одного поведения и они соответствуют разным классам объектов.
Просто мы еще не придумали бизнес логику, в которой они различны. Но ничего, 
специально для этого вот бизнес случай: **нужно извлечь текст из email и
сделать его цитирование** (например, для того, чтобы генерировать автоматические 
ответы на входящие письма). Различие в том, что для `text/html` цитирование будет 
выглядеть так:
```html
<blockquote>
  ...
</blockquote>
```
А для `text/plain` так:
```text
> ...
> ...
> ...
```

Не зная других особенностей бизнес логики, можно реализовать данный сценарий 
так:
```java
public interface TextContent {
    ...
    String asQuote();
}
```
```java
public final class HtmlTextContent implements TextContent {

    private final String text;

    ...
    
    @Override
    public String asQuote() {
        final StringBuilder quoteBuilder = new StringBuilder();

        quoteBuilder.append("<blockquote>");
        quoteBuilder.append(text);
        quoteBuilder.append("</blockquote>");

        return quoteBuilder.toString();
    }
}
```
```java
public final class SimpleTextContent implements TextContent {

    private final String text;
    
    ...

    @Override
    public String asQuote() {
        final StringTokenizer textLines = new StringTokenizer(text, "\n");
        final StringBuilder quoteBuilder = new StringBuilder();

        while (textLines.hasMoreTokens()) {
            quoteBuilder.append('>');
            quoteBuilder.append(' ');
            quoteBuilder.append(textLines.nextToken());
            quoteBuilder.append('\n');
        }

        return quoteBuilder.toString();
    }
}

```

Хорошо, хорошо. Как там у нас теперь будет выглядеть метод получения текстового 
контента электронного письма?
### Код без внешнего флажка `textIsHtml`
```java
import javax.mail.MessagingException;
import javax.mail.Multipart;
import javax.mail.Part;
import java.io.IOException;

public final class Mail {

    /**
     * Return the primary text content of the message.
     */
    private TextContent getText(final Part p) throws
        MessagingException, IOException {
        if (p.isMimeType("text/*")) {
            final String s = (String)p.getContent();
            final TextContent content;
            if (p.isMimeType("text/html")) {
                content = new HtmlTextContent(s);
            } else {
                content = new SimpleTextContent(s);
            }
            return content;
        }

        if (p.isMimeType("multipart/alternative")) {
            // prefer html text over plain text
            final Multipart mp = (Multipart)p.getContent();
            TextContent text = null;
            for (int i = 0; i < mp.getCount(); i++) {
                final Part bp = mp.getBodyPart(i);
                if (bp.isMimeType("text/plain")) {
                    if (text == null)
                        text = getText(bp);
                    continue;
                } else if (bp.isMimeType("text/html")) {
                    final TextContent s = getText(bp);
                    if (s != null)
                        return s;
                } else {
                    return getText(bp);
                }
            }
            return text;
        } else if (p.isMimeType("multipart/*")) {
            final Multipart mp = (Multipart)p.getContent();
            for (int i = 0; i < mp.getCount(); i++) {
                final TextContent s = getText(mp.getBodyPart(i));
                if (s != null)
                    return s;
            }
        }

        return null;
    }
}
```
Кстати, я дополнительно расставил `final`-ы везде, где это было возможно, и у нас
осталось всего три места, где нужно поставить `final`, но пока нельзя:
```java
TextContent text = null;
...
for (int i = 0; i < mp.getCount(); i++)
...
for (int i = 0; i < mp.getCount(); i++)
```
"В `for`-иках то зачем `final`?" - спросите вы. Там объявляется переменная — 
индекс `int i = 0`, и она имеет возможность быть переназначенной, как, 
собственно, и происходит в блоке действия, выполняемого после каждой итерации.
И, я уверяю вас, мы избавимся от этого не`final` поля через некоторое время. Но
пока, перейдем к другим ошибкам. 

## NoMultipleReturn
В нормальной жизни вас это бы не волновало, но
сейчас у нас мысленный эксперимент. Помните, да? Поэтому нужно попробовать 
обойтись одним `return`. Обычно, когда мне приходится решать подобные проблемки,
и переделывать множественный `return` в одинарный, я поступаю следующим образом.
Я выделяю переменную `final TextContent result;`, затем вместо каждого 
`return value;` делаю `result = value;`:
```java
private TextContent getText(final Part p) {
    final TextContent result;
    if (p.isMimeType("text/*")) {
        final String s = (String)p.getContent();
        if (p.isMimeType("text/html")) {
            result = new HtmlTextContent(s);
        } else {
            result = new SimpleTextContent(s);
        }
    } else if (p.isMimeType("multipart/alternative")) {
        ...
    } else if (p.isMimeType("multipart/*")) {
        ...
    } else {
        result = null;
    }
    return result;
}
```

Сложные случаи с циклами сейчас разберем отдельно. Возьмем из двух сложных 
циклов тот, что попроще и на нем научимся делать один `return`, да еще и 
`final` декларацию не сломать.
```java
} else if (p.isMimeType("multipart/*")) {
    final Multipart mp = (Multipart)p.getContent();
    for (int i = 0; i < mp.getCount(); i++) {
        final TextContent s = getText(mp.getBodyPart(i));
        if (s != null)
            return s;
    }
}
```
Что тут происходит? Итерируются по частям `multipart`-а, и первый не пустой 
ответ будет являться результатом. Достаточно ввести еще один список — список
непустого контента, найденного в частях `multipart`-а и проблема будет решена:
```java
} else if (p.isMimeType("multipart/*")) {
    final Multipart mp = (Multipart)p.getContent();
    final List<TextContent> foundTexts = new ArrayList<>();
    for (int i = 0; i < mp.getCount(); i++) {
        final TextContent s = getText(mp.getBodyPart(i));
        if (s != null) {
            foundTexts.add(s);
        }
    }
    if (!foundTexts.isEmpty()) {
        result = foundTexts.get(0);
    } else {
        result = null;
    }
}
```
### Итоговый результат с одним `return`:
```java
import javax.mail.MessagingException;
import javax.mail.Multipart;
import javax.mail.Part;
import java.io.IOException;
import java.util.ArrayList;
import java.util.List;

public final class Mail {

    /**
     * Return the primary text content of the message.
     */
    private TextContent getText(final Part p) throws
        MessagingException, IOException {
        final TextContent result;
        if (p.isMimeType("text/*")) {
            final String s = (String)p.getContent();
            if (p.isMimeType("text/html")) {
                result = new HtmlTextContent(s);
            } else {
                result = new SimpleTextContent(s);
            }
        } else if (p.isMimeType("multipart/alternative")) {
            // prefer html text over plain text
            final Multipart mp = (Multipart)p.getContent();
            final List<TextContent> foundPlain = new ArrayList<>();
            final List<TextContent> foundHtml = new ArrayList<>();
            final List<TextContent> foundOthers = new ArrayList<>();
            for (int i = 0; i < mp.getCount(); i++) {
                final Part bp = mp.getBodyPart(i);
                final TextContent foundText = getText(bp);
                if (bp.isMimeType("text/plain")) {
                    foundPlain.add(foundText);
                } else if (bp.isMimeType("text/html")) {
                    foundHtml.add(foundText);
                } else {
                    if (foundText != null) {
                        foundOthers.add(foundText);
                    }
                }
            }

            if (!foundHtml.isEmpty()) {
                result = foundHtml.get(0);
            } else if (!foundOthers.isEmpty()) {
                result = foundOthers.get(0);
            } else if (!foundPlain.isEmpty()) {
                result = foundPlain.get(0);
            } else {
                result = null;
            }
        } else if (p.isMimeType("multipart/*")) {
            final Multipart mp = (Multipart)p.getContent();
            final List<TextContent> foundTexts = new ArrayList<>();
            for (int i = 0; i < mp.getCount(); i++) {
                final TextContent s = getText(mp.getBodyPart(i));
                if (s != null) {
                    foundTexts.add(s);
                }
            }
            if (!foundTexts.isEmpty()) {
                result = foundTexts.get(0);
            } else {
                result = null;
            }
        } else {
            result = null;
        }

        return result;
    }
}
```
И текущие ошибки анализатора:
```java
nullfree
Mail.getText(Mail.java:37) > null
Mail.getText(Mail.java:50) > null
Mail.getText(Mail.java:57) > null
Mail.getText(Mail.java:64) > null
Mail.getText(Mail.java:67) > null

allfinal
Mail.getText(Mail.java:29) > int i = 0
Mail.getText(Mail.java:55) > int i = 0

allpublic
Mail.getText(Mail.java:13) > private 
```

## NullFree
Так так. `null` используется в этом коде для обозначения того, что текстовый 
контент не найден. Обычно, в таких ситуациях есть несколько вариантов:
- бросать исключение, если текстовый контент не найден (тогда прийдется 
  отлавливать его вместо проверки на `null`)
- 