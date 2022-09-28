---
title: "Some simple rules for maintainable software"
date: 2022-09-19T06:24:58+01:00
draft: false
featured_image: '/images/stock-dev.jpg'
---

Writing good software is an art rather than a science, and often decisions made quickly lead to a lot of pain down the line, because it is difficult to see the impact of them. With that said, here are a few of the rules I try and stick to when writing software in order to keep it maintainable and to allow course correction straightforwardly later.

## 1. Abstract away complexity

The KISS principle is well known among software developers, but it's all too common to see people try and incorporate clever solutions into their code bases. I am certainly not saying that this is never appropriate - if, for e.g., you hit a particularly bad performance problem then it may be the case that you need to switch to a more complex algorithm with better characteristics. However, this complexity should in most cases be abstracted away.

Take sorting for example. In almost every programming language, the complexity of sorting is hidden behind standard library functions. The user does not need to think about the underlying implementation. If you find that these are not sufficient for your use case, then the best thing you can do is to at least copy the interface of the standard methods, so that users will understand how to use it without having a significant cognitive load.

## 2. Know when to split code up to improve comprehensibility

I'm constantly surprised by the reluctance of many developers to split their code up either into independent modules or even into seperate files and functions. To me, large modules are a bit of a code smell, because they often lead to strong coupling between what could be independent components. On a practical level, it makes it hard to navigate the code base, and even harder to understand. Plus you end up doing a lot of scrolling around!

## 3. Document and test

Testing and integration of code is so important, a failure to do it well is often a death knell to a project a year or two down the line, even if the initial roll out goes well. The complexity in software is so often in hidden requirements and expected behaviours, which is one of the reason rewrites or restructures of parts of the code base often become fraught. Testing should capture these requirements, and avoid accidental breaking of functionality.

While I'm not a full proponent of TDD, I strongly believe that writing tests should be carried out alongside the initial feature development, because it often highlights inadequacies in the design. On many occasions, I've written a function that implements a feature and then realised that the API is difficult or clumsy to use. Likewise, when I come to a code base, I often find that shortcomings in this area on code bases I look at are accompanied by a lack of tests...

## 4. Make deployment easy

I have seen it many times, but when builds and deployments are hard, it forces developers to work in a way that is both defensive, and slower. The QA process ends up being longer, because by the time a release is done, many changes may have been incorporated. This increases risk substantially. I do not think that any professional software developer today can avoid taking some interest in CI/CD systems.

## 5. Make your project as standard as possible for the language you are using

Every language has it's own conventions about how to set up a project. When you stray from this happy path, it's generally a recipe for confusion and frustration because new developers (both in experience and to the team) need to learn the intracacies of how your specific project has been set up, rather than being able to draw from tutorails or experience. Learning a new code base is hard enough without the added complexity of a hand written build system and strange folder structure. Many languages have 'cookie cutter' templates that you can use.

## 6. Keep core dependencies up to date

Times move on, and it's important to keep up to date with current practice for your language and framework. Failing to do so is likely to cause you difficulty in upgrading to new versions, you may find that incorporating new dependencies and patterns into your project is hard or impossible, and it starts to become a barrier to entry to developers who have only learnt the framework in newer iterations.
