---
layout: post
title: Module Federation at scale
categories:
- Tech
---

Firstly, this article is not a webpack module federation getting started guide. I talk here about how this kind of technology can be brought to large companies and how we can scale, iterate rapidly, and be robust.

[Note to self: try to avoid the phrase 'game changer']

A large company I am working with operate a microservice  architecture, comprising of hundreds of apps serving a single experience to the many end users. Those many users who by the way, couldn’t care less about microservices. They might have a couple more opinions on inconsistent UX or broken functionality, though.

In a "traditional" setup (I love how we get to use words like this literally instantly after using a new technology), we might publish shared components to an npm registry, using a [semantic versioning policy](https://semver.org/). This is great because the various apps can pin to a major e.g. `1.x.x` meaning they don't need to worry about breaking changes coming their way. Even better, they can go further and use dependabot to automated minor & patch upgrades. The downside here is even in the most efficent update strategy, there is _always_ a lag between publishing a new version and the hundreds of apps pulling it in. And for internal dependencies, sometimes these aren’t the top priority.

This means there is an inevitable time period where users are seeing different versions of components across different pages. This is particularly noticable if this is something like navigation or a logo change.

In the style of Alan Partridge, we cut to an underside camera angle and offer a tonal gear shift. “Can we do better?”

Microfrontends are nothing new, but iframe implementations - although forcing a strict 'frame' around each delivery - have felt tricky to stitch together.

# Module federation as continuous component delivery

With federation we do away with dependency bumps. We deploy our code once, and its updated everywhere. The code, although fetched remotely through a network request, is executed in-place. This is one to be aware of; its great to consume shared state and any global reset styles, but important to have a well defined component boundary (or "frame").

## How tho

This is a new-ish technology and even if the API is fairly stable, the tooling and patterns around it are emerging rapidly. We chose to abstract as much of this as possible behind a helper component which will do much of the configuration for the app teams consuming the components. 

We have a few lines of code that allow us to define the webpack remote entrypoint at runtime (similar to the [dynamic-system-host example](https://github.com/module-federation/module-federation-examples/tree/master/dynamic-system-host)). This is essentially our component loader and a crucial part to serving users the right version of components.

These patterns are emerging, and there are [plugins evolving from community discussion on github](https://github.com/module-federation/module-federation-examples/issues/566).

For this reason, and well, we are writing frontend code anyway, we follow defensive architecture principles. Having a central definition for how we load components, we can refactor our component loader to use any mechanism we like, provided the API we provide to teams remains stable.

Given we have multiple remote applications serving different collections modules, we have a small registry of remotes applications. We use separate remotes following the same principles as we use microservices.

All this manifests as kind of interface for teams, which we've kept as 'Reacty' as possible, given that's the language apps are written in:

```tsx
<RemoteModule remote="Navigation" module="AccountSettings" fallback={<Loading />}>
```

Behind the scenes, we fetch the `Navigation` remote (or return a cached copy if another component got there first), and then call `.get('AccountSettings')` on the resolved container.

We also render an ErrorBoundary here which provides the owners of the `Navigation` remote application all the observability they could hope for. This happens transparently to the app team, but we pass the errors up to them too so they can handle their own UX gracefully.


## Deploying _everywhere_ kind of removes our safety net?

You're right, it does. We really need to up our game when it comes to e2e testing and monitoring. But you were doing this already right?

### Testing

Right now its difficult to fully test federated modules in Jest or similar because of the dependency on webpack globals. So we do rely more heavily on Cypress to ensure our components are fully loaded in a browser context.

### Monitoring

The nice thing about federated module remotes is of course that you are a running application, not just a library on npm. This is great because we can add much more detailed tracing and relevant alerts for how things are operating. A sudden flatline in traffic to our component endpoints? Perhaps we broke our component loader somehow. 

Most apps are static & client side rendered. So we've been fortunate to use React Suspense (although React 18 looks to have support for SSR with Suspense) without the need for additional libraries such as Loadable. 

## Fiddly bits

So far, by abstracting the slightly grittier code - remote module system loading - into our own helper library, we've kept away some of the complexity. We also provide a shared `createWebpackConfig()` function to keep the configuration required by app teams simple. 

It seems some of the most powerful elements of module federation is the shared module configuration. Having the ability to provide fallback modules should a host app not have a particular dependency available, or equally, to avoid loading it twice if its already available, makes for a highly robust system. 

Right now we share all available dependencies, and configure a few of them to be singletons to ensure we have a single copy of the various states. The real test for this will come when we need to gradually update one of the major libraries (React 18?). There are a few options here, but that's for another article.

# Conclusion

We want to convert more & more components to use this. We started with small, arguably non-critical components and will dial up our confidence to more critical ones as we go.

There were definitely some technical hurdles to overcome. We still have some overrides in our webpack config to make things behave. We also found some gotchas between development and production environments due to differing webpack defaults (such as module & chunk naming). The fixes have typically been one-liners, but the diagnosis a _lot_ more than that. I have a follow up article about one particular issue in the works.
