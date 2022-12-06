---
title: "My React JS pain points"
date: 2022-11-21T18:30:00+01:00
draft: true
featured_image: '/images/stock-dev.jpg'
---

In my current role, I've been learning to work with React over roughly the past 11 months. Prior to this, I'd done bits and pieces of frontend work, but using pretty dated techniques - I could JQuery and Bootstrap with the best of them but hadn't had much exposure to modern frontend frameworks. My experience has primarily been working on the backends of systems and I think it's fair to say that for me, doing front end work has not been a natural fit. With that said, I've found a number of pain points as I've started to learn React and modern JS/Typescript concurrently.

## Complex tooling and builds

React and modern Javascript in general suffers from a configurability problem. Someone starting out on a React project is likely to jump into a tutorial and make use of [Create React App](https://reactjs.org/docs/create-a-new-react-app.html) which sets up Babel as a transpiler for ES6/Typescript and Webpack as a bundler and optimiser. But as soon as you stray away from the happy path, you will find that you need to configure these tools. Create React App doesn't have a nice way to do this - you have to "eject" from that ecosystem, and take responsibility and ownership for fully configuring these tools, along with all of the other core dependencies, and updating them as appropriate. Once you have ejected, you have much more control over your toolchain, but you are significantly more likely to run into issues with dependencies not being compatible.

As an alternative to ejecting, you can make use of projects like [react-app-rewired](https://www.npmjs.com/package/react-app-rewired), or [customize-cra](https://github.com/arackaf/customize-cra) which allow you to for add additional tweaks to your configuration. But even in this landscape, you can end up getting in a bit of a mess. It's difficult to know how to modify the configuration, becuase if you for e.g. read Babel's documentation, you'd expect to be managing the config in a file called `babel.config.json`, but this won't exist by default in a CRA project. From what I have seen in studying open source projects on GitHub and elsewhere, projects often end up with a `config-overrides.js` file that contains various overrides of the default CRA settings all mixed together.

The result of all of this is that it feels like you have to stick to a very very vanilla setup, or invest a huge amount of time in understanding the tooling landscape upfront.

## Dependency Hell

I along with many other developers chuckled a few years ago when the [left-pad debacle](https://www.theregister.com/2016/03/23/npm_left_pad_chaos/) happened. But at that time, I didn't quite appreciate how much frontend developers relied on external libraries. To me, the balance feels wrong - taking on a dependency has to be weighed up in cost to implement the equivalent functionality vs the cost if the project becomes incompatible with the core libraries of the project. This is a pain I've felt many times before. Keeping up with frontend package updates seems to me to be a huge time sink, and for little benefit in terms of the product outcomes.


