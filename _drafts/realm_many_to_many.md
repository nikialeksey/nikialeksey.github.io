---
layout: post
title:  "RealmList - это many-to-many"
categories: java mobile realm
realm-many-to-many-doc-url: https://realm.io/docs/java/latest/#many-to-many
---

В [докомуентации](https://realm.io/docs/java/latest/#many-to-many) к
Realm сказано, что [`RealmList`](https://realm.io/docs/java/latest/api/io/realm/RealmList.html)
образует **Many-to-Many** связь. Разберемся, почему это так.

Для начала взглянем на сбивающее с толку утверждение в документации
Realm:
```java
public class Contact extends RealmObject {
    public String name;
    public RealmList<Email> emails;
}

public class Email extends RealmObject {
    public String address;
    public boolean active;
}
```
Этот код является примером реализации связи **Many-to-Many**. Путает
здесь то, что в традиционных реляционных базах данных реализация похожей
связи между сущностями будет иметь отношение **One-To-Many**:
{% plantuml %}
{% include skin.plantuml %}

object Contact {
    id
    name
}

object Email {
    contactId
    number
    address
    active
}

City "1" -{ "*" Apartment

{% endplantuml %}
Только там способ связи другой: у сщности, которая является элементом
списка (`Email`) есть ссылка на владельца этого списка.

