---
layout: post
title: Module Federation at scale
categories:
  - Tech
---

Firstly, this article is not a webpack module federation getting started guide. I talk here about how this kind of technology can be brought to large companies and how we can scale, iterate rapidly, and be robust.

[Note to self: try to avoid the phrase 'game changer']

A large company I am working with operate a microservice architecture, comprising of hundreds of apps serving a single experience to the many end users. Those many users who by the way, couldn’t care less about microservices. They might have a couple more opinions on inconsistent UX or broken functionality, though.

In a "traditional" setup (I love how we get to use words like this literally instantly after using a new technology), we might publish shared components to an npm registry, using a [semantic versioning policy](https://semver.org/). This is great because the various apps can pin to a major e.g. `1.x.x` and not worry about breaking changes. They might also use dependabot to automate some of this.

 The downside here is even in the most efficent update strategy, there is a lag between publishing a new version and the hundreds of apps pulling it in. This is particularly noticable if this is something like navigation or a logo change.

In the style of Alan Partridge, we cut to an underside camera angle and offer a tonal gear shift. 

> “Can we do better?”

Microfrontends are nothing new, but iframe implementations have felt tricky to stitch together.

# Module federation as continuous component delivery

With federation we do away with dependency bumps and think of our libraries as applications. We deploy our code once, and its updated everywhere. The code, although fetched remotely through a network request, is executed in-place.

This is one to be aware of; its useful to inherit shared state (think styled-component theme context), but important to have a well defined component boundary (or "frame").

## How?

This is a relatively recent addition to Webpack and even if the API is stable, the tooling and patterns around it are emerging rapidly. We chose to abstract a lot of the component loading logic into a library, allowing us to refactor the mechanisms here.

We have a few lines of code that allow us to define the webpack remote entrypoint at runtime (similar to the [dynamic-system-host example](https://github.com/module-federation/module-federation-examples/tree/master/dynamic-system-host) provided by Zach). This is our component loader and a crucial part to serving users the right version of components.

Given we have multiple remote applications, each serving their own modules, we have a small registry to store our remotes. We want to support this approach for the same reason we want to use microservices -- independent deployment cycles and team autonomy (amongst many other reasons).

All this manifests as a simple API, which we've kept as 'Reacty' as possible, given that's the language apps are written in:

```tsx
export const Navigation = () => <RemoteModule remote="Navigation" module="AccountSettings" fallback={<Loading />}>
```

Teams can then render this in a familiar fashion:

```tsx
<Navigation />
```

Behind the scenes, we fetch the `Navigation` remote (or return a cached copy if another component got there first), and then call `.get('AccountSettings')` - returning a promise which will resolve with the AccountSettings module.

We also render an ErrorBoundary here which provides the owners of the `Navigation` remote application all the observability they dream of.

Errors relevant to the component authors (in this case, the Navigation team) are sent transparently. We pass any errors up to the consuming (host) app so they can handle their own UX gracefully.

We lean heavily on `React.Suspense` for this and the asynchronous module loading.

## Deploying _everywhere_ kind of removes our safety net?

It does. Yes, we need to make sure our tests are covering all the right areas, and we make use of synthetic UI testing as much as possible on some of our consuming applications. We can’t issue breaking changes like we do with libraries.

Once in the mindset of thinking of your libraries as applications, there is much potential to tap. A/B testing (and many other forms of experimentation) becomes a breeze, for example.

### Testing

Right now its difficult to fully test federated modules in Jest (i.e. JSDOM) because of the dependency on webpack globals. We therefore rely on in-browser testing to ensure our components are fully loaded.

A nice pattern is to have your component library documentation pull in the components via federation, too. Not only does this demonstrate the asynchronous nature of the modules, but it provides the ideal test harness for your [e2e](https://www.cypress.io/) or [synthetic tests](https://www.datadoghq.com/blog/browser-tests/).

### Monitoring

Another advantage for applications as libraries is that we can more detailed observability. A sudden flatline in traffic to our component endpoints? Perhaps we broke our component loader somehow.

### Caching

Thanks to Webpack’s [deterministic module naming algorithm](https://webpack.js.org/configuration/optimization/#optimizationmoduleids), we can still benefit from long term caching for the modules that are loaded. It will also resolve any imports and re-use anything that’s been loaded already (either by the host app, or any other federated modules). This is all enabled by default in production on Webpack 5.

The thing to watch out for is the `remoteEntry.js` file - our manifest for each of the remote applications. This is something we don’t want to cache so apps are always being signposted to the right modules and their dependencies.

## Fiddly bits

So far, by abstracting the slightly grittier code - remote module system loading - into our own helper library, we’ve kept away some of the complexity. We also provide a shared `createWebpackConfig()` function to keep the configuration required by app teams simple.

[Shared module configuration](https://webpack.js.org/concepts/module-federation/) is incredibly powerful but can be tricky to grok. Having the ability to provide fallback modules should a host app not have a particular dependency available, or equally, to avoid loading it twice if its already available, makes for a highly robust system. But it can be tricky to diagnose when incorrect.

We also still have some overrides in our wepback config to make things behave, particularly as there is a [very different set of optimization defaults for development & production](https://webpack.js.org/configuration/optimization/) builds. I won’t lie to you, I have spent a lot of time reading through module manifests trying to work out why things aren’t loading as expected. Perhaps I’ll expand on some of these in another post.

# Conclusion

This feels like a big step forward for microfrontends and we are probably only scratching the surface

Of the technical gotchas, the fixes have typically been one-liners, but the diagnosis a _lot_ more than that. Even since we started this work a few months ago, the [documentation on webpack.js.org](https://webpack.js.org/concepts/module-federation/) has evolved, with suggestions for how to solve some common problems being added. These common problems are still bubbling up and I'm sure the patterns emerging will smooth over some of the gnarlier bits.

I really recommend [the book](https://module-federation.myshopify.com/products/practical-module-federation) as many of the resources in there go in to more detail than can be found online. However the [module-federation-examples on Github](https://github.com/module-federation/module-federation-examples) provide plenty of inspiration.

On balance, combined with how well this can scale, this is something really rather [~~game changing~~](https://www.macmillanthesaurus.com/game-changer) _scene altering_.
