---
title: Agera知识点整理
date: 2016-11-18
tags: [Android,Agera]
---

Agera中的几个重要概念：

* `Obserable` ： Agera中的被观察者，用于在合适的时机去通知观察者进行更新
* `Updatable` ：Agera中的观察者，用于观察Obserable
* `Supplier` ：Agera中提供数据的接口，用于观察Obserable
* `Repository`： Agera中集成了Obserable和Supplier功能的一个提供数据的被观察者



Agera--->Push event,pull data model