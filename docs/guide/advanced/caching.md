# Caching content

Often hosting providers offer caching on the edge. Most websites can experience incredible speeds (and cost savings) by taking advantage of caching. No cold start, no processing requests, no parsing Javascript... just HTML served immediately from a CDN.

By default we send the user's authentication data down to the client in the HTML. This might not be ideal if you're caching your pages. Users may be able to see other user's authentication data if not handled properly.

To add caching to your Nuxt app, follow the [Nuxt documentation on hybrid rendering](https://nuxt.com/docs/guide/concepts/rendering#hybrid-rendering).

:::warning
If you find yourself needing to server-rendered auth methods like `getProviders()`, you must set the `baseURL` option on the `auth` configuration object. **This applies in development too.**
:::

:::tip Acknowledgements
A big thanks to [KyleSmith0905](https://github.com/KyleSmith0905) for implementing routes rules into NuxtAuth. View their PR [here](https://github.com/sidebase/nuxt-auth/pull/610).
:::

## Page specific cache rules

If only a few of your pages are cached. Head over to the [Nuxt config `routeRules`](https://nuxt.com/docs/guide/concepts/rendering#route-rules), add the `auth` key to your cached routes. Set `disableServerSideAuth` to true.

```ts
export default defineNuxtConfig({
  modules: ['@sidebase/nuxt-auth'],
  auth: {
    // Optional - Needed for getProviders() method to work server-side
    baseURL: 'http://localhost:3000',
  },
  routeRules: {
    '/': {
      swr: 86400000,
      auth: {
        disableServerSideAuth: true,
      },
    },
  },
})
```

## Global cache rules

If all/most pages on your site are cached. Head over to the Nuxt config, add the `auth` key if not already there. Set `disableServerSideAuth` to true.

```ts
export default defineNuxtConfig({
  modules: ['@sidebase/nuxt-auth'],
  auth: {
    disableServerSideAuth: true,
    // Optional - Needed for getProviders() method to work server-side
    baseURL: 'http://localhost:3000',
  },
})
```

## Combining rules

Route-configured options take precedent over module-configured options. If you disabled server side auth in the module, you may still enable server side auth back by setting `auth.disableServerSideAuth` to `false`.

For example: It may be ideal to add caching to every page besides your profile page.

```ts
export default defineNuxtConfig({
  modules: ['@sidebase/nuxt-auth'],
  auth: {
    disableServerSideAuth: true,
  },
  routeRules: {
    // Server side auth is disabled on this page because of global setting
    '/': {
      swr: 86400000,
    },
    // Server side auth is enabled on this page - route rules takes priority.
    '/profile': {
      auth: {
        disableServerSideAuth: false,
      },
    },
  },
})
```
