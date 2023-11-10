Global and Per-Route Guards
===========================

In the last lesson we implemented a progress bar using component navigation guards on a single component. In this lesson we’ll apply the progress bar to our whole application using global navigation guards, and then look at how we can use per-route navigation guards to move our API call into our router.

Global Route Guards
-------------------

Our progress bar would make sense to appear on all our pages, to improve overall user experience. However, it would be a pain to add `NProgress.start()` and `NProgress.done()` to all our components. Instead we can use Vue Router Global Navigation Guards, `beforeEach` and `afterEach` inside our router file.

📃 **/src/router/index.js**

    import NProgress from 'nprogress'
    ...
    
    router.beforeEach(() => {
      NProgress.start()
    })
    
    router.afterEach(() => {
      NProgress.done()
    })
    
    export default router
    

I put these at the bottom of my router file (with the exception of the import statement). As you can likely intuit, this code will now run before navigating to each route, and after each route.

However, to make this work, we need to remove the progress bar `start` and `done` code from `beforeRouteEnter` and `beforeRouteUpdate`, since these will be called from our Global Route Guards. We also need to add `return` to the `beforeRouteUpdate` API call. Otherwise our router won’t wait for the API call to finish before calling `afterEach`. Here’s what it ends up looking like:

      beforeRouteEnter(routeTo, routeFrom, next) {
        EventService.getEvents(2, parseInt(routeTo.query.page) || 1)
          .then(response => {
            next(comp => {
              comp.events = response.data
              comp.totalEvents = response.headers['x-total-count']
            })
          })
          .catch(() => {
            next({ name: 'NetworkError' })
          })
      },
      beforeRouteUpdate(routeTo) {
        return EventService.getEvents(2, parseInt(routeTo.query.page) || 1) // <---
          .then(response => {
            this.events = response.data
            this.totalEvents = response.headers['x-total-count']
          })
          .catch(() => {
            return { name: 'NetworkError' }
          })
      },
    

That’s all we have to do, and now our webpage uses our progress bar for every route.

![https://firebasestorage.googleapis.com/v0/b/vue-mastery.appspot.com/o/flamelink%2Fmedia%2F01-all-page-progress-bar-1280.gif?alt=media&token=131e483e-c386-4f1e-be93-fb7d77115a34](https://firebasestorage.googleapis.com/v0/b/vue-mastery.appspot.com/o/flamelink%2Fmedia%2F01-all-page-progress-bar-1280.gif?alt=media&token=131e483e-c386-4f1e-be93-fb7d77115a34)

You might have noticed that our `beforeRouteUpdate` needs a return, while our `beforeRouteEnter` does not. This confused me too. It turns out that if we’re using `next` to signify to Vue Router when to continue navigation, we don’t have to return the promise. Makes sense right?

Per-Route Guards
----------------

When we click on an event, we do have the route guard, but it loads instantly without waiting for the API call to return. To fix this we could move our API call inside `beforeRouteEnter`, but there’s another way we use a route guard I want to teach you, this time in our router file. Let’s start by reviewing our `layout.vue`, where we make our API call to fetch a single event.

📃 **/src/views/event/layout.vue**

    <template>
      <div v-if="event">
        <h1>{{ event.title }}</h1>
        <div id="nav">
          <router-link :to="{ name: 'EventDetails' }">Details</router-link>
          |
          <router-link :to="{ name: 'EventRegister' }">Register</router-link>
          |
          <router-link :to="{ name: 'EventEdit' }">Edit</router-link>
        </div>
        <router-view :event="event" />
      </div>
    </template>
    
    <script>
    import EventService from '@/services/EventService.js'
    export default {
      props: ['id'],
      data() {
        return {
          event: null
        }
      },
      created() {
        EventService.getEvent(this.id)
          .then(response => {
            this.event = response.data
          })
          .catch(error => {
            if (error.response && error.response.status == 404) {
              this.$router.push({
                name: '404Resource',
                params: { resource: 'event' }
              })
            } else {
              this.$router.push({ name: 'NetworkError' })
            }
          })
      }
    }
    </script>
    

Let’s try moving the API call from here into the router, and only load this component if the API call is successful. We can do this by using the `beforeEnter` per-route guard, that gets run before the component is loaded. It’ll look something like this:

📃 **/src/router/index.js**

    ...
    {
        path: '/events/:id',
        name: 'EventLayout',
        props: true,
        component: EventLayout,
        beforeEnter: to => {
          // <--- put API call here
        }
        children: [ ... ]
    }
    

Unfortunately I can’t just paste the API call into here. There will be a few problems. First, we don’t have access to `this` from the component yet. We will need to return the API call so that our component doesn’t get loaded until the API call returns, change how we get the event id, how we redirect on errors, and how we set the event property passed into the component. Let’s deal with the first three issues first.

    import EventService from '@/services/EventService.js'
    ...
    {
        path: '/events/:id',
        name: 'EventLayout',
        props: true,
        component: EventLayout,
        beforeEnter: to => {
          return EventService.getEvent(to.params.id) // Return and params.id
          .then(response => {
            // Still need to set the data here
          })
          .catch(error => {
            if (error.response && error.response.status == 404) {
              return { // <--- Return
                name: '404Resource',
                params: { resource: 'event' }
              }
            } else {
              return { name: 'NetworkError' } // <--- Return
            }
          })
        }
        children: [ ... ]
        ...
    }
    

So good so far, but now we need to be able to get the data returned from the API call into the component itself. My initial thought was to set `to.params.event = response.data`, but this is a read only object. The solution we will implement will use our Global Store (`GStore`), which we first created in our [Flash Messages](https://www.vuemastery.com/courses/touring-vue-router/flash-messages) lesson. This is quite similar to how we might use Vuex to solve this issue, which is another type of Global Store. I’m not going to use Vuex here, because I don’t want to assume you know how to use it.

Here’s the current state of our `main.js`

📃 **/src/main.js**

    import { createApp, reactive } from 'vue'
    import App from './App.vue'
    import router from './router'
    import store from './store'
    import 'nprogress/nprogress.css'
    
    const GStore = reactive({ flashMessage: '' })
    
    createApp(App)
      .use(store)
      .use(router)
      .provide('GStore', GStore)
      .mount('#app')
    

See that `GStore` reactive object? Let’s do two things with it. Let’s move it into its own file and add a new reactive property called `event` to store the results from our API call.

📃 **/src/store/index.js**

    import { reactive } from 'vue'
    
    export default reactive({ flashMessage: '', event: null })
    

In our project we already had `/store/index.js` file, and I just replaced its contents with what you see above. Notice how we have two reactive properties: the `flashMessage` and the `event`. Now back inside our `main.js` we can use this file:

📃 **/src/main.js**

    import { createApp } from 'vue' // <--- removed reactive
    import App from './App.vue'
    import router from './router'
    import GStore from './store' // <--- Modified this import line
    import 'nprogress/nprogress.css'
    
    createApp(App)
      .use(router)  // <--- Removed use(store), because we're not using Vuex
      .provide('GStore', GStore)
      .mount('#app')
    

Why did we move `GStore` into its own file? So we can access it from inside our router:

📃 **/src/router/index.js**

    ...
    import GStore from '@/store'
    
    const routes = [
      ...
      {
        path: '/events/:id',
        name: 'EventLayout',
        component: EventLayout,  // <-- We removed the props: true.
        beforeEnter: to => {
          return EventService.getEvent(to.params.id)
            .then(response => {
              GStore.event = response.data // <--- Store the event
            })
            .catch(error => {
              if (error.response && error.response.status == 404) {
                return {
                  name: '404Resource',
                  params: { resource: 'event' }
                }
              } else {
                return { name: 'NetworkError' }
              }
            })
        },
        children: [ ... ]
        ...
    

Great, so now our API call will set the event on the Global Storage `GStore`, and we’ll be able to have our component access it from there. We no longer will be sending in the [event.id](http://event.id) as a prop, so we can remove `props: true`. Now back in our `layout.js` we can inject the `GStore`.

📃 **/src/views/event/layout.vue**

    <template>
      <div v-if="GStore.event">
        <h1>{{ GStore.event.title }}</h1>
        <div id="nav">
          <router-link :to="{ name: 'EventDetails' }">Details</router-link>
          |
          <router-link :to="{ name: 'EventRegister' }">Register</router-link>
          |
          <router-link :to="{ name: 'EventEdit' }">Edit</router-link>
        </div>
        <router-view :event="GStore.event" />
      </div>
    </template>
    
    <script>
    export default {
      inject: ['GStore']
    }
    </script>
    

Notice that we’ve removed the prop, the API call, added the `inject` to bring in `GStore`, and we’re referencing the event in the template as `GStore.event`, instead of just `event`. Now everything is working as before, except our API call is at the router layer.

![https://firebasestorage.googleapis.com/v0/b/vue-mastery.appspot.com/o/flamelink%2Fmedia%2F02-all-page-progress-bar-1280.gif?alt=media&token=d5a4feff-4f28-4672-bc12-14919c41f3e4](https://firebasestorage.googleapis.com/v0/b/vue-mastery.appspot.com/o/flamelink%2Fmedia%2F02-all-page-progress-bar-1280.gif?alt=media&token=d5a4feff-4f28-4672-bc12-14919c41f3e4)

Calling Order
-------------

You might be wondering when these different routing guards are called. Good question. Here’s what it looks like:

![https://firebasestorage.googleapis.com/v0/b/vue-mastery.appspot.com/o/flamelink%2Fmedia%2F3.1618588875881.jpg?alt=media&token=abd26acf-90f6-4a39-8441-89cedd353ff2](https://firebasestorage.googleapis.com/v0/b/vue-mastery.appspot.com/o/flamelink%2Fmedia%2F3.1618588875881.jpg?alt=media&token=abd26acf-90f6-4a39-8441-89cedd353ff2)

Notice how the callbacks on the right are inside our component, and the ones on the left are inside the router. Also, the difference between `beforeResolve` and `afterEach`, is that the navigation can only be cancelled inside `beforeResolve`.

**⏪ Let’s Revue**
-----------------

In this lesson we learned how to use both Global Guards and Per-Route Guards which get called upon navigation before our component is created. These can be very useful when we need to fetch data from an API, or when we’re doing any type of authorization.

### Lesson Resources

##### Source Code:

*   [Starting Code](https://github.com/Code-Pop/Touring-Vue-Router/tree/L10-start)
    
    Node v16 required
    
*   [Ending Code](https://github.com/Code-Pop/Touring-Vue-Router/tree/L10-end)
