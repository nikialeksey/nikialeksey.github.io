---
layout: post
title:  "Жизнь без .gitignore"
lang: ru_RU
---

В [статье](https://hackernoon.com/exclude-files-from-git-without-committing-changes-to-gitignore-986fa712e78d)
рассказывается, что файл `.gitignore` - мусорка, которая может еще и навредить. Например, вы положили туда паттерн,
который избавляет вас от ваших специфичных файлов, которые не должны попасть в 
[VSC](https://en.wikipedia.org/wiki/Version_control), а ваш коллега получил эту информацию, и узнал, 
например, какую вы, используете [IDE](https://en.wikipedia.org/wiki/IDE). В той статье так же описано,
как жить без `.gitignore` файла с помощью 
[`.git/info/exclude`](https://help.github.com/articles/ignoring-files/#explicit-repository-excludes). Но если вам 
не хочется что-то править в `.git` и вы используете IDE от JetBrains, то добро пожаловать.

<!--more-->

Основная идея в том, что вы сохраняете в настройках проекта информацию об игнорируемых для 
[VSC](https://en.wikipedia.org/wiki/Version_control) файлах. Вообще есть 
[мануал](https://www.jetbrains.com/help/idea/configuring-ignored-files.html) про игнорирование файлов,
но это не основной путь когда вы только создали/клонировали проект. [IDE](https://en.wikipedia.org/wiki/IDE)
от JetBrains следит за файлами в проекте, которые не попали в систему контроля и не игнорируются. 
Такие файлы называются `Unversioned Files`:
 
![Unversioned Files]({{ site.url }}/assets/imgs/unversioned-files.png)

Их можно найти на вкладке `Version Control` -> `Local Changes`. И прямо там же их можно добавить в игнорирование:

![Ignore unversioned files]({{ site.url }}/assets/imgs/ignore-unversioned-files.png)

Далее нужно будет выбрать паттерн для игнорирования: игнорировать ли все файлы в папке, 
игнорировать ли файлы с расширением, или игнорировать конкретный файл. И все. Больше никакого `.gitignore`, 
[IDE](https://en.wikipedia.org/wiki/IDE) сама будет следить за тем, чтобы игнорированные ей файлы не попали
в систему контроля версий. Да еще и подсвечивает желтым те файлы, которые игнорирует:

![Ignored files]({{ site.url }}/assets/imgs/ignored-files.png)
