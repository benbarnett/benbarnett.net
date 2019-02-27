---
layout: post
title: Shared components ten years on
categories:
- Ramblings
- Tech
---

[I wrote about writing shared code nearly ten years ago](https://benbarnett.net/blog/2010/08/30/writing-html-and-css-for-mobile-safari-just-the-same-old-code) when Javascript in Safari on iOS devices was budding and [Flash had its swan song](https://www.apple.com/hotnews/thoughts-on-flash/). Thankfully, things on the technical side have improved since then.

For the past few months, I’ve been involved in a project aiming to bring consistency between three existing properties – Web, iOS native, Android native – for a well-known news organisation.

Its been fun, particularly as a chance to play with some new toys. But I wanted to take a look at how much ten years of the web maturing has helped.

## Offline

Many organisations have an existing web and native presence. Today, for the web, we’re talking Service Workers and an [AppCache fallback if using iOS webviews](https://webkit.org/blog/8090/workers-at-your-service/). This is a huge improvement since 2010 when offline was a pipe dream (but still two solutions, grumble grumble).

So how should the components behave when offline?

Start a new project today and offline can be considered before you lay foundations. Take an existing project and coercing it to work offline is more involved. Even one that is _relatively_ modern. 

Of course the technical side is but one voice in the offline choir, and for once it is one of the more mature. The [how-to guides are excellent](https://developers.google.com/web/fundamentals/instant-and-offline/offline-cookbook/), and the [growing pains from AppCache](https://alistapart.com/article/application-cache-is-a-douchebag) have been built upon. 

What is less mature however, is the process for how teams wrangle this problem.

## Collaboration across teams, disciplines and deliveries

[This project](https://github.com/Financial-Times/x-dash/) required involvement from at least three different teams, each with their own goals and delivery cadences.

Being outside our usual autonomous boundaries, the edges & responsibilites of each team need time to bubble up. For us, we found ourselves needing to define lower level details much earlier in the process. Which team owns the code for the various elements? Offline cuts across features.

Unintentionally, we were moving closer to something like an outsourcing model. Don’t know what you want before asking others to build it? Risk time and money. Plan further ahead than comfortable and make course-correcting difficult.

Lets say you have a home page and an article page. These pages have different product teams, but use a shared component that needs to be changed to handle offline. You also run these pages on the three different device platforms.

Prioritising this work is a tough sell when the various product owners have existing commitments. Even if they dig your new thing, it might not be appropriate for them to drop other work in place of it. Team coordination issues can appear.

Bring in native release cycles (App stores) and you _really_ need to get this stuff right before shipping.

## How generic is too generic?

As we found with [Dough](https://benbarnett.net/blog/2015/01/29/making-with-dough), the shared component library for the [Money Advice Service](https://www.moneyadviceservice.org.uk/en), there is a debate on whether to embrace the constraints of a framework or the freedom of being tech agnostic. 

Dough was in a Ruby on Rails world, the [more recent project a Node.js one](https://github.com/Financial-Times/x-dash/). My experience is, and this was not always the case, that it is better to lean on an established toolset. *Make the boring tech choice.* 

Things get confusing when the bulk of the code is built with Rails, but there is a handful of custom tools alongside. The same goes for React. New developers have a tough time understanding ‘how much’ of the tool or framework is available to them.

## Where next?

Working towards consistency from a shared codebase can appear to slow things down to a point where it is hard to justify. Would it have been faster for the separate teams to implement their own verions, rather than come up with a shared solution? [...question left dramatically unanswered...]

The efficiency of writing once & deploying everywhere comes after the team coordination dance has been done. The key is getting past that point. The first phase is high cost, low (apparent) output.

Generally, I think we help team coordination by taking another look at product ownership. Slicing ownership by page doesn’t always cut it, and owning features without the context of the pages they live on isn’t always better. We have guilds for cross-cutting concerns, but we can go further.

For me, this remains the hardest piece of the puzzle. Whether it was working on [Dough](https://benbarnett.net/blog/2015/01/29/making-with-dough) or [x-dash](https://github.com/Financial-Times/x-dash/), the organisational fit for a quality set of shared components doesn’t necessarily align with a feature-focused outcome. This mainly applies to larger organisations with multiple teams.

While writing this post, [Darian Moody tweeted](https://twitter.com/djm_/status/1100776855236476928) something that distills this post into a few words:

> Changing code is easy.
> 
> Changing human process is the hard mode.

The need for this way of working isn’t going away. I hope to write another post – in less than a decade – that speaks of how the industry adapted to make all this a bit easier.
