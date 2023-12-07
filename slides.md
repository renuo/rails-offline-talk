---
layout: cover
background: https://sli.dev/demo-cover.png
theme: eloc
---

# Offline Mode in Rails

How to implement offline CRUD actions for an existing ruby on rails application with service workers, indexed DB and stimulus.

Short beertalk by Daniel

---

# Why should YOU care?

- You might be interested in implementing offline mode yourself
- Your clients might need to use the application in bad internet conditions
- You want to learn something about request / response caching?
- You like JavaScript, pain and

---

# Problems to be addressed

1. Saving views for offline use
1. Testing?
1. Preloading of certain views
1. Handling offline form submissions
1. Updating views
1. Syncing offline tasks

---

# Problem #1

As a user, I want to be able to access the page in offline mode, after visiting it in the past

## Solution:

Use a service worker to intercept outgoing HTTP requests and store them in the browser cache
Retrieve the cached response, if connection to the server cannot be established

## Tech
### Browser cache APIs
- [https://developer.mozilla.org/en-US/docs/Web/API/Cache](https://developer.mozilla.org/en-US/docs/Web/API/Cache)
- Cache.match(request, options)
- Cache.put(request, response)
### Service Worker API
- https://developer.mozilla.org/en-US/docs/Web/API/Service_Worker_API
### Service Worker Life Cycle

---

# What is a service worker?
> A service worker is an event-driven worker registered against an origin and a path. It takes the form of a JavaScript file that can control the web-page/site that it is associated with, intercepting and modifying navigation and resource requests, and caching resources in a very granular fashion to give you complete control over how your app behaves in certain situations (the most obvious one being when the network is not available).\
> https://developer.mozilla.org/en-US/docs/Web/API/Service_Worker_API

## Use Cases
- Background data synchronization.
- Responding to resource requests from other origins.
- Receiving centralized updates to expensive-to-calculate data such as geolocation or gyroscope, so multiple pages can make use of one set of data.
- Client-side compiling and dependency management of CoffeeScript, less, CJS/AMD modules, etc. for development purposes.
- Hooks for background services.
- Custom templating based on certain URL patterns.
- Performance enhancements, for example pre-fetching resources that the user is likely to need in the near future, such as the next few pictures in a photo album.
- API mocking.

---

# Registering Service Workers
```js
const registerServiceWorker = async () => {
  if ('serviceWorker' in navigator) {
    try {
      const registration = await navigator.serviceWorker.register('/service-worker.js', {
        scope: '/',
      })

      if (registration.installing) {
        console.log('Service worker installing')
      } else if (registration.waiting) {
        console.log('Service worker installed')
      } else if (registration.active) {
        console.log('Service worker active')
      }
    } catch (error) {
      console.error(`Registration failed with ${error}`)
    }
  }
}
registerServiceWorker()
```

---

# Step 1: Service Worker

## Network-First implementation of request caching:
Q: Why network first?
A: So that we don't need to deal with cache invalidation :)

### Case 1: User is online

1. Check if request url needs to be cached
2. Fetch the response from the network
3. Store the response alongside the request in the cache
4. Respond with the response

### Case 2: The user attempts to visit a page while offline
1. Check if request url needs to be cached
2. Attempt to fetch the response from the network and fail
3. Retrieve the response from the cache

### Case 3: The user attempts to visit a new page while offline
1. Check if request url needs to be cached
2. Attempt to fetch the response fromm the network and fail
3. Attempt to retrieve the response from cache and fail
4. Respond with a network error (because there is no fallback yet)

---

# Step 0: Intercept the outgoing fetch request
Q: What is the CacheManager, where does it come from?\
A: It is an abstraction over the browser cache APIs becuase I like abstraction so live with it.\
Q: Why are you not using something like Workbox?\
A: GOOD IDEA! IT IS VERY COOL THAT I THOUGHT OF IT ONLY AFTER WASTING HOURS TRYING TO SOLVE IT BY SCRATCH!

```js
const cacheManager = new CacheManager('v1')

self.addEventListener('fetch', (event) => {
  event.respondWith(cacheManager.retrieveOrStore(event))
})
```

---

# Step 1: Filter the request
```js
  async retrieveOrStore({ request }) {
    const filters = [
      (url) => url.startsWith('chrome-extension://'),
      ...
    ]

    if (request.method !== 'GET' || filters.some((filter) => filter(request.url))) {
      return fetch(request)
    }
    ...
  }
```

---

# Step 2: Attempt to retrieve the request from the network

```js
  async retrieveOrStore({ request }) {
    ...
    try {
      const networkResponse = await fetch(request)
      await this.putToCache(request, networkResponse.clone())
      return networkResponse
    } catch (error) {
      ...
    }
  }
```

---

# Step 3: Attempt to retrieve the response from the cache

```js
  async retrieveOrStore({ request }) {
      const cachedResponse = await this.getFromCache(request)
      if (cachedResponse) {
        return cachedResponse
      }

      const fallbackResponse = await this.getFromCache(this.fallbackUrl)
      if (fallbackResponse) return fallbackResponse

      return new Response({status: 503})
  }
```

---


---

```js

class CacheManager {
  cacheName = 'v1'

  constructor(cacheName, fallbackUrl) {
    this.cacheName = cacheName
    this.fallbackUrl = fallbackUrl
  }

  async putToCache(request, response) {
    console.log('[Serviceworker]', 'Putting to cache!', request.url)
    const cache = await caches.open(this.cacheName)
    return cache.put(request, response)
  }

  async getFromCache(request) {
    const cache = await caches.open(this.cacheName)
    const matchOptions = {
      ignoreSearch: true,
      ignoreVary: true,
      ignoreFragment: true,
    }
    return cache.match(request, matchOptions)
  }

  async retrieveOrStore({ request, preloadResponsePromise }) {
    console.log('[Serviceworker]', 'Retrieving or storing!', request.url)
    const filters = [
      (url) => /api.mapbox.com.+vector\.pbf/.test(url),
      (url) => url.startsWith('chrome-extension://'),
      (url) => url.endsWith('/connection_check'),
    ]

    if (request.method !== 'GET' || filters.some((filter) => filter(request.url))) {
      console.log('[Serviceworker]', 'Ignoring!', request.url)
      return fetch(request)
    }

    const preloadResponse = await preloadResponsePromise
    if (preloadResponse) {
      console.log('[Serviceworker]', 'Preload response!', request.url)
      await this.putToCache(request, preloadResponse.clone())
      return preloadResponse
    }

    try {
      if (self.emulateOfflineState) throw new Error('Emulating offline state!')
      console.log('[ServiceWorker]', 'not emulating offline!!')

      const networkResponse = await fetch(request)
      await this.putToCache(request, networkResponse.clone())
      return networkResponse
    } catch (error) {
      const cachedResponse = await this.getFromCache(request)
      if (cachedResponse) {
        console.log('[Serviceworker]', 'Found in cache!', request.url)
        return cachedResponse
      }
      console.log('[Serviceworker]', 'Cache miss!', request.url)

      const fallbackResponse = await this.getFromCache(this.fallbackUrl)
      if (fallbackResponse) return fallbackResponse

      throw new Error('fallback not cached!')
    }
  }
}

const cacheManager = new CacheManager('v1', '/offline_fallback')

self.addEventListener('install', (event) => {
    ...
})

self.addEventListener('fetch', (event) => {
  console.log('[Serviceworker]', 'Fetching!', event, event.request.url)
  event.respondWith(cacheManager.retrieveOrStore(event))
  cacheManager.logCacheContents()
})
```
