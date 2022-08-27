---
layout: post
title: Please, don't switch
tags: [code smell, switch statement]
published: false
---

_Switch statement_ is a code smell. That's the article's openening statement and, frankly speaking, might be the last. But bringing up a problem right away is a great addition to another subject, one of the most well-known design patterns, the factory method. Please note: I won't go into details. If you want to know how to apply the pattern, read another guide - for Ruby users, I highly endorse Russ Olsen's book "Ruby Design Patterns," which is probably well-known among Ruby programmers worldwide. Here, I'd like to share some thoughts about the pattern only.

### The _switch_ issue

The switch isn't polymorphic and should be avoided if a code follows OOP guidelines strictly. Your software may have an architecture bug if it frequently leads to the code. Years ago, I was confused to see that `react-redux` by default recommends using it to create a reducer. Switch? Why? Well, well, some may argue reducers prefer functional than object oriented way and the displayed “no-more-switch rule“ is pointless in such comparison. Anyways I remember times when migrating from _AngularJS_ to _React_ and we did look for the better style of coding reducers than using switch. There was the _switch_ at the end anyways, following official docs. Honestly, it still worries me when I look at the docs, and hopefully I'm not alone in feeling this way.

...but I don't code reducers any longer and my concerns flies away. The lack of polymorphism worries me all the time.