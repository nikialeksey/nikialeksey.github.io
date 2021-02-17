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