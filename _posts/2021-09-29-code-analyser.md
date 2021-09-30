---
layout: post
title: "Статический анализатор, который изменит вашу архитектуру"
lang: ru
image: /assets/imgs/tribore.jpeg
description: "Про бескомпромиссный статический анализатор кода"
tags:
- oop
- static analysis
---

Статический анализатор обычно помогает [поддерживать выбранный стиль кода](https://ktlint.github.io/). Иногда
он находит [нетривиальные шаблонные проблемы](https://pmd.github.io/latest/pmd_rules_java_design.html#godclass). 
Но сегодня посмотрим на то, как статический анализатор заставляет менять всю 
архитектуру.

![Tribore Menendez from Final Space]({{ site.url }}{{ page.image }})
_Tribore Menendez from [Final Space](https://www.imdb.com/title/tt6317068/)_

<!--more-->

Я решал задачу получения текстового контента из объекта, представляющее 
электронное письмо. Оказывается, задача непростая, ведь тело письма 
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
и запустил статический анализатор [youshallnotpass](https://youshallnotpass.dev/):
```java
> Task :youshallnotpass FAILED

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
- `NullFree` - нужно писать код без использования `null`{:.language-java}{:.highlight} ([Почему?](https://www.yegor256.com/2014/05/13/why-null-is-bad.html))
- `AllFinal` - нужно, чтобы везде стояло ключевое слово `final`{:.language-java}{:.highlight}, чтобы 
поддерживать иммутабельность ([Как?](https://www.yegor256.com/2016/09/07/gradients-of-immutability.html))
- `AllPublic` - нужно, чтобы все методы и классы были с модификатором доступа 
`public`{:.language-java}{:.highlight} ([Почему?](https://www.nikialeksey.com/java/oop/2017/03/31/private-method-should-be-new-class.html))
- `NoMultipleReturn` - нужно, чтобы в методах был только один оператор `return`{:.language-java}{:.highlight} 
(Почему? Потому что много операторов `return`{:.language-java}{:.highlight} мешают восприятию кода метода)

Ошибки статического анализатора нужно исправлять, иначе сборка в CI не соберется.
Ошибок достаточно много, поэтому придется исправлять поэтапно. Обычно, проще 
всего решать `AllFinal` ошибки. Это места, которые нужно пометить ключевым 
словом `final`{:.language-java}{:.highlight}. С них и начнем.

Хотя вот прямо сейчас нужно сделать отступление. Если вы нормальный программист,
то "Что, блин, происходит?!" уже звучит в вашей голове. Потому что для чего 
нужно пытаться писать программу без `null`{:.language-java}{:.highlight}-ов - не понятно, да и возможно ли 
это? А все методы `public`{:.language-java}{:.highlight} - это зачем? Ведь часто нужно сделать метод, 
необходимый только для одного конкретного класса. А также ссылочки там были на 
[этого противоречивого господина](https://www.yegor256.com/testimonials.html). 
Короче, далее будет еще не одно действие, после которого будет начинаться жжение
под копчиком. Поэтому предлагаю считать все происходящее в этой статье 
[мысленным экспериментом](https://en.wikipedia.org/wiki/Thought_experiment).

## AllFinal 1-ый этап
Вернемся к `final`{:.language-java}{:.highlight}. Где там в самом первом месте 
у нас `final`{:.language-java}{:.highlight}?
```java
Mail(Mail.java:8) > textIsHtml = false
```
Да, поле класса, которое будет изменено в зависимости от того, какой тип 
текстового контента нашел алгоритм: если `text/html`, то `textIsHtml == true`{:.language-java}{:.highlight}, 
если `text/[not html]`, то, соответственно, `textIsHtml == false`{:.language-java}{:.highlight}. Тут надо 
объяснить, для чего это нужно, если вы не парсили почту в своих программах 
ранее. Текстовый контент письма может быть представлен в разных форматах, чаще 
всего это `text/html` и `text/plain`. Почтовый клиент должен выбрать наилучший
формат отображения текстового контента, исходя из особенностей окружения, и,
возможно, спросив пользователя ([RFC 2046](https://tools.ietf.org/html/rfc2046#section-5.1.4)).
Поэтому, [JakartaMail FAQ](https://javaee.github.io/javamail/FAQ#mainbody) решила, 
что приоритетнее `text/html`, ну а если все же `html` не был найден, то нужно 
оставить флажок, чтобы разработчик понимал, что это найден не `html` контент.

Очевидно, если просто поставить `private final textIsHtml`{:.language-java}{:.highlight}, 
то это не решит проблему. И вот тут на помощь приходит [концепция объектов](https://en.wikipedia.org/wiki/Object-oriented_programming). 
Задекларируем следующее поведение:
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
реализации!". Верно, реализации *пока* одинаковые. Достаточно того, что у нас есть 
две реализации одного поведения и они соответствуют разным классам объектов.
Просто мы еще не придумали бизнес логику, в которой они различны. Но ничего, 
специально для этого вот вам бизнес случай: **нужно извлечь текст из email и
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
<details>
  <summary>Посмотреть</summary>
{% highlight java %}
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
{% endhighlight %}
</details>
Кстати, я дополнительно расставил `final`{:.language-java}{:.highlight}-ы везде, где это было возможно, и у нас
осталось всего три места, где нужно поставить `final`{:.language-java}{:.highlight}, но пока нельзя:
```java
TextContent text = null;
...
for (int i = 0; i < mp.getCount(); i++)
...
for (int i = 0; i < mp.getCount(); i++)
```
"В `for`{:.language-java}{:.highlight}-иках то зачем `final`{:.language-java}{:.highlight}?" - спросите вы. 
Там объявляется переменная—индекс `int i = 0`{:.language-java}{:.highlight}, и она имеет возможность быть переназначенной, как, 
собственно, и происходит в блоке действия, выполняемого после каждой итерации.
И, я уверяю вас, мы сможем избавиться от этого не `final`{:.language-java}{:.highlight} поля через некоторое время. Но
пока, перейдем к другим ошибкам. 

## NoMultipleReturn
В нормальной жизни вас это бы не волновало, но
сейчас у нас мысленный эксперимент. Помните, да? Поэтому нужно попробовать 
обойтись одним `return`{:.language-java}{:.highlight}. Обычно, когда мне приходится решать подобные проблемки,
и переделывать множественный `return`{:.language-java}{:.highlight} в одинарный, я поступаю следующим образом.
Я выделяю переменную `final TextContent result;`{:.language-java}{:.highlight}, затем вместо каждого 
`return value;`{:.language-java}{:.highlight} делаю `result = value;`{:.language-java}{:.highlight}:
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
циклов тот, что попроще, и на нем научимся делать один `return`{:.language-java}{:.highlight}, да еще и 
`final`{:.language-java}{:.highlight} декларацию не сломать.
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
### Итоговый результат с одним `return`{:.language-java}{:.highlight}:
<details>
  <summary>Посмотреть</summary>
{% highlight java %}
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
{% endhighlight %}
</details>
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
Так так. `null`{:.language-java}{:.highlight} используется в этом коде для обозначения того, что текстовый 
контент не найден. Обычно, в таких ситуациях есть несколько вариантов:
- бросать исключение, если текстовый контент не найден (тогда прийдется 
отлавливать его вместо проверки на `null`{:.language-java}{:.highlight})
- сделать еще одну реализацию `interface TextContent`{:.language-java}{:.highlight} для обозначения отсутствия
текста
- можно попробовать приравнять отсутствие текста к пустой строке

Вот третьим путем мы и пойдем, так как ограничения нашей несуществующей бизнес 
логики говорят, что, в случае отсутствия текстового контента в письме, можно 
считать, что письмо просто состоит из пустой строки. Это значит, что все 
присваивания `null`{:.language-java}{:.highlight}-ов можно заменить на `new SimpleTextContent("")`{:.language-java}{:.highlight}, а все 
проверки на `null`{:.language-java}{:.highlight} можно заменить на 
`textContent.isEmpty()`{:.language-java}{:.highlight} (этого метода там еще нет,
но ввести его не сложно, так как необходимая информация уже инкапсулирована в 
классы `SimpleTextContent`, `HtmlTextContent`).
Довольно просто!

### Код без `null`-ов
<details>
  <summary>Посмотреть</summary>
{% highlight java %}
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
                    if (!foundText.isEmpty()) {
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
                result = new SimpleTextContent("");
            }
        } else if (p.isMimeType("multipart/*")) {
            final Multipart mp = (Multipart)p.getContent();
            final List<TextContent> foundTexts = new ArrayList<>();
            for (int i = 0; i < mp.getCount(); i++) {
                final TextContent s = getText(mp.getBodyPart(i));
                if (!s.isEmpty()) {
                    foundTexts.add(s);
                }
            }
            if (!foundTexts.isEmpty()) {
                result = foundTexts.get(0);
            } else {
                result = new SimpleTextContent("");
            }
        } else {
            result = new SimpleTextContent("");
        }

        return result;
    }
}
{% endhighlight %}
</details>
Однако ошибки анализатора все еще присутствуют:
```java
allfinal
Mail.getText(Mail.java:29) > int i = 0
Mail.getText(Mail.java:55) > int i = 0

allpublic
Mail.getText(Mail.java:13) > private 
```

## AllFinal 2-ой этап
`for (final int i = 0; i < mp.getCount(); i++) {`{:.language-java}{:.highlight} - вот так, как вы понимаете,
сделать нельзя. Вот если бы можно было проитерироваться по элементам 
`multipart` 😌:
```java
for (final Part part: mp) {
    ...    
}
```
Однако, [`Multipart`](https://jakarta.ee/specifications/platform/8/apidocs/javax/mail/multipart)
не реализует [`Iterable`](https://docs.oracle.com/javase/8/docs/api/java/lang/Iterable.html).

Что ж, не беда, сделаем свою обертку! Для начала, нужно реализовать 
[`Iterator`](https://docs.oracle.com/javase/8/docs/api/java/util/Iterator.html):
```java
import javax.mail.MessagingException;
import javax.mail.Multipart;
import javax.mail.Part;
import java.util.Iterator;
import java.util.NoSuchElementException;
import java.util.concurrent.atomic.AtomicInteger;

public final class PartsIterator implements Iterator<Part> {

    private final Multipart multipart;
    private final AtomicInteger position;

    public PartsIterator(
        final Multipart multipart
    ) {
        this(multipart, new AtomicInteger(0));
    }

    public PartsIterator(
        final Multipart multipart,
        final AtomicInteger position
    ) {
        this.multipart = multipart;
        this.position = position;
    }


    @Override
    public boolean hasNext() {
        try {
            return position.intValue() < multipart.getCount();
        } catch (final MessagingException e) {
            throw new IllegalStateException(e);
        }
    }

    @Override
    public Part next() {
        if (!this.hasNext()) {
            throw new NoSuchElementException(
                "The iterator doesn't have any more items"
            );
        }
        try {
            return multipart.getBodyPart(position.getAndIncrement());
        } catch (final MessagingException e) {
            throw new IllegalStateException(e);
        }
    }
}
```

Теперь можно и `Iterable`. Для скорости, воспользуемся библиотекой 
[`cactoos`](https://github.com/yegor256/cactoos), в частности классом 
[`IterableOf`](https://github.com/yegor256/cactoos/blob/0.49/src/main/java/org/cactoos/iterable/IterableOf.java):
```java
new IterableOf<>(new PartsIterator(mp))
```
### Почти конечный вариант c `final` везде
<details>
  <summary>Посмотреть</summary>
{% highlight java %}
import org.cactoos.iterable.IterableOf;

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
            final IterableOf<Part> parts = new IterableOf<>(
                new PartsIterator(mp)
            );
            final List<TextContent> foundPlain = new ArrayList<>();
            final List<TextContent> foundHtml = new ArrayList<>();
            final List<TextContent> foundOthers = new ArrayList<>();
            for (final Part bp: parts) {
                final TextContent foundText = getText(bp);
                if (bp.isMimeType("text/plain")) {
                    foundPlain.add(foundText);
                } else if (bp.isMimeType("text/html")) {
                    foundHtml.add(foundText);
                } else {
                    if (!foundText.asString().isEmpty()) {
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
                result = new SimpleTextContent("");
            }
        } else if (p.isMimeType("multipart/*")) {
            final Multipart mp = (Multipart)p.getContent();
            final IterableOf<Part> parts = new IterableOf<>(
                new PartsIterator(mp)
            );
            final List<TextContent> foundTexts = new ArrayList<>();
            for (final Part bp: parts) {
                final TextContent s = getText(bp);
                if (!s.asString().isEmpty()) {
                    foundTexts.add(s);
                }
            }
            if (!foundTexts.isEmpty()) {
                result = foundTexts.get(0);
            } else {
                result = new SimpleTextContent("");
            }
        } else {
            result = new SimpleTextContent("");
        }

        return result;
    }
}
{% endhighlight %}
</details>
Осталась одна ошибка анализатора: 
```java
allpublic
Mail.getText(Mail.java:15) > private 
```

## AllPublic

Мысленный эксперимент продолжается, и нам нужно, казалось бы, просто взять и 
сделать этот метод `public`, чего сложного то? Сложно тут то, что если просто 
сделать этот метод публичным, то обновится поведение у класса `Mail`, владеющего
этим методом. Поэтому, как объяснялось в 
[статье](https://www.nikialeksey.com/java/oop/2017/03/31/private-method-should-be-new-class.html):

> Приватный метод - повод для нового класса

Логика в методе `Mail.getText` получилась непростой, она однозначно требует 
тестирования. Скорее всего, она может быть переиспользована в других участках 
кода. Вынести этот код в новый класс - значит решить целых три проблемы:
- анализатор больше не будет ругаться
- получение текстового контента можно будет тестировать
- получение текстового контента можно будет переипользовать

Остается только выделить поведение такого класса, задеклалировать интерфейс,
и реализовать его. Интерфейс наверняка должен быть простым, от интерфейса нам 
требуется только метод, возвращающий `TextContent`. Или можно воспользоваться 
поведением существующего `TextContent`, написав реализацию, которая будет 
принимать `Part` в конструкторе.

## Конечный вариант без ошибок анализатора

<details>
  <summary>Посмотреть</summary>
{% highlight java %}
public final class PartTextContent implements TextContent {

    private final Unchecked<TextContent> text;

    public PartTextContent(final Part p) {
        this(new Unchecked<>(() -> {
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
                final IterableOf<Part> parts = new IterableOf<>(
                    new PartsIterator(mp)
                );
                final List<TextContent> foundPlain = new ArrayList<>();
                final List<TextContent> foundHtml = new ArrayList<>();
                final List<TextContent> foundOthers = new ArrayList<>();
                for (final Part bp: parts) {
                    final TextContent foundText = new PartTextContent(bp);
                    if (bp.isMimeType("text/plain")) {
                        foundPlain.add(foundText);
                    } else if (bp.isMimeType("text/html")) {
                        foundHtml.add(foundText);
                    } else {
                        if (!foundText.asString().isEmpty()) {
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
                    result = new SimpleTextContent("");
                }
            } else if (p.isMimeType("multipart/*")) {
                final Multipart mp = (Multipart)p.getContent();
                final IterableOf<Part> parts = new IterableOf<>(
                    new PartsIterator(mp)
                );
                final List<TextContent> foundTexts = new ArrayList<>();
                for (final Part bp: parts) {
                    final TextContent s = new PartTextContent(bp);
                    if (!s.asString().isEmpty()) {
                        foundTexts.add(s);
                    }
                }
                if (!foundTexts.isEmpty()) {
                    result = foundTexts.get(0);
                } else {
                    result = new SimpleTextContent("");
                }
            } else {
                result = new SimpleTextContent("");
            }

            return result;
        }));
    }

    public PartTextContent(final Unchecked<TextContent> text) {
        this.text = text;
    }

    @Override
    public String asString() {
        return text.value().asString();
    }

    @Override
    public String asQuote() {
        return text.value().asQuote();
    }

    @Override
    public boolean isEmpty() {
        return text.value().isEmpty();
    }
}
{% endhighlight %}
</details>

Там довольно много кода, но я выделю главные изменения:
```java
public final class PartTextContent implements TextContent {

    private final Unchecked<TextContent> text;

    public PartTextContent(final Part p) {
        this(new Unchecked<>(() -> {
            final TextContent result;
            if (p.isMimeType("text/*")) {
                ...
            } else if (p.isMimeType("multipart/alternative")) {
                ...
                for (final Part bp: parts) {
                    final TextContent foundText = new PartTextContent(bp);
                    ...
                }

                ...
            } else if (p.isMimeType("multipart/*")) {
                ...
                for (final Part bp: parts) {
                    final TextContent s = new PartTextContent(bp);
                    ...
                }
                ...
            } else {
                ...
            }

            return result;
        }));
    }

    public PartTextContent(final Unchecked<TextContent> text) {
        this.text = text;
    }

    ...
}
```
Видите, да? Там конструктор вызывается в конструкторе. Вы можете подумать, что 
это такой особый вид ректальных неудовольствий (или удовольствий). Потому что 
вызывать конструктор рекурсивно - это уж совсем неоптимально, непроизводительно 
и т.д. Но если немного подумать, то можно понять, что в email почти никогда нет 
вложенных друг в друга `multipart`-ов, поэтому рекурсия не будет глубокой.

Кстати, вам может показаться, что код конструктора получился громоздким (мне тоже так кажется).
Поэтому можно выделить еще пару классов `class MultipartAlternativeTextContent implements TextContent`{:.language-java}{:.highlight} и
`class MultipartTextContent implements TextContent`{:.language-java}{:.highlight}, 
тогда итоговый конструктор `class PartTextContent`{:.language-java}{:.highlight}
будет выглядеть совсем просто:
<details>
  <summary>Посмотреть</summary>
{% highlight java %}
public final class PartTextContent implements TextContent {
    public PartTextContent(final Part p) {
        this(new Unchecked<>(() -> {
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
                final Multipart mp = (Multipart) p.getContent();
                result = new MultipartAlternativeTextContent(mp);
            } else if (p.isMimeType("multipart/*")) {
                final Multipart mp = (Multipart) p.getContent();
                result = new MultipartTextContent(mp);
            } else {
                result = new SimpleTextContent("");
            }

            return result;
        }));
    }
    ...
}
{% endhighlight %}
</details>

---

Мысленный эксперимент закончился. Мы просто попробовали переписать код так, 
чтобы в нем все переменные были `final`{:.language-java}{:.highlight}, чтобы там не использовались `null`{:.language-java}{:.highlight}, чтобы
был только один `return`{:.language-java}{:.highlight} и чтобы все методы были `public`{:.language-java}{:.highlight}. На самом деле в 
анализаторе [`youshallnotpass`](https://youshallnotpass.dev) есть еще несколько 
правил. Например, не использовать getter-ы/setter-ы, не использовать ключевое 
слово `static`{:.language-java}{:.highlight}. И вы, может быть, удивитесь, но
оказывается можно писать код, который соответствует всем этим правилам, и такой 
код даже будет решать бизнес задачи. Это и была основная цель статьи - показать, 
что так тоже можно. Я не буду утверждать, что код, соответствующий таким 
правилам, лучше или хуже того, что мы привыкли видеть каждый день (хотя уже 
есть некоторые исследования на тему 
[использования null](https://ieeexplore.ieee.org/abstract/document/9392959)).
Это был всего лишь мысленный эксперимент, где соответствие некоторым формальным
правилам подхода [ElegantObjects](https://www.elegantobjects.org/) с помощью
автоматического инструмента заставило поменять архитектуру. 