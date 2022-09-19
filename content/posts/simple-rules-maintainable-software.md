---
title: "Some simple rules for maintainable software"
date: 2022-09-19T06:24:58+01:00
draft: false
featured_image: '/images/django.png'
---

Writing good software is an art rather than a science, and often decisions made quickly lead to a lot of pain down the line, because it is difficult to see the impact of them. With that said, here are a few of the rules I try and stick to when writing software in order to keep it maintainable and to allow course correction straightforwardly later.

## 1. Abstract away complexity

The KISS principle is well known among software developers, but it's all too common to see people try and incorporate clever solutions into their code bases. I am certainly not saying that this is never appropriate - if, for e.g., you hit a particularly bad performance problem then it may be the case that you need to switch to a more complex algorithm with better characteristics. However, this complexity should in most cases be abstracted away.

Take sorting for example. In almost every programming language, the complexity of sorting is hidden behind standard library functions. The user does not need to think about the underlying implementation. If you find that these are not sufficient for your use case, then the best thing you can do is to at least copy the interface of the standard methods, so that users will understand how to use it without having a significant cognitive load.

## 2. Use standard design patterns to solve known problems - and know when not to use them

The software industry is well known for being an industry in which the barrier to entry can be quite low - someone working alone in their bedroom can get to a point of being employable in under year. With that said, 

## 3. Know when to split code up to improve comprehensibility

I'm constantly surprised by the reluctance of many developers to split their code up either into independent modules or even into seperate files and functions. To me, large modules are a bit of a code smell, because they often lead to strong coupling between what could be independent components. On a practical level, it makes it hard to navigate the code base, and even harder to understand. Plus you end up doing a lot of scrolling around!

## 4. Make deployment easy

I have seen it many times, but when builds and deployments are hard, it forces developers to work in a way that is both defensive, and slower. The QA process ends up being longer, because by the time a release is done, many changes may have been incorporated. This increases risk substantially. I do not think that any professional software developer today can avoid taking some interest in CI/CD systems.

## 5. Make your project as standard as possible for the language you are using, and keep it up to date

Every language has it's own conventions about how to set up a project. When you stray from this happy path, it's generally a recipe for confusion and frustration because new developers (both in experience and to the team) need to learn the intracacies of how your specific project has been set up, rather than being able to draw from tutorails or experience. Learning a new code base is hard enough without the added complexity of a hand written build system and strange folder structure.
