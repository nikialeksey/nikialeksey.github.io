---
layout: post
title: "Piece of code reason"
lang: en
image: /assets/imgs/history-for-selection/git-show-history-for-selection.jpeg
description: "How to quickly find the reason behind a piece of code?"
tags:
  - development
  - git
---

How to quickly find the reason behind a piece of code? 🤔

![Git -> Show History for Selection]({{ site.url }}{{ page.image }})
<!--more-->

In Git, there's a way to view the history for a selected portion of 
code rather than the entire file. This goes beyond simply checking 
the author of the latest changes. While programmers often use "git blame", 
there's a more powerful tool. 💪

[`git log -L`][git-log-l] allows you to examine the change history for a 
specified range of lines within a chosen file. In JetBrains IDEs, 
such a feature is integrated into the Git plugin, and you can 
simply highlight a piece of code, then right-click and select:

Git -> Show History for Selection 😊👍

[git-log-l]: https://git-scm.com/docs/git-log#Documentation/git-log.txt--Lltstartgtltendgtltfilegt
