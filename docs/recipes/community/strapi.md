# Strapi + Provider `authjs`

<RecipeHeader author="justserdar" :providers="['authjs']" :tags="['strapi']" />

This section gives an example of how the `NuxtAuthHandler` can be configured to use Strapi JWTs for authentication via the `CredentialsProvider` provider.

You have to configure the following places to make `nuxt-auth` work with Strapi:
- `STRAPI_BASE_URL` in `.env`: Add the Strapi environment variable to your .env file
- [`runtimeConfig.STRAPI_BASE_URL`-key in `nuxt.config.ts`](https://nuxt.com/docs/guide/going-further/runtime-config): Add the Strapi base url env variable to the runtime config
- [`auth`-key in `nuxt.config.ts`](/guide/application-side/configuration): Configure the module itself, e.g., where the auth-endpoints are, what origin the app is deployed to, ...
- [NuxtAuthHandler](/guide/authjs/nuxt-auth-handler): Configure the authentication behavior, e.g., what authentication providers to use

For a production deployment, you will have to at least set the:
- `STRAPI_BASE_URL` Strapi base URL for all API endpoints by default http://localhost:1337

1. Create a `.env` file with the following lines:
```
// Strapi v4 url, out of the box
AUTH_ORIGIN=http://localhost:3000
NUXT_SECRET=a-not-so-good-secret
STRAPI_BASE_URL=http://localhost:1337/api
```

1. Set the following options in your `nuxt.config.ts`:
```ts
export default defineNuxtConfig({
  runtimeConfig: {
    // The private keys which are only available server-side
    NUXT_SECRET: process.env.NUXT_SECRET,
    // Default http://localhost:1337/api
    STRAPI_BASE_URL: process.env.STRAPI_BASE_URL,
  },
})
```

1. Create the catch-all `NuxtAuthHandler` and add the this custom Strapi credentials provider:
```ts
// file: ~/server/api/auth/[...].ts
import CredentialsProvider from 'next-auth/providers/credentials'
import { NuxtAuthHandler } from '#auth'
const config = useRuntimeConfig()

export default NuxtAuthHandler({
  secret: config.NUXT_SECRET,
  providers: [
    // @ts-expect-error You need to use .default here for it to work during SSR. May be fixed via Vite at some point
    CredentialsProvider.default({
      name: 'Credentials',
      credentials: {}, // Object is required but can be left empty.
      async authorize(credentials: any) {
        const response = await $fetch(
          `${config.STRAPI_BASE_URL}/auth/local/`,
          {
            method: 'POST',
            body: JSON.stringify({
              identifier: credentials.username,
              password: credentials.password,
            }),
          }
        )

        if (response.user) {
          const u = {
            id: response.id,
            name: response.user.username,
            email: response.user.email,
          }
          return u
        }
        else {
          return null
        }
      },
    }),
  ]
})
```

:::tip Learn More
Checkout this blog-post for further notes and explanation: https://darweb.nl/foundry/article/nuxt3-sidebase-strapi-user-auth
:::
