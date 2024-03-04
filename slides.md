---
layout: cover
theme: the-unnamed
company: Renuo AG
date: 08.12.2023
author: Daniel
---

# Adapting Ruby on Rails for the Offline User Experience

<!-- A guide on how to use IndexedDB, Service Workers and Stimulus -->

Created by Daniel Bengl @ Renuo

---
layout: about-me

helloMsg: Hello!
name: Daniel Bengl
imageSrc: 4CFBBE4B-ACB5-42CD-8458-7B445BD16CF0_1_105_c.jpeg
job: Software Engineering IMS Intern
line1: 
line2: 
social1: github.com/CuddlyBunion341
social2: reddit.com/user/CuddlyBunion341
---

---

# Why should YOU care?

- You might be interested in implementing offline mode yourself.
- Your clients might need to use the application in bad internet conditions.
- You want to learn something about request / response caching?

---

# Problems and Solutions
Based on my experience with Offline mode for the "Vogelhuesli" project

1. Caching already visited views
1. Preloading of views
1. Submiting forms when offline
1. Task syncing when online
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
- fetch event

## Browser cache APIs
- [https://developer.mozilla.org/en-US/docs/Web/API/Cache](https://developer.mozilla.org/en-US/docs/Web/API/Cache)
- Cache.match(request, options)
- Cache.put(request, response)

---
layout: quote
---

# What is a service worker?

<!-- > A service worker ios an event-driven worker registered against an origin and a path. It takes the form of a JavaScript file that can control the web-page/site that it is associated with, intercepting and modifying navigation and resource requests, and caching resources in a very granular fashion to give you complete control over how your app behaves in certain situations (the most obvious one being when the network is not available).\ -->
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
# Important events

::left::

**Registration**
  - Browser is informed about the existence and the script location of the service worker.

**Installation**
  - Precache static assets / essential resources

**Activation**
  - Cache management (delete old caches)
  - Update client data

::right::

**Fetch**
  - Cache-first Strategy: Check if resource is in cache, fetch from network otherwise.
  - Network-first Strategy: Attempt to fetch ressource from network and cache, refer to cache otherwise.
  - Offline Fallback Strategy: Respond with predefined fallback / custom offline page when cache and network is unavailable.

**Message**
  - Communication between client and service worker

---

# Registering Service Workers

```js{all|5-7}
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
layout: two-cols-header
---

# Network first approach to caching
What is this approach about and why should we use it?

::left::

**Pros**

- No need for cache invalidation
- Resources are always up to date
- It is simple to implement

**Cons**

- Heavier network usage
- Is not suitable for all resources

::right::

<img src="https://miro.medium.com/v2/resize:fit:1400/format:webp/0*5l-2aP5-sWYjjnY0.png" width="500px">

---

# Implementing the Service Worker
This is my implementation of the service worker for the "Vogelhuesli" project.

```js
class CacheManager {
  constructor(cacheName, fallbackUrl) {...}

  async putToCache(request, response) {...}

  async getFromCache(request) {...}

  async retrieveOrStore(event) {...}
}

const cacheManager = new CacheManager('v1', '/offline_fallback')
```

---

# Intercepting requests
1. `fetch` is called by the client when loading a page / asset / performing ajax call.
2. `fetch` event is intercepted by service worker.
3. `event.respondWith` responds with a response object, either from cache or network.
6. `cacheManager.retrieveOrStore(event)` handles the network first Caching

```js
FetchEvent:
+ request
+ respondWith()
+ waitUntil()
```

```js
const cacheManager = new CacheManager('v1')

self.addEventListener('fetch', (event) => {
  event.respondWith(cacheManager.retrieveOrStore(event))
})
```

---

# Filtering requests
- Chrome extensions cannot be cached
- POST / PUT / PATCH requests should not be cached
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

# Retrieving response from the network
Also cache the responses...

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

# Falling back to cache

What if the response is not present in the cache?

```js
  async retrieveOrStore({ request })
    ...
    } catch (error) {
        const cachedResponse = await this.getFromCache(request)
        if (cachedResponse) {
            return cachedResponse
        }

        const fallbackResponse = await this.getFromCache(this.fallbackUrl)
        if (fallbackResponse) return fallbackResponse

        return new Response({status: 503})
    }
}
```

---
layout: cover
---

# Service Worker problems
Some problems with service workers I encountered while developing "Vogelhuesli"
1. Service worker installed but not intercepting requests
1. Resource is present in the cache, but is not being retrieved

---

# Installed but not intercepting requests
https://felixgerschau.com/service-worker-lifecycle-update/

`clients.claim()`
affects first-page visit.

```js
self.addEventListener("install", (event) => {
  self.skipWaiting()
  event.waitUntil(clients.claim());
})
```

Before:
<img src="https://felixgerschau.com/static/db3e6b65c7440c80a6124e7b4a9f34f5/29007/service-worker-not-controlling.png" width="500px">
After:
<img src="https://felixgerschau.com/static/515778442fc0bfe8424b6b64f608c5ab/29007/service-worker-controlling.png" width="500px">

---

# Resource present, response absent
Service worker cache seems to include the resource, but does not retrieve it when offline.

Troubleshooting:
Logging the cache keys and the cache contents.

```js
  async logCacheContents() {
    const cacheKeys = await caches.keys()
    console.log('cacheKeys', cacheKeys)
    const cache = await caches.open(this.cacheName)
    const cacheContents = await cache.keys()
    console.log('cacheContents', cacheContents)
  }
```

The problem:
- The headers of the response (for resource preloading at least) were different from the headers of the request.
- The service worker was not able to match the request with the response.

---

# Resource present, response absent
Service worker cache seems to include the resource, but does not retrieve it when offline.

Solution:
- Use less restrictive cache matching options

```js
  async getFromCache(request) {
    const cache = await caches.open(this.cacheName)
    const matchOptions = {
      ignoreVary: true,
    }
    return cache.match(request, matchOptions)
  }
```

---
layout: intro
---

# Problem #2
As a user, I want to be able to preload pages, I might use in the future when in offline mode.

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

    location_paths = lambda { |location| ... }

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

# Action view
This is a code snippet from the _route_list_item.html.erb partial.

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

# Stimulus controller
This is the stimulus controller for the route item.

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

# What is IndexedDB?
- Like local storage but for storing records
- Structured like a database with tables and indexes
- Key-value store
- Asynchronous API
- https://developer.mozilla.org/en-US/docs/Web/API/IndexedDB_API
- Large storage capacity

---

# Define the IndexedDB
Why are we using Dexie.js?

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

# Define the stimulus controller

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

# Intercept the submit event

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

# Store the request in the IndexedDB

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
As a user, I want to be able to sync submitted forms when I am back online, so that my forms get sent and the data gets stored.

**Solution:**
- Offline mode controller
- Offline sync controller for individual sync items
- List of all sync tasks from the indexedDB
- Buttons for syncing / removing sync items
- Feedback when items get synced / requests fail

---

# Sync item controller
- Import necessary modules and types
- Declare target elements (`syncButtonTarget` and `removeButtonTarget`)
- Declare value `itemId` of type `number`
- Implement static method `updateSyncButton`
- Implement `connect` method
- Implement `onSync` method
- Implement `onRemove` method
- Implement private method `sendRequest`
- Implement private method `deleteRequestQueueItem`
- Implement private method `removeElement`
- Implement private method `getRequestQueueItem`
- Implement private method `updateSyncButton`

---

# Item syncing
Deserialize request queue items, retrieve relevant data and perform fetch request

```ts
private sendRequest(item: RequestQueueItem) {
    const formData = new FormData()
    Object.entries(JSON.parse(data) as Record<string, string>).forEach((element) => {
        const [key, value] = element
        formData.append(key, value)
    })
    return fetch(url, {
        method,
        headers: {
            'X-CSRF-Token': token,
            credentials: 'include',
        },
        body: formData,
    })
}
```
---
layout: cover
---

# How to test offline functionality?
The better question is how to properly emulate offline mode for selenium specs...

---

# Testing offline mode

```ruby
module Helpers
  module Offline
    def enter_offline_mode
      page.driver.browser.network_conditions = { offline: true, latency: 0, throughput: 0 }
      page.evaluate_script('navigator.serviceWorker.controller.postMessage("EMULATE_OFFLINE")')
    end

    def exit_offline_mode
      page.driver.browser.network_conditions = { offline: false, latency: 0, throughput: 0 }
      page.evaluate_script('navigator.serviceWorker.controller.postMessage("STOP_EMULATING_OFFLINE")')
    end

    def wait_for_service_worker
      timer_end = Time.zone.now + 5
      while page.evaluate_script('navigator.serviceWorker.controller').nil?
        sleep(0.1)
        if Time.zone.now > timer_end
          raise 'Service worker not found'
        end
      end
    end
  end
end
```

---

# Testing offline mode
Message event listener in the service worker to emulate offline mode

```js

self.addEventListener('message', (event) => {
  console.log('[Serviceworker]', 'Message!', event)
  switch (event.data) {
    case 'CLEAR_CACHE':
      cacheManager.clearCache()
      break
    case 'EMULATE_OFFLINE':
      self.emulateOfflineState = true
      break
    case 'STOP_EMULATING_OFFLINE':
      self.emulateOfflineState = false
      break
    default:
      throw new Error(`Unknown message: ${event.data}`)
  }
})

```
---

# Testing offline mode
Emulate offline mode in the service worker

```js
...
try {
    if (self.emulateOfflineState) throw new Error('Emulating offline state!')
    console.log('[ServiceWorker]', 'not emulating offline!!')

    const networkResponse = await fetch(request)
    await this.putToCache(request, networkResponse.clone())
    return networkResponse
} catch (error) {
...
```
---

# Final thoughts
- Service workers are a powerful tool for offline mode
- They can be used to cache resources, intercept requests and handle background sync
- They are not easy to implement and require a lot of testing
- They are not supported by all browsers
- You can emulate `offline` mode with service worker state and messages

## Topics not discussed in this talk:
- Background sync
- Preloading of static resources
- Preloading of fingerprinted resources
- Cache invalidation
- Parallel service workers

---

# Thank you for your attention!
Have a nice day!
