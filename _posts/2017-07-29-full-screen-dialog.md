---
layout: post
title:  "Android Full-Screen Dialog Library"
lang: ru_RU
---

Существует [компонент](https://material.io/guidelines/components/dialogs.html#dialogs-full-screen-dialogs)
в [Material Design](https://material.io/guidelines/material-design/introduction.html), 
который предназначен для отображения диалога во весь экран вашего приложения. Выглядит это так:  
![True full-screen dialog]({{ site.url }}/assets/imgs/components-dialogs-fullscreen.jpg)

К сожалению, в [стандартной библиотеке дизайна](https://developer.android.com/training/material/design-library.html)
не реализован такой компонент. Поэтому предлагаю вашему вниманию 
[Full-Screen Dialog Library](https://github.com/nikialeksey/FullScreenDialog).

Давайте посмотрим, как она работает: 

![Full-Screen Dialog]({{ site.url }}/assets/gifs/full-screen-dialog.gif)

Для того, чтобы добавить такую возможность себе в проект, необходимо в вашем 
`build.gradle` добавить зависимость:
```groovy
repositories {
    jcenter()
}

dependencies {
    compile('com.nikialeksey:fullscreendialog:<latest version>@aar') {
        transitive true
    }
}
```

Далее нужно создать и открыть дилог:
```java
new FsDialog(context, R.style.AppTheme, "Title", new FsDialogCloseAction() {
    @Override
    public void onClose(@NonNull final FsDialog dialog) {
// close action
    }
}, "Action Title", new FsDialogAction() {
    @Override
    public void onAction(@NonNull final FsDialog dialog) {
//  base action        
    }
}, contentView).show();
```

, где: 
 - `R.style.AppTheme` - тема, из нее берется цвет основного текста для 
 заголовка диалога, и кнопок действий, основной цвет для `Toolbar`
 - `"Title"` - это текст заголовка диалога
 - `"Action Title"` - это текст положительного действия диалога
 - `contentView`- это вьюха контента диалога

Важно, что `FsDialog` наследуется от `AppCompatDialog`, но, например, изменить заголовок 
методом `setTitle` не удастся. Предполагается, что объект `FsDialog` иммутабельный. 

В компоненте `full-screen dialog` описано, что необходимо при закрытии диалога 
показывать предупреждение, если пользователь успел изменить данные в контенте. В следующих 
версиях такая возможность будет добавлена.
