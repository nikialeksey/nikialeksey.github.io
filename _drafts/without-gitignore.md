---
layout: post
title:  "Жизнь без .gitignore"
---

В [статье](https://hackernoon.com/exclude-files-from-git-without-committing-changes-to-gitignore-986fa712e78d)
рассказывается, что файл `.gitignore` - мусорка, которая может еще и навредить. Например, вы положили туда паттерн,
который избавляет вас от ваших специфичных файлов, которые не должны попасть в 
[VSC](https://en.wikipedia.org/wiki/Version_control), а ваш коллега получил эту информацию, и узнал, 
например, какую вы, используете [IDE](https://en.wikipedia.org/wiki/IDE). В той статье так же описано,
как жить без `.gitignore` файла с помощью 
[`.git/info/exclude`](https://help.github.com/articles/ignoring-files/#explicit-repository-excludes). Но если вам 
не хочется залазить и что-то править в `.git` и вы используете IDE от JetBrains, то добро пожаловать.

   