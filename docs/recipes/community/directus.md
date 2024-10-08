# Directus + Provider `authjs`

<RecipeHeader author="madsh93" :providers="['authjs']" :tags="['directus']" />

This section gives an example of how the `NuxtAuthHandler` can be configured to use Directus JWTs for authentication via the `CredentialsProvider` provider and how to implement a token refresh for the Directus JWT.

The below is a code-example that needs to be adapted to your specific configuration:
```ts
import CredentialsProvider from 'next-auth/providers/credentials'
import { NuxtAuthHandler } from '#auth'

/**
 * Takes a token, and returns a new token with updated
 * `accessToken` and `accessTokenExpires`. If an error occurs,
 * returns the old token and an error property
 */
async function refreshAccessToken(refreshToken: {
  accessToken: string
  accessTokenExpires: string
  refreshToken: string
}) {
  try {
    console.warn('trying to post to refresh token')

    const refreshedTokens = await $fetch<{
      data: {
        access_token: string
        expires: number
        refresh_token: string
      }
    } | null>('https://domain.directus.app/auth/refresh', {
      method: 'POST',
      headers: {
        'Content-Type': 'application/json',
      },
      body: {
        refresh_token: refreshToken.refreshToken,
        mode: 'json',
      },
    })

    if (!refreshedTokens || !refreshedTokens.data) {
      console.warn('No refreshed tokens')
      throw refreshedTokens
    }

    console.warn('Refreshed tokens successfully')
    return {
      ...refreshToken,
      accessToken: refreshedTokens.data.access_token,
      accessTokenExpires: Date.now() + refreshedTokens.data.expires,
      refreshToken: refreshedTokens.data.refresh_token,
    }
  }
  catch (error) {
    console.warn('Error refreshing token', error)
    return {
      ...refreshToken,
      error: 'RefreshAccessTokenError',
    }
  }
}

export default NuxtAuthHandler({
  // secret needed to run nuxt-auth in production mode (used to encrypt data)
  secret: process.env.NUXT_SECRET,

  providers: [
    // @ts-expect-error You need to use .default here for it to work during SSR. May be fixed via Vite at some point
    CredentialsProvider.default({
      // The name to display on the sign in form (e.g. 'Sign in with...')
      name: 'Credentials',
      // The credentials is used to generate a suitable form on the sign in page.
      // You can specify whatever fields you are expecting to be submitted.
      // e.g. domain, username, password, 2FA token, etc.
      // You can pass any HTML attribute to the <input> tag through the object.
      credentials: {
        email: { label: 'Email', type: 'text' },
        password: { label: 'Password', type: 'password' },
      },
      async authorize(credentials: any) {
        // You need to provide your own logic here that takes the credentials
        // submitted and returns either a object representing a user or value
        // that is false/null if the credentials are invalid.
        // NOTE: THE BELOW LOGIC IS NOT SAFE OR PROPER FOR AUTHENTICATION!

        try {
          const payload = {
            email: credentials.email,
            password: credentials.password,
          }

          const userTokens = await $fetch<{
            data: { access_token: string, expires: number, refresh_token: string }
          } | null>('https://domain.directus.app/auth/login', {
            method: 'POST',
            body: payload,
            headers: {
              'Content-Type': 'application/json',
              'Accept-Language': 'en-US',
            },
          })

          const userDetails = await $fetch<{
            data: {
              id: string
              email: string
              first_name: string
              last_name: string
              role: string
              phone?: string
              cvr?: string
              company_name?: string
            }
          } | null>('https://domain.directus.app/users/me', {
            method: 'GET',
            headers: {
              'Content-Type': 'application/json',
              'Accept-Language': 'en-US',
              'Authorization': `Bearer ${userTokens?.data?.access_token}`,
            },
          })

          if (!userTokens || !userTokens.data || !userDetails || !userDetails.data) {
            throw createError({
              statusCode: 500,
              statusMessage: 'Next auth failed',
            })
          }

          const user = {
            id: userDetails.data.id,
            email: userDetails.data.email,
            firstName: userDetails.data.first_name,
            lastName: userDetails.data.last_name,
            role: userDetails.data.role,
            phone: userDetails.data.phone,
            cvr: userDetails.data.cvr,
            companyName: userDetails.data.company_name,
            accessToken: userTokens.data.access_token,
            accessTokenExpires: Date.now() + userTokens.data.expires,
            refreshToken: userTokens.data.refresh_token,
          }

          const allowedRoles = [
            '53ed3a6a-b236-49aa-be72-f26e6e4857a0',
            'd9b59a92-e85d-43e2-8062-7a1242a8fce6',
          ]

          // Only allow admins and sales
          if (!allowedRoles.includes(user.role)) {
            throw createError({
              statusCode: 403,
              statusMessage: 'Not allowed',
            })
          }

          return user
        }
        catch (error) {
          console.warn('Error logging in', error)

          return null
        }
      },
    }),
  ],

  session: {
    strategy: 'jwt',
  },

  callbacks: {
    async jwt({ token, user, account }) {
      if (account && user) {
        console.warn('JWT callback', { token, user, account })
        return {
          ...token,
          ...user,
        }
      }

      // Handle token refresh before it expires of 15 minutes
      if (token.accessTokenExpires && Date.now() > token.accessTokenExpires) {
        console.warn('Token is expired. Getting a new')
        return refreshAccessToken(token)
      }

      return token
    },
    async session({ session, token }) {
      session.user = {
        ...session.user,
        ...token,
      }

      return session
    },
  },
})
```

This was contributes by [@madsh93 from Github](https://github.com/madsh93) here:
- Github Comment: https://github.com/sidebase/nuxt-auth/v0.6/issues/64#issuecomment-1330308402
- Gist: https://gist.github.com/madsh93/b573b3d8f070e62eaebc5c53ae34e2cc
