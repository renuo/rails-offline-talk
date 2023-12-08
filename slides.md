---
layout: cover
company: Renuo AG
date: 08.12.2023
author: Daniel
---

# Offline Mode in Rails

A guide on how to use IndexedDB, Service Workers and Stimulus

Short beertalk by Daniel

---

# Why should YOU care?

- You might be interested in implementing offline mode yourself.
- Your clients might need to use the application in bad internet conditions.
- You want to learn something about request / response caching?
- You like JavaScript, pain and suffering.

---

# Problems and Solutions
Based on my experience with Offline mode for the "Vogelhuesli" project

1. Caching already visited views
1. Preloading of views
1. Submiting forms when offline
1. Task syncing when online
1. Updating views
1. Testing


---
layout: intro
---

# Problem #1
As a user, I want to be able to access the page in offline mode, after visiting it in the past

**Solution:**

- Use a service worker to intercept outgoing HTTP requests and store them in the browser cache
- Retrieve the cached response, if connection to the server cannot be established

---

# Tech
## Service Worker API
- https://developer.mozilla.org/en-US/docs/Web/API/Service_Worker_API
- navigator.serviceWorker.register(...)
- install event
- fetch event

## Browser cache APIs
- [https://developer.mozilla.org/en-US/docs/Web/API/Cache](https://developer.mozilla.org/en-US/docs/Web/API/Cache)
- Cache.match(request, options)
- Cache.put(request, response)

---
layout: quote
---

# What is a service worker?

<!-- > A service worker is an event-driven worker registered against an origin and a path. It takes the form of a JavaScript file that can control the web-page/site that it is associated with, intercepting and modifying navigation and resource requests, and caching resources in a very granular fashion to give you complete control over how your app behaves in certain situations (the most obvious one being when the network is not available).\ -->
<!-- > https://developer.mozilla.org/en-US/docs/Web/API/Service_Worker_API -->

- Background JavaScript file in a web browser.
- Acts as a proxy, intercepting network requests.
- Enables caching, offline capabilities, and performance improvements.
- Handles push notifications and event listening. 


---
layout: two-cols
---

# Service Worker life cycle

<h1 style="display: none;"></h1>
<p>PWA Essentials: Introduction to Service Workers - Techglimpse</p>

<img src="https://techglimpse.com/wp-content/uploads/2019/11/PWA-Service-Worker-Life-Cycle.png" style="width: 300px">

::right::
# Break down

1. Registration: Register service worker with `navigator.serviceWorker.register(scriptURL, options)`
2. Installation: Installation event is emited by the service worker
3. Activation: The service worker is activated
4. Idle: The service worker is active and ready to intercept requests
5. Fetch: Intercept a fetch request, return the response

---
layout: two-cols-header
---
# Imporant events

::left::

**Registration**
  - Browser is informed about the existence and the script location of the service worker.

**Installation**
  - Precache static assets / essential resources
  - Self-skip waiting

**Activation**
  - Cache management (delete old caches)
  - Update client data
  - Claim clients -> take control of all open client pages instead of waiting for reload.

::right::

**Fetch**
  - Cache-first Strategy: Check if resource is in cache, fetch from network otherwise.
  - Network-first Strategy: Attempt to fetch ressource from network and cache, refer to cache otherwise.
  - Offline Fallback Strategy: Respond with predefined fallback / custom offline page when cache and network is unavailable.

**Message**
  - Communication between client and service worker
  - Background sync

---

# Use Cases
- Background data synchronization.
- Caching resources for offline use.
- Hooks for background services.
- Performance enhancements like navigation preloading.

---

# Registering Service Workers
- Service Worker
- Companion script

```js{2,19,20|3-6,14-16|8-44}
// companion_script.ts
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
Diagram of the network first approach to caching

<img src="https://miro.medium.com/v2/resize:fit:1400/format:webp/0*5l-2aP5-sWYjjnY0.png" width="600px">

---

```js{3}
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

---
mdc: true
---

# Problems with service workers
## Service worker installed but not controlling the page
https://felixgerschau.com/service-worker-lifecycle-update/

1. Before clients.claim()
![](https://felixgerschau.com/static/db3e6b65c7440c80a6124e7b4a9f34f5/29007/service-worker-not-controlling.png){width=200px lazy}
1. After clients.claim()
![](https://felixgerschau.com/static/515778442fc0bfe8424b6b64f608c5ab/29007/service-worker-controlling.png){width=500px}

---

# Problems with service workers
## Service worker activated but not installing
skipWaiting()

---

# Problems with service workers
## Service worker intercepting requests but not caching
skipWaiting()

---
layout: intro
---

# Problem #2
As a user, I want to be able to preload pages, I might use in the future when in offline mode

**Solution:**

- Retrieve all paths that might need to be rendered in advance
- Pass them to a stimulus controller
- Preload them when the user presses the 'Save for Offline use' button

---
layout: full
---

```ruby
module RouteHelper
  def preload_route_paths(route)
    observation_paths = lambda do |observation|
      [
        observation_path(observation),
        edit_observation_path(observation)
      ]
    end

    location_paths = lambda do |location|
      [
        location_path(location),
        edit_location_path(location),
        new_location_observation_path(location)
      ]
    end

    route_paths = [
      route_path(route),
      edit_route_path(route),
      new_route_location_path(route)
    ]

    [
      *route_paths,
      *route.locations.flat_map { |location| location_paths.call(location) },
      *route.locations.includes(:observations).flat_map do |location|
        location.observations.flat_map(&observation_paths)
      end
    ]
  end
end
```

---

```erb{4,12-16}
<div class="accordion-item"
     data-controller="route-item"
     data-route-item-route-id-value="<%= route.id %>"
     data-route-item-preload-paths-value="<%= preload_route_paths(route).join(',') %>">
  <h2 class="accordion-header bg-success-subtle" id="accordion-heading<%= route.id %>">
    <button class="accordion-button collapsed" type="button"
            data-bs-toggle="collapse"
            data-bs-target="#accordion-collapse<%= route.id %>"
            aria-expanded="false"
            aria-controls="accordion-collapse<%= route.id %>">
...
        <% if can? :preload, route %>
          <%= button_tag t('javascript.offline.preloadable_button_text'), class: 'btn btn-success mt-1',
                                                                          data: { route_item_target: 'offlineButton',
                                                                                  action: 'route-item#preload' } %>
        <% end %>
...

```

---

```ts{4-6,14-16}
import { Controller } from '@hotwired/stimulus'

export default class RouteItemController extends Controller<HTMLDivElement> {
  static values = {
    preloadPaths: String
  }
  ...
  async preload() {
    if (this.isPreloaded) return

    this.updateLoadingState('preloading')

    try {
      const paths: string[] = this.preloadPathsValue.split(',')
      const requests = paths.map((preloadPath: string) => fetch(preloadPath))
      await Promise.all(requests)
      this.updateLoadingState('preloaded')
    } catch (error) {
      console.error(error)
      this.updateLoadingState('preloadable')
      this.handlePreloadError()
    } finally {
      this.markAsPreloaded()
    }
  }
  ...
  set preloaded(value: string) {
    const localStorageKey = `preload-route-${this.routeIdValue}`
    localStorage.setItem(localStorageKey, value)
  }
}
```

---
layout: intro
---

# Problem #3
As a user, I want to be able to submit forms while offline, so that I don't have to wait for internet connection.

**Solution:**

- Unless user is online, intercept the submit form event.
- Create a 'RequestQueueItem' table in the IndexedDB if not already present.
- Serialize the formdata and save it to the database.

---

# Define the IndexedDB

```ts
import Dexie from 'dexie'
import { RequestQueueItem } from '../types'

class OfflineDatabase extends Dexie {
  requestQueue!: Dexie.Table<RequestQueueItem, number>

  constructor() {
    super('OfflineDatabase')
    this.version(1).stores({
      requestQueue: '++id, model, data, url, method, token',
    })
  }
}

const database = new OfflineDatabase()
database.open()

export default database

```
---

```ts{all|10-14}
import { Controller } from '@hotwired/stimulus'
import Mustache from 'mustache'
import { RequestQueueItem } from '../types'
import database from '../pwa/database'
import ToastHelper from '../toast_helper'
import { isOnline } from './offline_status_controller'

export default class extends Controller<HTMLFormElement> {
  ...
  async submit(event: Event) { ... }

  private storeRequest(request: RequestQueueItem) { ... }

  private async formDataString(formData: FormData) { ... }
}
```

---

```ts{all|4|8,9,14|10,15,17|12,16,19,20}
export default class extends Controller<HTMLFormElement> {
  ...
  async submit(event: Event) {
    if (isOnline()) return

    event.preventDefault()

    const form = this.element
    const formData = new FormData(form)
    const method = formData.get('_method')?.toString() || 'post'

    const action = (method === 'post') ? 'created' : 'updated'

    const data = await this.formDataString(formData)
    const url = form.action
    const model = Vogelhuesli.I18n.model.observation
    const token = document.querySelector<HTMLMetaElement>('[name="csrf-token"]')!.content

    const locationSelect = document.querySelector<HTMLSelectElement>('#observation_location_id')
    const locationName = locationSelect?.options[locationSelect.selectedIndex].text || ''

    const request = {
      model, data, url, action, method, token, name: locationName,
    }

    await this.storeRequest(request)

    ToastHelper.create('saved-offline', Vogelhuesli.I18n.form.saved_offline, 'info')

    window.location.href = '/offline'
  }

  private storeRequest(request: RequestQueueItem) {
    return database.requestQueue.add(request)
  }

  private async formDataString(formData: FormData) {
    const object: Record<string, FormDataEntryValue> = {}
    formData.forEach(async (value, key) => {
      if (key === '_method') return
      object[key] = value
    })
    return JSON.stringify(object)
  }
}
```

---

```ts{all|4-9,16-18|11|13}
export default class extends Controller<HTMLFormElement> {
  ...
  async submit(event: Event) {
    ...
    const request = {
      model, data, url, action, method, token, name: locationName,
    }

    await this.storeRequest(request)

    ToastHelper.create('saved-offline', Vogelhuesli.I18n.form.saved_offline, 'info')

    window.location.href = '/offline'
  }

  private storeRequest(request: RequestQueueItem) {
    return database.requestQueue.add(request)
  }

  private async formDataString(formData: FormData) {
    const object: Record<string, FormDataEntryValue> = {}
    formData.forEach(async (value, key) => {
      if (key === '_method') return
      object[key] = value
    })
    return JSON.stringify(object)
  }
}
```

---

# Problem 4
As a user, I want to be able to sync submited forms when I am back online, so that my forms get sent and the data gets stored.

**Solution:**
- Offline mode controller
- List of all sync tasks from the indexedDB
- Buttons for syncing / removing sync items
- Feedback when items get synced / requests fail

---
