---
layout: post
title:  "Читаемый перенос кода"
image: /assets/imgs/chairs.webp
lang: ru_RU
---

Поговорим о переносе кода, о том, когда код не влезает в одну строчку и нужно как-то его перенести. 
Думаю, не стоит рассказывать, что программист читает код чаще, чем пишет, поэтому читабельные переносы помогут 
быстрее понять происходящее. 

![Chairs by Claudia Meyer]({{ site.url }}/assets/imgs/chairs.webp)

### Читабельные/нечитабельные переносы
Не понятно, что такое "читабельный перенос"/"нечитабельный перенос". Поэтому приведу пример:
```java
class WrappedText {
    public WrappedText(@NonNull final WrappingAlgorithm algorithm, @NonNull final String text) {
        ...
    }
}
```
Потом с течением времени в алгоритм переноса решили добавить гравитацию для текста:
```java
class WrappedText {
    public WrappedText(@NonNull final WrappingAlgorithm algorithm, @NonNull final TextGravity gravity, @NonNull final String text) {
```
Допустим 120 символов в строке - это предельная ширина строки, определенная вашей командой. Сигнатура конструктора 
`WrappedText` превышает 120 символов. Нужно переносить код. Как это сделать? Например, перенести последний аргумент 
на новую строчку с отступом слева таким же, как у первого аргумента:
```java
class WrappedText {
    public WrappedText(@NonNull final WrappingAlgorithm algorithm, @NonNull final TextGravity gravity, 
                       @NonNull final String text) {
```
Ок. Прошло еще немного времени и к алгоритму переноса в конструктор решили добавить символ, используемый для переноса:
```java
class WrappedText {
    public WrappedText(@NonNull final WrappingAlgorithm algorithm, @NonNull final TextGravity gravity, @NonNull final WrappingSymbol wrappingSymbol, 
                       @NonNull final String text) {
```
И снова текст вылез за 120 символов и нужно переносить, например, так:
```java
class WrappedText {
    public WrappedText(@NonNull final WrappingAlgorithm algorithm, @NonNull final TextGravity gravity, 
                       @NonNull final WrappingSymbol wrappingSymbol, @NonNull final String text) {
```
Подобным образом переносы сделают большинство из нас, не так ли?

Чтобы сразу было с чем сравнивать предлагаю и топлю за использования следующего алгоритма переноса:
как только строчка становится большой, то необходимо переносить все параметры конструктора/метода/вызова метода, каждый 
параметр на отдельной строчке, причем первый параметр переносится сразу на следующую строчку со стандартным отступом
слева, закрывающая скобка тоже на новой строчке. Другими словами, когда появилось три параметра у`WrappedText`, то, 
согласно алгоритму, описанному выше, сигнатура конструктора выглядела бы так:
```java
class WrappedText {
    public WrappedText(
        @NonNull final WrappingAlgorithm algorithm, 
        @NonNull final TextGravity gravity, 
        @NonNull final String text
    ) {
```
Четыре параметра выглядели бы так:
```java
class WrappedText {
    public WrappedText(
        @NonNull final WrappingAlgorithm algorithm, 
        @NonNull final TextGravity gravity, 
        @NonNull final WrappingSymbol wrappingSymbol, 
        @NonNull final String text
    ) {
```

Внимание, вопрос: где быстрее сосчитать количество параметров на картинке ниже:
![Внимание, вопрос: где быстрее сосчитать количество параметров?]({{ site.url }}/assets/imgs/code-wrapping-easy-reading.jpg)

### Git diff
Посмотрите, как будет выглядеть `git diff` между тремя и четырьмя параметрами, если вы форматируете код как текст в 
книжке:
![Bad diff]({{ site.url }}/assets/imgs/code-wrapping-diff-bad.jpg)
То есть есть одно удаление и одна вставка.

Вот как `git diff` будет выглядеть, если вы форматируете предложенным способом:
![Good diff]({{ site.url }}/assets/imgs/code-wrapping-diff-good.jpg)
Здесь только одна вставка и ни одного удаления - это выглядит проще и пул реквест будет читаться быстрее с таким 
форматированием.


### Реальный пример
Ок, пример выше надуманный. Вот [реальный](https://github.com/nikialeksey/FullScreenDialog/blob/master/lib/src/androidTest/java/com/nikialeksey/fullscreendialog/DissmissOnCloseDialogTest.java#L40-L45) 
код:
```java
public static class TestActivity extends FsDialogActivity {
    @Override
    public Dialog createDialog() {
        return new DismissOnCloseDialog(new FsDialog(this, R.style.Theme_AppCompat,
            new FsDialogToolbar(this, DIALOG_TITLE,
                new FsCloseButton(new SimpleButton(), new ColorDrawable()),
                new FsActionButton(new SimpleButton(), DIALOG_ACTION)), new FrameLayout(this)));
    }
}
```
А теперь сделаем его читаемым с помощью переносов:
```java
public static class TestActivity extends FsDialogActivity {
    @Override
    public Dialog createDialog() {
        return new DismissOnCloseDialog(
            new FsDialog(
                this, 
                R.style.Theme_AppCompat,
                new FsDialogToolbar(
                    this, 
                    DIALOG_TITLE,
                    new FsCloseButton(new SimpleButton(), new ColorDrawable()),
                    new FsActionButton(new SimpleButton(), DIALOG_ACTION)
                ), 
                new FrameLayout(this)
            )
        );
    }
}
```

### Автоматизируйте переносы
Ок, вы согласились, но у вас совсем нет времени делать такие переносы вручную. Например, чтобы настроить автоматическое
форматирование в `Intellij Idea` при переносе параметров в сигнатуре метода нужно перейти в
`Settings -> Editor -> Code Style -> Java -> Wrapping And Braces`:

![Method declaration parameters]({{ site.url }}/assets/imgs/code-wrapping-settings.jpg)

---
Переносы в коде не стоит игнорировать. Правильно оформленный код быстрее поддается форматированию и проще читается, 
тогда как код, оформленный как текст в книжке, читать и понимать намного сложнее. 