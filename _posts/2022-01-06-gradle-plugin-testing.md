---
layout: post
title: "Как тестировать gradle-плагины"
lang: ru
image: /assets/imgs/gradle-plugin-testing/title.png
description: "Про разнообразное автоматическое тестирование gradle-плагинов"
tags:
- testing
- gradle-plugin
---

Когда я писал свой первый gradle-плагин, я проверял его работоспособность 
следующим образом:
1. Опубликовал версию `n` в [plugins.gradle.org][plugins-gradle]
2. Проверил опубликованный плагин вручную на тестовом проекте
3. Нашел ошибку/доработал, увеличил версию `n=n+1`, затем снова пункт 1

Такой вот PDD (Publish Driven Development). Сегодня поговорим о том, как писать
эффективные тесты на собственные gradle плагины.

![Drake Yes/No meme]({{ site.url }}{{ page.image }})
<!--more-->

## Сложности тестирования
Почему в начале я тестировал gradle-плагин именно через публикацию?
Потому что я не знал, как тестировать иначе. Я не знал, как опубликовать 
gradle-плагин в [maven local repository][local-repositories-cases]. Я не знал,
как написать unit-тест для gradle-плагина. Поэтому я пошел самым легким путем. 
Тем, что наверняка сработает. Это, кстати, одна из причин, почему люди не пишут
unit-тесты: они не знают, как это делать. В случае с gradle-плагином сложностей
добавляет тот факт, что gradle - не самая простая система сборки. Обычно на вход подается 
некоторая часть build-скрипта, а на выходе какое-то действие, усложняющее работу
build-скрипта. Повторить и проверить то же самое в unit-тестах сходу сложно.

## Берешь и пишешь тест
Хорош ныть. Берешь и пишешь, чего сложного то. Какая там сигнатура у основного 
метода плагина?
```java
@Override
public void apply(final Project project) {
```
То есть нужно просто подготовить объект `project`, передать его в метод `apply`, 
затем проверить, все ли верно отработало в этом `apply`. Тогда тестовый метод 
будет выглядеть следующим образом:
```java
@Test
public void someSimpleTest() {
  Project project = Mockito.spy(Project.class);    
  Mockito.when(project.getExtensions()).thenReturn(...);
  Mockito.when(project. ...).thenReturn(...);
  ...
    
  new MyPlugin().apply(project);
    
  Mockito.verify(project, Mockito.times(1)).getExtensions();
  Mockito.verify(project). ... ;
  ...
}
```
Вот и все. Можно написать еще парочку подобных и нет проблем, да? Но проблемы есть. Тут 
используются [моки][mocks]. Такой тест будет всегда падать, если реализацию метода `apply`
менять без изменения поведения плагина. [Подробнее о том, почему тестирование с 
использованием моков - это плохо.][mocks-are-bad]

## Специальная подготовка проекта
Кто немножечко углублялся в тему тестирования gradle-плагинов, наверняка 
сталкивался с классом [`org.gradle.testfixtures.ProjectBuilder`][ProjectBuilder-doc].
Он позволяет подготовить `project` более естественным для плагина путем. Без 
тонких знаний реализации метода `apply` вашего плагина. В самом примитивном 
варианте можно сделать вот так:
```java
Project project = ProjectBuilder.builder().build()
```
и этого будет достаточно, чтобы дальше применять ваш плагин и проверять, как он 
отработал. Можно будет через полученный экземпляр `project` проверить, были 
ли созданы ожидаемые `Task`-и или `Extension`-ы, например. Или даже, если ваш
плагин зависит от `project.afterEvaluate(...)`, то можно будет [запустить 
`evaluate` проекта прямо в тесте с помощью][afterEvaluate-in-tests]:
```java
project.evaluationDependsOn(":");
```
Такой подход позволяет более гибко и независимо от реализации `apply` 
настроить проект перед проверкой вашего плагина. [Простой пример][tests-with-project-builder]:
```kotlin
@Test
fun configurePluginWithEmptyExtension() {
  val projectFolder = tmpFolder.newFolder()
  val project = ProjectBuilder.builder()
    .withProjectDir(projectFolder)
    .build()
  project.pluginManager.apply("com.nikialeksey.arspell")
  project.evaluationDependsOn(":")

  MatcherAssert.assertThat(
    project.extensions.getByType(ArspellExtension::class.java),
    IsNull.notNullValue()
  )
  MatcherAssert.assertThat(
    project.tasks.getByName("arspell"),
    IsNull.notNullValue()
  )
}
```

## Проверки из реальной жизни
После написания unit-тестов на gradle-плагин все равно остается соблазн 
проверить плагин в работе вручную "на всякий" сразу после публикации. Потому что
часто вместе с плагином идет какой-нибудь хитрый DSL (Domain Specific Language) 
у его Extensions, который с одной 
стороны нужно правильно реализовать, и с другой у него есть несколько способов 
быть вызванным. Кроме того ваш плагин может быть использован в gradle-скриптах написанных 
на разных языках: kotlin или groovy. В groovy может быть использованы различные
способы настройки `extension`-ов:
```groovy
myPlugin {
  myArgument = "myValue"
}
```
или так:
```groovy
myPlugin.myArgument = "myValue"
```
или даже так (это, конечно, зависит от реализации `extension`-а):
```groovy
myPlugin {
  setMyArgument("myValue")
}
```
Все это должно работать (по крайней мере то, что вы ожидаете). Проверить 
такое в вышеупомянутых unit-тестах не получится. Но автоматизировать подобные 
проверки очень хочется. Не проверять же, в конце концов, каждый раз это руками. 

## Composite builds
На помощь к нам приходят [composite builds][composite-builds]. Они позволяют 
использовать локально один gradle-проект в другом gradle-проекте без публикации 
в какие-либо репозитории. Поэтому предлагаю следующее: под каждую проверку 
плагина из "реальной жизни" (в скрипте kotlin, в скрипте на groovy, в скрипте с 
разным использованием одного и того же `extension`) подготовить отдельный 
gradle-проект, и в каждый из этих тестовых gradle-проектов включить проект, 
содержащий наш тестируемый плагин. Вот так это будет выглядеть в файловой 
системе:
```html
- myPlugin/ <!-- Our plugin main module -->
- myPlugin-module-1/ <!-- Our plugin additional module -->
- myPlugin-module-2/ <!-- Our plugin additional module -->
- myPlugin-tests/ <!-- The plugin integration tests -->
  - kts/ <!-- For kts scripts -->
    - build.gradle.kts
    - settings.gradle.kts <!-- Here we should include our plugin -->
  - groovy/ <!-- For groovy scripts -->
    - build.gradle
    - settings.gradle <!-- Here we should include our plugin -->
- build.gradle
- settings.gradle    
```
Файлы `./myPlugin-tests/kts/settings.gradle.kts`, 
`./myPlugin-tests/groovy/settings.gradle` выглядят так:
```groovy
includeBuild("../../") // include main plugin module
```
Теперь достаточно проверить, что `build` проектов `./myPlugin-tests/kts` и
`./myPlugin-tests/groovy` прошел успешно, или то, 
что запускается какая-нибудь задача вашего плагина. Это будет означать, что 
скрипты с различным использованием `extension`-ов вашего плагина хотя бы компилируются, и,
вероятно, передают необходимые входные данные в плагин. Такие проверки можно написать в 
виде `bash`-скриптов, но я выбрал более простой путь - `.github/workflows` (тут
будет пример из реального проекта, поэтому [вот его исходник][arspell-workflows]):
```yml
jobs:
  build:
    steps:
      - name: Build with Gradle
        run: ./gradlew check
      - name: Check arspell plugin usage with groovy
        working-directory: ./arspell-plugin-gradle-test/groovy/
        run: ./gradlew arspell
      - name: Check arspell plugin usage with kts
        working-directory: ./arspell-plugin-gradle-test/kts/
        run: ./gradlew arspell
```
Параметр [`working-directory`][workflows-working-directory-doc] помог выполнить 
нужные проверки без лишних сложностей с [`cd`][cd-command].

Пожалуй, на этом можно закончить. Описанные подходы по тестированию gradle 
плагинов я успешно применяю в тестировании своих небольших проектах, исходники
которых вы можете найти на GitHub: 
[arspell][arspell-github], [porflavor][porflavor-github]. 

[plugins-gradle]: https://plugins.gradle.org
[local-repositories-cases]: https://docs.gradle.org/7.3.3/userguide/declaring_repositories.html#sec:case-for-maven-local
[mocks]: https://en.wikipedia.org/wiki/Mock_object
[mocks-are-bad]: {{ site.url }}/2019/04/12/mocks-are-bad.html
[ProjectBuilder-doc]: https://docs.gradle.org/7.3.3/javadoc/org/gradle/testfixtures/ProjectBuilder.html
[afterEvaluate-in-tests]: https://stackoverflow.com/a/70558639/3422245
[tests-with-project-builder]: https://github.com/nikialeksey/arspell/blob/890664124ddf372c154c1a15cd4ce9a5d0478664/arspell-plugin-gradle/src/test/java/com/nikialeksey/arspell/ArspellPluginTest.kt#L18-L35
[composite-builds]: https://docs.gradle.org/7.3.3/userguide/composite_builds.html
[arspell-workflows]: https://github.com/nikialeksey/arspell/blob/890664124ddf372c154c1a15cd4ce9a5d0478664/.github/workflows/ci.yml
[workflows-working-directory-doc]: https://docs.github.com/en/actions/learn-github-actions/workflow-syntax-for-github-actions#defaultsrun
[arspell-github]: https://github.com/nikialeksey/arspell
[porflavor-github]: https://github.com/nikialeksey/porflavor
[cd-command]: https://www.man7.org/linux/man-pages/man1/cd.1p.html