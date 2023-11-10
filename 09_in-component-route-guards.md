In-Component Route Guards
=========================

In this lesson we’ll use **In-Component Route Guards** to show a progress bar and only load the component if it’s successfully returned from our API. The goal is to provide a better user experience.

**🛑 Problem: When our API is slow our page looks broken**
----------------------------------------------------------

At the moment EventList looks like this when it loads slowly:

![https://firebasestorage.googleapis.com/v0/b/vue-mastery.appspot.com/o/flamelink%2Fmedia%2F1.1615823301385.gif?alt=media&token=cb880d53-0440-4e31-8c70-2311a56cb0a9](https://firebasestorage.googleapis.com/v0/b/vue-mastery.appspot.com/o/flamelink%2Fmedia%2F1.1615823301385.gif?alt=media&token=cb880d53-0440-4e31-8c70-2311a56cb0a9)

We need to let our users know the data is on the way, and have something happen when they click a link that requires an API call.

If you’re building the event app alongside us, just know that after we install the NProgress bar, we’ll be showing you two different ways to implement it.

*   In-component route guards (this lesson)
*   Global and per-route guards (next lesson)

Each of these solutions are 100% worth learning, as they’ll teach you additional Vue syntax.

**☑️ Solution : In-Component Route Guards**
-------------------------------------------

We have four steps to implement our solution:

1.  Move the API call into our `beforeRouteEnter` hook, so we ensure the API call is successful before we load the component.
2.  Install `nprogress` progress bar library.
3.  Start the progress bar when routing to the component.
4.  When API returns finish progress bar.
5.  Ensure pagination works properly with the progress bar.

Let’s jump in:

* * *

1\. Move API call into an In-Component Route Guard
--------------------------------------------------

In order to do this, we’ll need a short introduction to three new lifecycle hooks provided by Vue Router.

Vue gives us many different component lifecycle hooks, like `created()`, `mounted()`, `updated()`, etc. When using Vue Router, we get three more component hooks called [**Route Navigation Guards**](https://router.vuejs.org/guide/advanced/navigation-guards.html#in-component-guards):

        beforeRouteEnter(routeTo, routeFrom, next)
        beforeRouteUpdate(routeTo, routeFrom, next)
        beforeRouteLeave(routeTo, routeFrom, next)
    

We can define each of these inside our components, just like lifecycle hooks. First, let’s learn about the parameters:

*   **routeTo** - This refers to the route that is about to be navigated to.
*   **routeFrom -** This refers to the route that is about to be navigated away from.
*   **next -** This is a function that can be called in each of them to resolve the hook, and continue navigation.

Now let’s take a closer look at when each of these is called when it’s defined inside a component.

**beforeRouteEnter(routeTo, routeFrom, next)**
----------------------------------------------

This is called before the component is created. Since the component has not been created yet, we can’t use the `this` keyword here. However, if we want to set some reactive data inside our component, there’s a way to set it using the `next` which we’ll show in our example below.

**beforeRouteUpdate(routeTo, routeFrom, next)**
-----------------------------------------------

This is called when the route changes, but is still using the same component. An example here is when we paginate, and we switch from page to page but still using the same component. It does have access to “this”.

**beforeRouteLeave(routeTo, routeFrom, next)**
----------------------------------------------

This is called when this component is navigated away from. It does have access to “this”.

Looking into Next
-----------------

As I mentioned above, each of these methods at some point can call `next()` (and previously it was required). Here’s what you can do with next:

*   `next()` - Called by itself will continue navigation to the component, which is referenced inside  `routeTo`.
*   `next(false)` - Cancels the navigation.
*   `next('/')`Redirects page to the / path.
*   `next({ name: 'event-list' })` - Redirects to this named path

Any options you might put in a `router-link`’s `to` property you can send into `next()` for redirecting navigation.

* * *

**beforeRouteLeave Example**
----------------------------

If I want to confirm that the user wants to leave the page before saving changes, I might use the following code inside my component:

      data: function() {
        return {
          unsavedChanges: false  // <-- Flag gets set to true if anything 
                                 //          is changed on the form
        }
      },
      beforeRouteLeave(routeTo, routeFrom, next) {
        if (this.unsavedChanges) {
          const answer = window.confirm(
            'Do you really want to leave? You have unsaved changes!'
          )
          if (answer) {
            next() // <-- Confirms the navigation
          } else {
            next(false) // <-- Cancels the navigation
          }
        } else {
          next() // <-- Confirms the navigation
        }
      }
    

Here you can see it in action:

![https://firebasestorage.googleapis.com/v0/b/vue-mastery.appspot.com/o/flamelink%2Fmedia%2F2.1615823301386.gif?alt=media&token=cfffe9b3-bf51-42f1-b7ba-d5584ae1318e](https://firebasestorage.googleapis.com/v0/b/vue-mastery.appspot.com/o/flamelink%2Fmedia%2F2.1615823301386.gif?alt=media&token=cfffe9b3-bf51-42f1-b7ba-d5584ae1318e)

New v4 Router Syntax

With the new version of Vue Router that ships with Vue 3, there’s another way we could write this which is a little shorter. Instead of using `next`, we can use return values. In this case:

*   No return value - Will continue navigation.
*   `return true` - Will continue navigation.
*   `return false` - Cancels the navigation.
*   `return '/'` Redirects page to the / path.
*   `return { name: 'event-list' }` - Redirects to this named path

Using this, our new code could be simplified to:

      data: function() {
        return {
          unsavedChanges: false  // <-- Flag gets set to true if anything 
                                 //          is changed on the form
        }
      },
      beforeRouteLeave(routeTo, routeFrom) {
        if (this.unsavedChanges) {
          const answer = window.confirm(
            'Do you really want to leave? You have unsaved changes!'
          )
          if (!answer) {
            return false // <-- Cancels the navigation
          }
        } 
      }
    

So, h**ow might we use these with our EventList component?**
------------------------------------------------------------

In our EventList.vue we are currently doing an API call which looks like this:

📄 **/src/views/EventList.vue**

    created() {
      watchEffect(() => {
        this.events = null
        EventService.getEvents(this.perPage, this.page)
          .then(response => {
            this.events = response.data
            this.totalEvents = response.headers['x-total-count']
          })
          .catch(error => {
            console.log(error)
          })
       })
    },
    

We can remove the `this.events=null` because we’re going to ensure our component doesn’t load until the API call is returned. Also, inside the `beforeRouteEnter` hook, `watchEffect` won’t work because it wouldn’t be associated with our component. So does this mean we can just move this code into `beforeRouteEnter` like so?

    beforeRouteEnter(routeTo, routeFrom, next) {
      EventService.getEvents(2, this.page)
          .then(response => {
            this.events = response.data
            this.totalEvents = response.headers['x-total-count']
          })
          .catch(() => {
            this.$router.push({ name: 'NetworkError' })
          })
    },
    

Nope! We’re using `this` all over the place, and we don’t have access to `this` inside `beforeRouteEnter`, so what do we do?

You might think that we could pull the `page` number out of the props for the component, but the component isn’t loaded yet, and this line from our router doesn’t get run until the component is navigated to:

    props: route => ({ page: parseInt(route.query.page) || 1 }) 
    

So we’ll have to make due parsing it out as needed.

Then we have the issue of not having access to set the `events` and `totalEvents`. To address this we’ll use `next()` and send in code to run inside the component, like so:

    return EventService.getEvents(2, parseInt(routeTo.query.page) || 1)
          .then(response => {
            next(comp => {
              comp.events = response.data
              comp.totalEvents = response.headers['x-total-count']
            })
          })
    

Sending in a function to `next` will cause that function to be run inside the component once it’s loaded, essentially setting our reactive properties. I’m using `comp` here to represent the Component, but we can use anything and inside the Vue Router documentation you’ll often find that they use `vm` (it stands for Vue Model, which is a Vue internals thing).

Finally if there’s an error on this API call, let’s send the user to the network error page.

          .catch(() => {
            next({ name: 'NetworkError' })
          })
    

Here’s the entire code:

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
    

If you’re coding along, you’ll want to make sure this works first before continuing.

2\. Installing the `nprogress` Progress Bar library
---------------------------------------------------

We’ll need to install **NProgress,** our progress bar library by running:

        $ npm install nprogress
    

and then add the CSS for NProgress:

📃 **/src/main.js**

        import 'nprogress/nprogress.css'
    

3\. Start progress bar when routing to the component
----------------------------------------------------

4\. When API returns finish progress bar
----------------------------------------

Finally, we can add our progress bar and we’ll do these two steps all at once. We’ll want to import our NProgress library, start the progress bar before our API call, and finish the progress bar inside the `finally` callback. The `finally` callback on our promise (API call) is called whether our API call is successful or it errors out (this is a JavaScript feature, not Vue). Here’s what our code looks like:

    ...
    import NProgress from 'nprogress' // <---
    
    export default {
      name: 'EventList',
      props: ['page'],
      components: {
        EventCard
      },
      data() {
        return {
          events: null,
          totalEvents: 0
        }
      },
      beforeRouteEnter(routeTo, routeFrom, next) {
        NProgress.start()
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
          .finally(() => {
            NProgress.done()
          })
      },
      ...
    

Now our application is working with the Progress bar.

![https://firebasestorage.googleapis.com/v0/b/vue-mastery.appspot.com/o/flamelink%2Fmedia%2F3.1615823306744.gif?alt=media&token=f8f1ea7e-cfb3-470b-9da0-a39b51ed2ceb](https://firebasestorage.googleapis.com/v0/b/vue-mastery.appspot.com/o/flamelink%2Fmedia%2F3.1615823306744.gif?alt=media&token=f8f1ea7e-cfb3-470b-9da0-a39b51ed2ceb)

🛑 Problem: Pagination stopped working
--------------------------------------

If we try to change pages we’ll notice the same problem we ran into earlier in this course, where nothing happens. Like so:

![https://firebasestorage.googleapis.com/v0/b/vue-mastery.appspot.com/o/flamelink%2Fmedia%2F4.1615823309670.gif?alt=media&token=d2cdda5f-5682-4ffb-8ed9-95c14ef58d53](https://firebasestorage.googleapis.com/v0/b/vue-mastery.appspot.com/o/flamelink%2Fmedia%2F4.1615823309670.gif?alt=media&token=d2cdda5f-5682-4ffb-8ed9-95c14ef58d53)

✅ Solution: beforeRouteUpdate
-----------------------------

This is because `beforeRouteEnter` just like `created` doesn’t get run again if we navigate to the same component. Previously we fixed this with `watchEffect` which we can’t use anymore. To fix this here, we need to use another navigation hook `beforeRouteUpdate`. Like so:

    beforeRouteUpdate(routeTo, routeFrom, next) {
      NProgress.start()
      EventService.getEvents(2, parseInt(routeTo.query.page) || 1)
        .then(response => {
          this.events = response.data // <-----
          this.totalEvents = response.headers['x-total-count'] // <-----
          next() // <-----
        })
        .catch(() => {
          next({ name: 'NetworkError' })
        })
        .finally(() => {
          NProgress.done()
        })
    },
    

Notice this code is close to identical to the code we wrote above, with one important difference. BeforeRouteUpdate does have access to `this`, so we can set our event data using `this`, and simply call `next()`.

Simplifying Using Return
------------------------

Now that we’re not using the special functionality of `next()` with a function (remember the `comp` thing) we can choose to use the other `return` syntax of Vue Router v4. Like so:

    beforeRouteUpdate(routeTo) {
      NProgress.start()
      return EventService.getEvents(2, parseInt(routeTo.query.page) || 1)
        .then(response => {
          this.events = response.data // <---
          this.totalEvents = response.headers['x-total-count'] // <---
        })
        .catch(() => {
          return { name: 'NetworkError' } // <---
        })
        .finally(() => {
          NProgress.done()
        })
    },
    

Now our pagination is working again!

![https://firebasestorage.googleapis.com/v0/b/vue-mastery.appspot.com/o/flamelink%2Fmedia%2F5.1615823311963.gif?alt=media&token=8274d49f-d059-47c7-87f6-55b6a7775d2c](https://firebasestorage.googleapis.com/v0/b/vue-mastery.appspot.com/o/flamelink%2Fmedia%2F5.1615823311963.gif?alt=media&token=8274d49f-d059-47c7-87f6-55b6a7775d2c)

Next we’ll dive into global and per-route guards, which provide a useful mechanism for extracting logic into the router.

### Lesson Resources

##### Source Code:

*   [Starting Code](https://github.com/Code-Pop/Touring-Vue-Router/tree/L9-start)
    
    Node v16 required
    
*   [Ending Code](https://github.com/Code-Pop/Touring-Vue-Router/tree/L9-end)
