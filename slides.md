---
layout: cover
theme: the-unnamed
company: Renuo AG
date: 08.12.2023
author: Daniel Bengl
---

# Adapting Ruby on Rails for the Offline User Experience

Created by Daniel Bengl @ Renuo

---
layout: about-me

helloMsg: Hello There!
name: Daniel Bengl (19)
imageSrc: 4CFBBE4B-ACB5-42CD-8458-7B445BD16CF0_1_105_c.jpeg
job: Software Engineering IMS Intern
line1: 
line2: 
social1: github.com/CuddlyBunion341
social2: reddit.com/user/CuddlyBunion341
social3: youtube.com/@cuddlybunion3416
---
---
layout: two-cols
---

# About Vogelhuesli

- Birdhouse management system for nature association Degersheim

**Technologies**
- Ruby on Rails with Stimulus & TypeScript
- Active Storage with S3
- Mapbox
- CanCanCan

**Features**
- Management of birdhouse routes
- Management of birdhouse locations
- Management of birdhouse observations 

::right::

<div style="position: absolute; top: -100px">
  <img src="4072F617-543B-4290-8C36-C3847AD6316A_1_102_a.jpeg">
</div>

<!-- Vogelhuesli is the application is the application I have adapted to offline usage and would like to use as an example for this Beertalk -->
<!-- There are some other features beyond these 3 I listed but these are the foundational ones I needed to make offline usable -->

---
layout: two-cols
---


# Problems and Solutions

Based on the offline requirements for Vogelhüsli

1. Caching already-visited views
1. Preloading of views
1. Offline form submission
1. Synchronizing progress when back online
1. Capybara testing

**Tradeoffs**

- Offline tasks are shown on one view only
- Manual synchronization
- Works best with WiFi turned off


::right::

<div style="position: absolute; top: -100px;">
  <img style="height: 100vh; object-fit: cover;" src="https://images.unsplash.com/photo-1558022103-603c34ab10ce?q=80&w=3271&auto=format&fit=crop&ixlib=rb-4.0.3&ixid=M3wxMjA3fDB8MHxwaG90by1wYWdlfHx8fGVufDB8fHx8fA%3D%3D">
</div>

---

# 1. Caching Already-Visited Views

*"As a user, I want to be able to access the page in offline mode after visiting it in the past"*

- Intercept and cache outgoing HTTP requests with Service Workers
- Caching approaches

**Service Worker API**
- `navigator.serviceWorker.register(...)`
- `fetch` event

**Browser cache APIs**
- `Cache.match(request, options)`
- `Cache.put(request, response)`

---

# Network-First Approach to Caching

What is this approach about and why should we use it?

**Pros**

- No need for cache invalidation
- Resources are always up to date
- It is simple to implement

**Cons**

- Heavier network usage
- Is not suitable for all resources

<img class="absolute" src="https://miro.medium.com/v2/resize:fit:1400/format:webp/0*5l-2aP5-sWYjjnY0.png" alt="medium" width="500px" style="right: 40px; top: 35%; position: absolute">

---

# Registering Service Workers

```js{all}
// companion_script.ts
const registerServiceWorker = async () => {
  if ('serviceWorker' in navigator) {
    try {
      const registration = await navigator.serviceWorker.register('/service-worker.js', {
        scope: '/',
      })
    } catch (error) {
      console.error(`Registration failed with ${error}`)
    }
  }
}
registerServiceWorker()
```

---

# Implementing the Service Worker
This is my implementation of the service worker for the "Vogelhuesli" project

https://vogelhuesli.renuoapp.ch/service-worker.js

```js
class CacheManager {
  constructor(cacheName, fallbackUrl) {...}

  async putToCache(request, response) {...}

  async getFromCache(request) {...}

  async retrieveOrStore(event) {...}
}

const cacheManager = new CacheManager('v1', '/offline_fallback')
```

- Or use a library like https://developer.chrome.com/docs/workbox

---

# Intercepting Requests
1. `fetch` is called by the client when loading a page / asset / performing ajax call.
2. `fetch` event is intercepted by service worker.
3. `event.respondWith` responds with a response object, either from cache or network.
6. `cacheManager.retrieveOrStore(event)` handles the network-first caching

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

# Filtering Requests
- Chrome extensions cannot be cached
- Mapbox tiles get cached automatically
- POST / PUT / PATCH / DELETE requests should not be cached
```js
  async retrieveOrStore({ request }) {
    const filters = [
      (url) => url.startsWith('chrome-extension://'),
      (url) => /api.mapbox.com.+vector\.pbf/.test(url),
      ...
    ]

    if (request.method !== 'GET' || filters.some((filter) => filter(request.url))) {
      return fetch(request)
    }
    ...
  }
```

---

# Retrieving Response from the Network
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

# Falling-Back to Cache

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
layout: two-cols
---

# 2. Preloading of Views
*"As a walker, I want to be able to preload pages, I might use in the future when in offline mode."*

**Solution:**

- Retrieve all paths that might need to be rendered in advance
- Pass them to a stimulus controller
- Preload them when the user presses the 'Save for Offline use' button

<br>

<svg class="logo" width="370" height="60" viewBox="0 0 370 60" xmlns="http://www.w3.org/2000/svg">
  <g class="logo__icon" fill="#D9C6A4">
    <path d="M1.76 1.85885e-08H58.2399C58.7016 -6.70275e-05 59.1447 0.181237 59.474 0.50484C59.8032 0.828443 59.9921 1.26844 59.9999 1.73V7.37823C59.9999 7.87239 59.9999 8.11947 59.9038 8.30822C59.8192 8.47424 59.6842 8.60923 59.5182 8.69382C59.3294 8.78999 59.0823 8.78999 58.5882 8.78999H45.92C44.7722 8.83556 43.6502 9.14343 42.64 9.68999L15.64 25.55C14.6261 26.0964 13.5008 26.4042 12.35 26.45H1.41176C0.9176 26.45 0.670518 26.45 0.481773 26.3538C0.315747 26.2692 0.180765 26.1342 0.0961706 25.9682C0 25.7795 0 25.5324 0 25.0382V17.2717C0 16.7776 0 16.5305 0.0961706 16.3418C0.180765 16.1757 0.315747 16.0408 0.481773 15.9562C0.670518 15.86 0.9176 15.86 1.41176 15.86H12.35C13.5001 15.9106 14.6243 16.2181 15.64 16.76L18.64 18.51C19.1037 18.7494 19.6181 18.8743 20.14 18.8743C20.6619 18.8743 21.1762 18.7494 21.64 18.51L25.48 16.25C25.6438 16.1661 25.7812 16.0386 25.8772 15.8815C25.9732 15.7245 26.024 15.544 26.024 15.36C26.024 15.1759 25.9732 14.9955 25.8772 14.8384C25.7812 14.6814 25.6438 14.5539 25.48 14.47L17.4 9.71999C16.3879 9.17798 15.2669 8.8704 14.12 8.81999H1.41176C0.9176 8.81999 0.670518 8.81999 0.481773 8.72382C0.315747 8.63923 0.180765 8.50424 0.0961706 8.33822C0 8.14947 0 7.90239 0 7.40823V1.76C0 1.29322 0.185428 0.845555 0.515492 0.515492C0.845555 0.185428 1.29322 1.85885e-08 1.76 1.85885e-08Z"></path>
    <path d="M47.65 15.8799C46.4998 15.9305 45.3757 16.238 44.36 16.7799L17.36 32.6499C16.3479 33.1919 15.2269 33.4994 14.08 33.5499H1.41176C0.9176 33.5499 0.670518 33.5499 0.481773 33.646C0.315747 33.7306 0.180765 33.8656 0.0961706 34.0316C0 34.2204 0 34.4674 0 34.9616V42.7281C0 43.2222 0 43.4693 0.0961706 43.6581C0.180765 43.8241 0.315747 43.9591 0.481773 44.0437C0.670518 44.1398 0.9176 44.1398 1.41176 44.1398H12.35C13.5001 44.0892 14.6243 43.7817 15.64 43.2398L42.64 27.3699C43.652 26.8278 44.773 26.5203 45.92 26.4699H58.5882C59.0823 26.4699 59.3294 26.4699 59.5182 26.3737C59.6842 26.2891 59.8192 26.1541 59.9038 25.9881C59.9999 25.7993 59.9999 25.5523 59.9999 25.0581V17.2916C59.9999 16.7975 59.9999 16.5504 59.9038 16.3616C59.8192 16.1956 59.6842 16.0606 59.5182 15.976C59.3294 15.8799 59.0823 15.8799 58.5882 15.8799H47.65Z"></path>
    <path d="M47.65 33.56C46.4998 33.6106 45.3757 33.9182 44.36 34.46L17.36 50.32C16.3497 50.8666 15.2277 51.1744 14.08 51.22H1.41176C0.9176 51.22 0.670518 51.22 0.481773 51.3162C0.315747 51.4008 0.180765 51.5358 0.0961706 51.7018C0 51.8905 0 52.1376 0 52.6318V58.28C0.0129646 58.739 0.203789 59.175 0.532182 59.4959C0.860576 59.8168 1.30083 59.9976 1.76 60H58.2399C58.7042 59.9974 59.1488 59.8125 59.4781 59.4852C59.8073 59.1578 59.9947 58.7142 59.9999 58.25V52.6018C59.9999 52.1076 59.9999 51.8605 59.9038 51.6718C59.8192 51.5058 59.6842 51.3708 59.5182 51.2862C59.3294 51.19 59.0823 51.19 58.5882 51.19H45.88C44.7322 51.1444 43.6102 50.8366 42.6 50.29L34.52 45.55C34.3536 45.4671 34.2136 45.3394 34.1157 45.1813C34.0179 45.0232 33.966 44.8409 33.966 44.655C33.966 44.4691 34.0179 44.2868 34.1157 44.1287C34.2136 43.9706 34.3536 43.843 34.52 43.76L38.36 41.51C38.8237 41.2706 39.3381 41.1457 39.86 41.1457C40.3819 41.1457 40.8962 41.2706 41.36 41.51L44.36 43.25C45.3739 43.7964 46.4991 44.1042 47.65 44.15H58.5882C59.0823 44.15 59.3294 44.15 59.5182 44.0538C59.6842 43.9693 59.8192 43.8343 59.9038 43.6682C59.9999 43.4795 59.9999 43.2324 59.9999 42.7383V34.9718C59.9999 34.4776 59.9999 34.2305 59.9038 34.0418C59.8192 33.8758 59.6842 33.7408 59.5182 33.6562C59.3294 33.56 59.0823 33.56 58.5882 33.56H47.65Z"></path>
  </g>
  <g class="logo__wordmark" fill="#fff">
    <path d="M363.73 6.09338C365.043 6.29182 366.288 6.80629 367.358 7.59253C368.357 8.37618 369.067 9.46955 369.377 10.7008C369.772 12.4038 369.772 14.1748 369.377 15.8778L368.837 19.0261L355.604 20.6951L356.344 16.2476H348.578L347.149 24.523H355.634C357.361 24.5096 359.086 24.64 360.791 24.9127C362.164 25.1068 363.472 25.6203 364.609 26.4119C365.655 27.1652 366.416 28.25 366.768 29.4901C367.152 31.2012 367.152 32.9761 366.768 34.6872L365.209 43.6821C364.924 45.5578 364.301 47.3661 363.37 49.0191C362.61 50.3318 361.552 51.4478 360.282 52.2773C359.005 53.0816 357.582 53.6251 356.094 53.8763C354.361 54.1698 352.605 54.3103 350.847 54.2961H340.633C338.905 54.3138 337.178 54.1733 335.476 53.8763C334.153 53.6701 332.909 53.1185 331.868 52.2773C330.912 51.4324 330.265 50.2929 330.029 49.0391C329.733 47.2717 329.774 45.4644 330.149 43.7121L330.689 40.4939L344.161 38.7249L343.281 43.7821H351.926L353.466 34.987H344.98C343.235 35.0073 341.491 34.8601 339.773 34.5473C338.423 34.3357 337.15 33.7817 336.075 32.9382C335.099 32.0974 334.425 30.9601 334.157 29.7C333.823 27.923 333.857 26.0964 334.256 24.3331L335.716 15.9178C335.977 14.0865 336.595 12.3243 337.535 10.7308C338.317 9.46015 339.385 8.38915 340.653 7.60253C341.938 6.84202 343.356 6.33008 344.831 6.09338C346.482 5.82458 348.154 5.6942 349.828 5.7036H358.823C360.467 5.69 362.109 5.82042 363.73 6.09338ZM104.837 6.09344C106.15 6.29188 107.395 6.80635 108.465 7.59259C109.464 8.37624 110.174 9.46961 110.484 10.7008C110.879 12.4039 110.879 14.1748 110.484 15.8779L109.944 19.0261L96.7117 20.6952L97.4513 16.2477H89.6857L88.2565 24.523H96.7417C98.4685 24.5097 100.194 24.6401 101.899 24.9128C103.271 25.1068 104.579 25.6203 105.717 26.4119C106.763 27.1653 107.523 28.2501 107.875 29.4902C108.259 31.2013 108.259 32.9762 107.875 34.6873L106.316 43.6822C106.031 45.5578 105.408 47.3661 104.477 49.0191C103.717 50.3319 102.659 51.4478 101.389 52.2773C100.112 53.0817 98.6892 53.6252 97.2014 53.8764C95.4681 54.1699 93.7123 54.3103 91.9544 54.2962H81.7401C80.0119 54.3139 78.2857 54.1734 76.5831 53.8764C75.2607 53.6701 74.0161 53.1185 72.9751 52.2773C72.0191 51.4324 71.372 50.293 71.1361 49.0391C70.8406 47.2717 70.8813 45.4645 71.2561 43.7122L71.7958 40.494L85.2682 38.725L84.3886 43.7821H93.0338L94.5729 34.9871H86.0877C84.3419 35.0074 82.5983 34.8601 80.8806 34.5473C79.5308 34.3358 78.2576 33.7818 77.1827 32.9382C76.2065 32.0975 75.5325 30.9601 75.2638 29.7001C74.9298 27.9231 74.9638 26.0965 75.3638 24.3331L76.8229 15.9179C77.0844 14.0866 77.7024 12.3243 78.6419 10.7308C79.4245 9.46021 80.492 8.38921 81.7601 7.60258C83.0458 6.84208 84.4629 6.33014 85.9378 6.09344C87.5897 5.82463 89.2613 5.69426 90.9349 5.70366H99.9299C101.574 5.69006 103.216 5.82048 104.837 6.09344ZM115.761 6.24855H150.991L148.882 18.2917H138.158L131.981 53.7517H118.209L124.386 18.2917H113.662L115.761 6.24855ZM154.928 6.24855L146.643 53.7517H160.425L168.701 6.24855H154.928ZM192.008 25.0779L205.381 6.24855H217.963L209.688 53.7517H195.876L200.223 28.9157L188.48 45.8162L182.264 29.2555L177.996 53.7517H166.083L174.358 6.24855H184.642L192.008 25.0779ZM245.415 6.24855L239.179 42.1183H231.063L237.31 6.24855H223.547L217.031 43.7373C216.652 45.4935 216.624 47.3076 216.951 49.0743C217.211 50.3401 217.891 51.4811 218.88 52.3125C219.973 53.1474 221.258 53.6939 222.618 53.9016C224.337 54.2062 226.08 54.3501 227.825 54.3313H238.109C239.927 54.3469 241.743 54.2031 243.536 53.9016C245.059 53.6605 246.52 53.1204 247.834 52.3125C249.103 51.491 250.162 50.3816 250.922 49.0743C251.839 47.4179 252.452 45.6103 252.731 43.7373L259.227 6.24855H245.415ZM264.791 6.24855H278.573L272.397 41.7085H287.189L285.09 53.7517H256.516L264.791 6.24855ZM319.011 6.24855L312.774 42.1183H304.659L310.905 6.24855H297.143L290.627 43.7373C290.247 45.4935 290.22 47.3076 290.547 49.0743C290.807 50.3402 291.486 51.4812 292.476 52.3125C293.569 53.1474 294.854 53.6939 296.214 53.9016C297.932 54.2062 299.675 54.3501 301.421 54.3313H311.705C313.523 54.3469 315.339 54.2031 317.132 53.9016C318.655 53.6605 320.116 53.1204 321.429 52.3125C322.699 51.491 323.757 50.3816 324.518 49.0743C325.435 47.4179 326.048 45.6103 326.327 43.7373L332.823 6.24855H319.011Z"></path>
  </g>
</svg>

::right::

![alt text](<CleanShot 2024-03-06 at 00.30.45@2x.png>)

---
layout: full
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
layout: full
---

```ts{1-99|4-6,14-16|27-30}{maxHeight:'100%'}
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
layout: two-cols
---

<h1 style="max-width: 69%;">3. Offline Form Submission</h1>

*"As a walker, I want to be able to submit forms when offline, so that I can create observations in bad network conditions.*

**Solution:**

- Unless user is online, intercept the submit form event.
- Create a 'RequestQueueItem' table in the IndexedDB if not already present.
- Serialize the formdata and save it to the database.

::right::
<img src="image.png" style="position: absolute; top: -100px;">

<!-- Dormice -->

---

# What is IndexedDB?
- https://developer.mozilla.org/en-US/docs/Web/API/IndexedDB_API
- Like local storage but for storing records
- Structured like a database with tables and indicies
- Terrible Asynchronous API
- Large storage capacity
- Persistent storage
- Service Worker accesibility

---

# Define the IndexedDB
Why are we using Dexie.js again?

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
layout: two-cols
---

# 4. Synchronisation
*"As a user, I want to be able to sync submitted forms when I am back online, so that my forms get sent and the data gets stored."*

**Solution:**
- Offline mode controller
- Offline sync controller for individual sync items
- List of all sync tasks from the indexedDB
- Buttons for syncing / removing sync items
- Feedback when items get synced / requests fail

::right::

<img src="CleanShot 2024-03-06 at 00.52.45@2x-1.png" style="height: 25%; overflow: hidden; width: 100%; object-fit: cover; object-position: center top;">

![alt text](<CleanShot 2024-03-06 at 00.56.58@2x.png>)

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

# 5. Testing: RSpec Helpers

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

# 5. Testing: Service Worker Messages
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

# 5. Testing: Simulating Offline Mode in Service Worker
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
- Offline mode can be "easy", depending on the scope
- Service workers are a powerful tool for offline mode
- They can be used to cache resources, intercept requests and handle background sync
- They are not easy to implement and require a lot of testing
- You can emulate `offline` mode with service worker state and messages

## Topics not discussed in this talk:
- Service Worker Lifecycle
- Background sync
- Preloading of static resources
- Preloading of fingerprinted resources
- Cache invalidation

---

# Thank you for your attention!
Have a nice rest of the day!

---

# Sources
- https://developer.mozilla.org/en-US/docs/Web/Progressive_web_apps
- https://alicia-paz.medium.com/make-your-rails-app-work-offline-part-1-pwa-setup-3abff8666194
- https://alicia-paz.medium.com/make-your-rails-app-work-offline-part-2-caching-assets-and-adding-an-offline-fallback-334729ade904
- https://alicia-paz.medium.com/make-your-rails-app-work-offline-part-3-crud-actions-with-indexeddb-and-stimulus-ad669fe0141c
