---
layout: post
title: Module Federation at scale
categories:
  - Tech
---

Firstly, this article is not a webpack module federation getting started guide. I talk here about how this kind of technology can be brought to large companies and how we can scale, iterate rapidly, and be robust.

[Note to self: try to avoid the phrase 'game changer']

Take a microservice ecosystem, comprising of hundreds of apps serving a single experience to many end users. Those many users who by the way, couldn’t care less about microservices. They might have a couple more opinions on inconsistent UX or broken functionality, though.

In a "traditional" setup (I love how we get to use words like this literally instantly after using a new technology), we might publish shared components to an npm registry, using a [semantic versioning policy](https://semver.org/). This is great because the various apps can pin to a major e.g. `1.x.x` and not worry about breaking changes. They might also use dependabot to automate some of this.

The downside here is even in the most efficent update strategy, there is a lag between publishing a new version and the hundreds of apps pulling it in. This is particularly evident if this is something like navigation or a logo change.

In the style of Alan Partridge, we cut to an underside camera angle and offer a tonal gear shift:

> “Can we do better?”

Microfrontends are nothing new, but [iframe implementations have been tricky to stitch together](https://martinfowler.com/articles/micro-frontends.html#Run-timeIntegrationViaIframes).

# Module federation as continuous component delivery

With federation we do away with dependency bumps and think of our libraries as applications. Deploy code once, and its updated everywhere. The code, although fetched asynchronously through a network request, is executed in-place.

This is one to be aware of; its useful to inherit shared state (think `styled-components` theme context), but important to have a well defined boundary to avoid unwanted overlap.

## How?

This is a relatively recent addition to Webpack and even if the API is stable, the tooling and patterns around it are emerging rapidly. You might consider abstracting the component loading side of things, allowing easier refactoring as mechanisms evolve. This approach leaves consuming applications with a stable API.

Depending on your setup, you might need to define the remote application urls at runtime (similar to the [dynamic-system-host example](https://github.com/module-federation/module-federation-examples/tree/master/dynamic-system-host) provided by Zach). This can all form part of the component loader and therefore take into account any environment specific information (locales, branch deploys, etc).

One of the key benefits to microfrontends is giving component authors autonomy over their development cycles and having simple, decoupled codebases. These advantages will sound familiar if you’re used to working with microservices.

At a larger company, there will likely be multiple remote applications, each providing their own set of modules. For this reason it can be useful to have a simple registry of known remotes to keep this under control.

All this manifests as a simple API, which we've kept as 'Reacty' as possible, given that's the language apps are written in:

```tsx
// RemoteModule.tsx - a highly simplified version of a component loader

// conditional props for remote applications and their known modules
type ModuleProps =
  | { remote: "Navigation"; module: "AccountSettings" | "Footer" | "Header" }
  | { remote: "OtherRemote"; module: "OtherComponent" };

// union for props above and the optional fallback state for React Suspense
type RemoteModuleProps = ModuleProps & { fallback: React.ReactNode | null };

export const RemoteModule = ({
  remote,
  module,
  fallback,
}: RemoteModuleProps) => {
  const Module = React.lazy(() => loadComponent({ remote, module }));

  return (
    <ErrorBoundary>
      <React.Suspense fallback={fallback}>{Module}</React.Suspense>
    </ErrorBoundary>
  );
};
```

Then over in our Navigation component, we can export an implementation:

```tsx
export const Navigation = () => <RemoteModule remote="Navigation" module="AccountSettings" fallback={<Loading />}>
```

Teams can then render this in a familiar fashion:

```tsx
<Navigation />
```

Behind the scenes, `loadComponent()` will fetch the `Navigation` remote (or return a cached copy if another component got there first), and then call `Navigation.get('AccountSettings')` - returning a promise which will resolve with the AccountSettings module.

It is advisable to wrap the remote module in an [`<ErrorBoundary />`](https://reactjs.org/docs/error-boundaries.html) here which provides the owners of the `Navigation` remote application more insight into when things go awry.

Errors relevant to the component authors (in this case, the Navigation team) can be sent transparently, but it is worth passing errors up to the consuming (host) app so they can handle their own UX gracefully as well.

## Deploying _everywhere_ kind of removes our safety net?

It does. Yes, we need to make sure our tests are covering all the key areas. We can’t issue breaking changes like we do with libraries.

Once in the mindset of thinking of your libraries as applications, there is much potential to tap. A/B testing (and many other forms of experimentation) becomes a breeze, for example.

### Testing

Right now its difficult to fully test federated modules in Jest (i.e. JSDOM) because of the dependency on webpack globals. We therefore rely on in-browser testing to ensure our components are fully loaded.

A nice pattern is to have your component library documentation pull in the components via federation, too. Not only does this demonstrate the asynchronous nature of the modules, but it provides the ideal test harness for your [e2e](https://www.cypress.io/) or [synthetic tests](https://www.datadoghq.com/blog/browser-tests/).

### Monitoring

Another advantage for applications as libraries is that we have better observability. A sudden flatline in traffic to our component endpoints? Perhaps we broke our component loader somehow. We can monitor latency, error rates, everything we’re used to doing with applications.

### Caching

Thanks to Webpack’s [deterministic module naming algorithm](https://webpack.js.org/configuration/optimization/#optimizationmoduleids), long term caching for the modules is still possible. It will also resolve imports and re-use anything that’s been loaded already (either by the host app, or any other federated modules). This is all enabled by default in production on Webpack 5.

The thing to watch out for is the `remoteEntry.js` file - our manifest for each of the remote applications. This is something we don’t want to cache so apps are always being signposted to the right modules and their dependencies.

## Fiddly bits

[Shared module configuration](https://webpack.js.org/concepts/module-federation/) is incredibly powerful but can be tricky to grok. Having the ability to provide fallback modules should a host app not have a particular dependency available, or equally, to avoid loading it twice if its already available, makes for a highly robust system. It can be taxing to diagnose when incorrect.

In my experiments so far, I’ve needed some overrides to optimization settings in the webpack config. I won’t lie to you, I have spent a lot of time reading through module manifests trying to work out why things aren’t loading as expected. Usually it boils down to a [different set of defaults in production](https://webpack.js.org/configuration/optimization/), or a mismatch in the Shared Module configuration.

# Conclusion

This feels like a big step forward for microfrontends, and frontend in general. We are really only scratching the surface.

Of the technical gotchas, the fixes have typically been one-liners, but the diagnosis a _lot_ more than that. The [documentation on webpack.js.org](https://webpack.js.org/concepts/module-federation/) is improving, with suggestions for how to solve some common problems being added.

I really recommend reading [Practical Module Federation](https://module-federation.myshopify.com/products/practical-module-federation) by Jack Herrington & Zach Jackson as many of the resources in there go in to more detail than can be found online. However the [module-federation-examples on Github](https://github.com/module-federation/module-federation-examples) provide plenty of inspiration. The community is really active over there, too.

On balance, combined with how well this can scale, this is something really rather [~~game changing~~](https://www.macmillanthesaurus.com/game-changer) _scene altering_.
