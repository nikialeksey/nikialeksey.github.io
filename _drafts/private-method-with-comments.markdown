---
layout: post
title:  "Приватный метод с комментарием - повод для нового класса"
categories: java oop
---

Следующий код требует рефакторинга:
```java
public class MailingServiceImpl implements MailingService {

    @Override
    public sendMail() { ... }

    /**
     * Добавление подписи по умолчанию
     * @param message сообщение
     * @return сообщение с подписью
     */
    private String addSign(String message) { ... }
}
```
Конкретно, метод `addSign` нужно вынести в отдельный сервис. Почему?
