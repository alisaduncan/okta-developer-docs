---
title: Okta Auth JavaScript SDK and Vue
language: Vue
icon: code-vue
excerpt: Integrate Okta with a Vue app using Auth JS.
---

> **Note:** This document is only for Classic Engine. If you’re using Okta Identity Engine, see [Sign in to SPA with Auth JS](/docs/guides/sign-in-to-spa-authjs/vue/main). See [Identify your Okta solution](https://help.okta.com/okta_help.htm?type=oie&id=ext-oie-version) to determine your Okta version.

This guide walks you through integrating authentication into a Vue app with Okta by performing these steps:

* [Prerequisites](#prerequisites)
* [Add an OpenID Connect Client in Okta](#add-an-openid-connect-client-in-okta)
* [Create a Vue App](#create-a-vue-app)
* [Install Dependencies](#install-dependencies)
* [Create a Custom Sign-In Form](#create-a-custom-sign-in-form)
* [Create Routes](#create-routes)
* [Start your app](#start-your-app)
* [Conclusion](#conclusion)

> **Note**: This guide is for `@okta/okta-auth-js` >= v5.0.0 and < 6.0.0, `vue` 3.

## Prerequisites

If you don’t already have an Okta Integrator Free Plan org, you can create one at <https://developer.okta.com/signup/>.

## Add an OpenID Connect client in Okta

* Sign in to the Okta developer dashboard, and select **Create New App**.
* Choose **Single Page App (SPA)** as the platform, then populate your new OpenID Connect app with appropriate values for your app. For example:

   | Setting              | Value                                               |
   | -------------------  | --------------------------------------------------- |
   | App name             | OpenID Connect App (must be unique)                 |
   | Login redirect URIs  | `http://localhost:8080/login/callback`              |
   | Logout redirect URIs | `http://localhost:8080/`                            |
   | Allowed grant types  | Authorization Code                                  |

> **Note:** It's important to choose the appropriate app type for apps that are public clients. Failing to do so may result in Okta API endpoints attempting to verify an app's client secret. Public clients aren’t designed to a client secret, hence breaking the sign-in or sign-out flow.
>
> **Note:** CORS is automatically enabled for the granted login redirect URIs.

## Create a Vue app

To quickly create a Vue app, Okta recommends the Vue CLI. Follow [their guide](https://cli.vuejs.org/guide/installation.html) or use the following steps.

```bash
npm install -g @vue/cli
vue create okta-vue-auth-example
cd okta-vue-auth-example
```

## Install dependencies

A simple way to add authentication to a Vue app is using the [Okta Auth JavaScript SDK](/docs/guides/auth-js) (auth.js). You can install it through `npm`:

```bash
npm install @okta/okta-auth-js
```

## Create a custom sign-in form

[Auth.js](/docs/guides/auth-js) provides more options that the Sign-In Widget: user lifecycle operations, MFA, and more. In this example, you create a simple username and password form without multifactor authentication.

Create `src/components/About.vue` with the following HTML:

```html
<template>
  <div>
    <h2>About</h2>
  </div>
</template>
```

Create `src/components/Dashboard.vue`. This page can only be viewed by authenticated users.

```html
<template>
  <div>
    <h2>Dashboard</h2>
    <p>Yay you made it!</p>
  </div>
</template>
```

Create a `src/auth.js` file:

```js
const OktaAuth = require('@okta/okta-auth-js').OktaAuth
const authClient = new OktaAuth({
  issuer: 'https://{yourOktaDomain}',
  clientId: '{clientId}',
  scopes: ['openid', 'email', 'profile'],
  redirectUri: window.location.origin + '/login/callback'
})

export default {
  login (email, pass, cb) {
    cb = arguments[arguments.length - 1]
    if (localStorage.token) {
      if (cb) cb(true)
      this.onChange(true)
      return
    }
    return authClient.signInWithCredentials({
      username: email,
      password: pass
    }).then(transaction => {
      if (transaction.status === 'SUCCESS') {
        return authClient.token.getWithoutPrompt({
          responseType: ['id_token', 'token'],
          sessionToken: transaction.sessionToken,
        }).then(response => {
          localStorage.token = response.tokens.accessToken
          localStorage.idToken = response.tokens.idToken
          if (cb) cb(true)
          this.onChange(true)
        })
      }
    }).catch(err => {
      console.error(err.message)
      if (cb) cb(false)
      this.onChange(false)
    })
  },

  getToken () {
    return localStorage.token
  },

  logout (cb) {
    delete localStorage.token
    delete localStorage.idToken
    if (cb) cb()
    this.onChange(false)
    return authClient.signOut()
  },

  loggedIn () {
    return !!localStorage.token
  },

  onChange () {
  }
}
```

Replace `{yourOktaDomain}` with your Okta domain in the previous code example. Replace `{clientId}` with the client ID from the app that you created earlier.

Change `src/App.vue` to have the following code:

```html
<template>
  <div id="app">
    <h1>Okta Auth JS Example</h1>
    <ul>
      <li>
        <router-link v-if="loggedIn" to="/logout">Log out</router-link>
        <router-link v-if="!loggedIn" to="/login">Log in</router-link>
      </li>
      <li>
        <router-link to="/about">About</router-link>
      </li>
      <li>
        <router-link to="/dashboard">Dashboard</router-link>
        (authenticated)
      </li>
    </ul>
    <template v-if="$route.matched.length">
      <router-view></router-view>
    </template>
    <template v-else>
      <p>You are logged {{ loggedIn ? 'in' : 'out' }}</p>
    </template>
  </div>
</template>

<script>
import auth from './auth'
export default {
  data () {
    return {
      loggedIn: auth.loggedIn()
    }
  },
  created () {
    auth.onChange = loggedIn => {
      this.loggedIn = loggedIn
    }
  }
}
</script>

<style>
  html, body {
    font-family: -apple-system, BlinkMacSystemFont, "Segoe UI", Roboto, Helvetica, Arial, sans-serif, "Apple Color Emoji", "Segoe UI Emoji", "Segoe UI Symbol";
    color: #2c3e50;
  }

  #app {
    padding: 0 20px;
  }

  ul {
    line-height: 1.5em;
    padding-left: 1.5em;
  }

  a {
    color: #7f8c8d;
    text-decoration: none;
  }

  a:hover {
    color: #4fc08d;
  }
</style>
```

Add a `src/components/Login.vue` to render your sign-in form:

```html
<template>
  <div>
    <h2>Login</h2>
    <p v-if="$route.query.redirect">
      You need to login first.
    </p>
    <form @submit.prevent="login" autocomplete="off">
      <label><input v-model="email" placeholder="email" v-focus></label>
      <label><input v-model="pass" placeholder="password" type="password"></label><br>
      <button type="submit">login</button>
      <p v-if="error" class="error">Bad login information</p>
    </form>
  </div>
</template>

<script>
  import auth from '../auth'
  export default {
    data () {
      return {
        email: '',
        pass: '',
        error: false
      }
    },
    methods: {
      login () {
        auth.login(this.email, this.pass, loggedIn => {
          if (!loggedIn) {
            this.error = true
          } else {
            this.$router.replace(this.$route.query.redirect || '/')
          }
        })
      }
    }
  }
</script>

<style>
  .error {
    color: red;
  }
</style>
```

To make the `v-focus` directive on the email field work, modify app creation code in `src/main.js` as follows:

```js
createApp(App)
.directive('focus', {
  // When the bound element is inserted into the DOM...
  mounted: function (el) {
    // Focus the element
    el.focus()
  }
})
.use(router)
.mount('#app')
```

## Create routes

Some routes require authentication before rendering. Let's look at what routes are needed for this example:

* `/`: The default landing page.
* `/about`: A simple about page.
* `/dashboard`: A route that's protected.
* `/login`: The sign-in form.
* `/logout`: A route to sign out a user and redirect them back to the default page.

Create `src/router/index.js` with the following code.

```js
import { createRouter, createWebHistory } from 'vue-router'
import auth from '@/auth'
import About from '@/components/About.vue'
import Dashboard from '@/components/Dashboard.vue'
import Login from '@/components/Login.vue'

const router = createRouter({
  history: createWebHistory(__dirname),
  routes: [
    { path: '/about', component: About },
    { path: '/dashboard', component: Dashboard, beforeEnter: requireAuth },
    { path: '/login', component: Login },
    { path: '/logout',
      beforeEnter (to, from, next) {
        auth.logout()
        next('/')
      }
    }
  ]
})

function requireAuth (to, from, next) {
  if (!auth.loggedIn()) {
    next({
      path: '/login',
      query: { redirect: to.fullPath }
    })
  } else {
    next()
  }
}

export default router;
```

## Start your app

Finally, start your app:

```bash
npm run serve
```

<!--
// todo: this PR hasn't been approved and DevEx isn't sure they want Auth JS
// samples with the rest of the samples.
## Source Code

The source code for this guide can be found in [in the Vue Samples](https://github.com/okta/samples-js-vue/tree/master/okta-auth-js) on GitHub.
-->

## Conclusion

You have now successfully authenticated with Okta. With a user's `id_token`, you have basic claims for the user's identity. You can extend the set of claims by modifying the `scopes` to retrieve custom information about the user. This includes `locale`, `address`, `groups`, and [more](https://developer.okta.com/docs/api/openapi/okta-oauth/guides/overview/).
