---
layout: post
title:  "RealmList - это many-to-many"
categories: java mobile realm
realm-many-to-many-doc-url: https://realm.io/docs/java/latest/#many-to-many
---

В [докомуентации](https://realm.io/docs/java/latest/#many-to-many) к
Realm сказано, что [`RealmList`](https://realm.io/docs/java/latest/api/io/realm/RealmList.html)
образует **Many-To-Many** связь. Разберемся, почему это так.

Для начала взглянем на утверждение в документации Realm:
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
Этот код является примером реализации связи **Many-To-Many**:
{% plantuml %}
{% include skin.plantuml %}

object Contact {
    name
}

object Email {
    address
    active
}

Contact "*" }-{ "*" Email

{% endplantuml %}
То есть в Realm возможно хранить один элемент списка у разных владельцев.

TODO
Нужно потестить на скорость RealmList и One-To-Many