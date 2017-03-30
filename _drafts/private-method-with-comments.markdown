---
layout: post
title:  "Приватный метод с комментарием - повод для нового класса"
categories: java oop
---

Следующий код требует рефакторинга:
```java
public class MailingServiceImpl implements MailingService {

    @Override
    public sendMail(Message message) {
        Message signedMessage = addDefaultSign(message);
        ...
    }

    /**
     * Добавление подписи по умолчанию
     * @param message сообщение
     * @return сообщение с подписью
     */
    private Message addDefaultSign(Message message) { ... }
}
```
Конкретно, метод `addDefaultSign` нужно вынести в отдельный сервис.
Почему?

Потому что сервис отправки почты должен заниматься только отправкой
почты и ничем другим. Необходимо сформировать сообщение заранее и
передать его в сервис отправки почты.
