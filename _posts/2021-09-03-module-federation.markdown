---
layout: post
title: Module Federation at scale
categories:
  - Tech
---

Firstly, this article is not a webpack module federation getting started guide. I talk here about how this kind of technology can be brought to large companies and how we can scale, iterate rapidly, and be robust.

[Note to self: try to avoid the phrase 'game changer']

Take a microservice ecosystem, comprising of hundreds of apps serving a unified set of experiences to many end users. Those many users who by the way, couldn’t care less about microservices. They might have a couple more opinions on inconsistent UX or broken functionality, though.

In a "traditional" setup (I love how we get to use words like this literally instantly after using a new technology), we might publish shared components to an npm registry, using a [semantic versioning policy](https://semver.org/). This is great because the various apps can pin to a major e.g. `1.x.x` and not worry about breaking changes. They might also use dependabot to automate some of this.

The downside here is even in the most efficent update strategy, there is a lag between publishing a new version and potentially hundreds of apps pulling it in. This is particularly evident to users for global elements like navigation or logos.

In the style of Alan Partridge, we cut to a side camera angle and offer a tonal gear shift:

> “Can we do better?”

Microfrontends are nothing new, but [iframe implementations have been tricky to stitch together](https://martinfowler.com/articles/micro-frontends.html#Run-timeIntegrationViaIframes).

# Module federation as continuous component delivery

With federation we do away with dependency bumps and think of our libraries as applications. Deploy code once, and its updated everywhere. The code, although fetched asynchronously through a network request, is executed in-place.

This is one to be aware of; its useful to inherit shared state (think `styled-components` theme context), but important to have a well defined boundary to avoid unwanted overlap.

## How?

This is a relatively recent addition to Webpack and even if the API is stable, the tooling and patterns around it are emerging rapidly. You might consider abstracting the component loading side of things, allowing easier refactoring as mechanisms evolve. This approach leaves consuming applications with a stable API.

Depending on your setup, you might need to define the remote application urls at runtime (similar to the [dynamic-system-host example](https://github.com/module-federation/module-federation-examples/tree/master/dynamic-system-host) provided by Zach). This can all form part of the component loader and therefore take into account any environment specific information (locales, branch deploys, etc).

One of the key benefits to microfrontends is giving component authors autonomy over their development cycles and having simple, decoupled codebases. These advantages will sound familiar if you’re used to working with microservices.

At a larger company, there could be multiple remote applications, each providing their own set of modules. For this reason it can be useful to have a simple registry of known remotes and a generic loader to keep this under control.

All this manifests as a simple API (implemented here in React):

```tsx
// LazyModuleGeneric.tsx - a highly simplified version of a component loader

// conditional props for remote applications and their known modules
type ModuleProps =
  | { remote: "Navigation"; module: "AccountSettings" | "Footer" | "Header" }
  | { remote: "OtherRemote"; module: "OtherComponent" };

// union for props above and the nullable fallback state for React Suspense
type LazyModuleGenericProps = ModuleProps & { fallback: React.ReactNode | null };

export const LazyModuleGeneric = ({
  remote,
  module,
  fallback,
}: LazyModuleGenericProps) => {
  const Module = React.lazy(() => loadComponent({ remote, module }));

  return (
    <ErrorBoundary>
      <React.Suspense fallback={fallback}>
        <Module />
      </React.Suspense>
    </ErrorBoundary>
  );
};
```

Then over in our AccountSettings component, we can export an implementation:

```tsx
export const AccountSettings = () => {
  return (
    <LazyModuleGeneric
      remote="Navigation"
      module="AccountSettings"
      fallback={<Loading />}
    />
  );
};
```

Behind the scenes, `loadComponent()` will fetch the `Navigation` remote (or return a cached copy if another component got there first), and then call `Navigation.get('AccountSettings')`. This returns a promise which will resolve with the AccountSettings module. There is more detail on what ModuleFederation is doing here [on the Webpack docs](https://webpack.js.org/concepts/module-federation/#dynamic-remote-containers).

It is advisable to wrap the remote module in an [Error Boundary](https://reactjs.org/docs/error-boundaries.html) which can be configured to offer observability to the owners of `Navigation`.

Errors relevant to the component authors can be sent transparently, but it is worth passing errors up to the consuming (host) app so they can handle their own UX gracefully as well.

## Deploying _everywhere_ kind of removes our safety net?

It does. Yes, we need to make sure our tests are covering all the key areas. We can’t issue breaking changes like we do with libraries.

Once in the mindset of thinking of your libraries as applications, there is much potential to tap;

### Testing

Right now its difficult to fully test federated modules in Jest (i.e. JSDOM) because they depend on webpack globals such as `__webpack_init_sharing__` ([see on the docs](https://webpack.js.org/concepts/module-federation/#dynamic-remote-containers)). In my experiments I decouple the components from the globals, but in some cases I mock them. Its not bomb proof though, so I definitely recommend at least _some_ in-browser testing.

A nice pattern is to have your Design System docs site pull in its components via federation, too. Not only does this demonstrate the asynchronous nature of the modules, but it provides the ideal test harness for your [e2e](https://www.cypress.io/) or [synthetic](https://www.datadoghq.com/blog/browser-tests/) testing.

### Monitoring

Applications as libraries means we have better observability. A sudden flatline in traffic to our component endpoints? Perhaps we broke our component loader somehow. We can monitor latency, error rates, everything we’re used to doing with applications.

If your remote component is mission critical, it is possible to fallback to a local copy of the component, using the 'traditional' approach we talked about earlier. Ok, the app might have a slightly stale version, but its better than nothing should your remote application fail to respond.

### Caching

Thanks to Webpack’s [deterministic module naming](https://webpack.js.org/configuration/optimization/#optimizationmoduleids), we get long term caching for the modules. It will also resolve imports and re-use anything that’s been loaded already (either by the host app, or any other federated modules). This is all enabled by default in production on Webpack 5.

The thing to watch out for is the `remoteEntry.js` file - the remote application's module manfiest. This is something we want to cache carefully so apps are always being signposted to the right modules and their dependencies.

## Fiddly bits

[Shared module configuration](https://webpack.js.org/concepts/module-federation/) is incredibly powerful but can be tricky to grok. Providing fallback modules should a host app not have a particular dependency available, or equally, to avoid loading it twice if its already available, makes for a highly robust system. It can be taxing to diagnose when incorrect.

In my experiments so far, I’ve needed some overrides to optimization settings in the webpack config. I won’t lie to you, I have spent a lot of time reading through module manifests trying to work out why things aren’t loading as expected. Usually it boils down to a [different set of defaults in production](https://webpack.js.org/configuration/optimization/), or a mismatch in the Shared Module configuration.

# Conclusion

Of the technical gotchas, the fixes have typically been one-liners, but the diagnosis a _lot_ more than that. The [documentation on webpack.js.org](https://webpack.js.org/concepts/module-federation/) is improving, with suggestions for how to solve some common problems being added.

I recommend reading [Practical Module Federation](https://module-federation.myshopify.com/products/practical-module-federation) by Jack Herrington & Zach Jackson as it goes into more detail than I’ve been able to find online. Having said that, the [module-federation-examples on Github](https://github.com/module-federation/module-federation-examples) provide plenty of inspiration. The community is really active over there, too.

On balance, combined with how well this can scale, this really is something rather [~~game changing~~](https://www.macmillanthesaurus.com/game-changer) _scene altering_.
