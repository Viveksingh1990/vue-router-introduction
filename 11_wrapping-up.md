Wrapping Up
===========

We’ve covered some of the primary features of Vue Router in this course, and great job getting this far. In this lesson we’ll take a quick look at some advanced features and why you might use them, such as route meta fields, lazy loading routes, and scroll behavior.

* * *

Route Meta Fields
-----------------

There are times when you want to associate information with a series of routes, and define them at the router level. The best example of this is authorization, when you only want certain people to be able to follow certain routes. It could be that only logged-in users can view certain routes, and it could be that only users can edit their own created content (not other users’ content). In this case, it’s useful to wrap-in authorization at the router level, rather than at the component level.

Let’s play with this in our existing router from the course. FYI, I’m just playing around so you won’t find this code in github, but you are welcome to copy and paste.

📃 **/src/router/index.js**

    ...
    {
      path: 'edit',
      name: 'EventEdit',
      component: EventEdit,
      meta: { requireAuth: true }
    }
    

As you can see, the meta option simply takes a hash, and you can place any value in that hash. This information can then be used any place where we have access to a route object (usually in the `to` or `from` arguments).

Here’s an example of how we might use this information inside the router file itself:

    router.beforeEach((to, from) => {
      NProgress.start()
    
      const notAuthorized = true
      if (to.meta.requireAuth && notAuthorized) {
        GStore.flashMessage = 'Sorry, you are not authorized to view this page'
    
        setTimeout(() => {
          GStore.flashMessage = ''
        }, 3000)
    
        return false
      }
    })
    

I’m using `notAuthorized` as a constant for now, but this could easily be calling another method from a library that has all the authorization code. I’m using the `flashMessage` that we first created in the blah blah lesson, and finally `return false` is telling Vue Router to simply stop the navigation. Here’s what it looks like:

![https://firebasestorage.googleapis.com/v0/b/vue-mastery.appspot.com/o/flamelink%2Fmedia%2F1.1622835295500.gif?alt=media&token=98f5d6cc-655b-4831-970e-cbfa0eb3fe5f](https://firebasestorage.googleapis.com/v0/b/vue-mastery.appspot.com/o/flamelink%2Fmedia%2F1.1622835295500.gif?alt=media&token=98f5d6cc-655b-4831-970e-cbfa0eb3fe5f)

There is one issue with this solution, though. When I navigate directly to the URL to edit the page, I get a blank page that looks like this:

![https://firebasestorage.googleapis.com/v0/b/vue-mastery.appspot.com/o/flamelink%2Fmedia%2F2.1622835295501.gif?alt=media&token=8e4f52c9-96f8-4027-9251-069e009001ca](https://firebasestorage.googleapis.com/v0/b/vue-mastery.appspot.com/o/flamelink%2Fmedia%2F2.1622835295501.gif?alt=media&token=8e4f52c9-96f8-4027-9251-069e009001ca)

To get around this, we can check if the `from` route has an `href` associated with it. If it doesn’t, then this must be someone going directly to this URL, in which case let’s redirect them to the root path: the list of events:

    router.beforeEach((to, from) => {
      NProgress.start()
    
      const notAuthorized = true
      if (to.meta.requireAuth && notAuthorized) {
        GStore.flashMessage = 'Sorry, you are not authorized to view this page'
    
        setTimeout(() => {
          GStore.flashMessage = ''
        }, 3000)
    
        if (from.href) { // <--- If this navigation came from a previous page.
          return false
        } else {  // <--- Must be navigating directly
          return { path: '/' }  // <--- Push navigation to the root route.
        }
      }
    })
    

Now we can see that navigating to the page directly properly gives the error message and redirects to the front page.

![https://firebasestorage.googleapis.com/v0/b/vue-mastery.appspot.com/o/flamelink%2Fmedia%2F3.1622835300541.gif?alt=media&token=e3b8ceb4-a065-476c-87a8-8aaeea7e08cf](https://firebasestorage.googleapis.com/v0/b/vue-mastery.appspot.com/o/flamelink%2Fmedia%2F3.1622835300541.gif?alt=media&token=e3b8ceb4-a065-476c-87a8-8aaeea7e08cf)

It’s worth mentioning that when using children routes, like we do with our `EventLayout` in our example app, we can place `meta` options on our parent route, and it will automatically get inherited by children routes. This could be useful when we have admin pages that all require authorization.

* * *

Lazy Loading Routes
-------------------

When building apps that contain large amounts of JavaScript, the initial page load (where that JavaScript is downloaded) might start to get large and slow down how fast the application loads. There might also be parts of your application that are only loaded by certain people, such as content creators vs content consumers. There are a lot more people on YouTube who just watch videos vs create videos. The people who just _watch_ videos don’t need to download all the JavaScript needed to load the web pages that video _creators_ use.

**Lazy Loading allows us to separate our application into separate JavaScript bundles, so that a bundle is only downloaded when needed.**

You might have already seen the basic version of lazy loading when you created your first Vue application. The Vue Router by default contained this:

📃 **/src/router/index.js**

    {
        path: '/about',
        name: 'About',
        // route level code-splitting
        // this generates a separate chunk (about.[hash].js) for this route
        // which is lazy-loaded when the route is visited.
        component: () => import(/* webpackChunkName: "about" */ '../views/About.vue')
      }
    

If we load up the basic webpage, and look in Chrome Dev Tools Network tab, we can see that it’s not until we click the “about” link, that the about page JavaScript is loaded.

![https://firebasestorage.googleapis.com/v0/b/vue-mastery.appspot.com/o/flamelink%2Fmedia%2F4.1622835303324.gif?alt=media&token=845b4b3a-db1e-4be6-9ef6-cb92bac93ee9](https://firebasestorage.googleapis.com/v0/b/vue-mastery.appspot.com/o/flamelink%2Fmedia%2F4.1622835303324.gif?alt=media&token=845b4b3a-db1e-4be6-9ef6-cb92bac93ee9)

Notice the JavaScript it loaded was called “about.js”; that’s the webpageChunkName that we specified in the comment.

Another way we might often see the above code written in Vue is as such:

    const About = () => import(/* webpackChunkName: "about" */ '../views/About.vue')
    
    const routes = [
      ...
      {
        path: '/about',
        name: 'About',
        component: About
      }
    

As you might imagine, if you had a bunch of YouTube creator pages, you’d use the same webpackChunkName, like so:

    const Uploader = () => import(/* webpackChunkName: "creator" */ '../views/Uploader.vue')
    const Editor = () => import(/* webpackChunkName: "creator" */ '../views/Editor.vue')
    const Publisher = () => import(/* webpackChunkName: "creator" */ '../views/Publisher.vue')
    

Now when any of those routes are navigated to, the `creator.js` JavaScript bundle is loaded onto the page.

* * *

Scroll Behavior
---------------

Notice how when we are on Google and search for Vue Mastery, when we scroll down and click on the next button, we show up at the top of the second page of results:

![https://firebasestorage.googleapis.com/v0/b/vue-mastery.appspot.com/o/flamelink%2Fmedia%2F5.1622835306037.gif?alt=media&token=cb0ccc0f-95ce-4801-ad07-78545e405e47](https://firebasestorage.googleapis.com/v0/b/vue-mastery.appspot.com/o/flamelink%2Fmedia%2F5.1622835306037.gif?alt=media&token=cb0ccc0f-95ce-4801-ad07-78545e405e47)

One unintended side effect of having a single page application is that when you scroll down a page, and click a button, you’re not (by default) scrolled up to the top of the page. See our events pagination:

![https://firebasestorage.googleapis.com/v0/b/vue-mastery.appspot.com/o/flamelink%2Fmedia%2F6.1622835310806.gif?alt=media&token=c59a5eb7-5a85-4e17-8d64-27e4542fec64](https://firebasestorage.googleapis.com/v0/b/vue-mastery.appspot.com/o/flamelink%2Fmedia%2F6.1622835310806.gif?alt=media&token=c59a5eb7-5a85-4e17-8d64-27e4542fec64)

Luckily we can easily give our application this functionality, by adding a little code to our router:

📃 **/src/router/index.js**

    ...
    const router = createRouter({
      history: createWebHistory(process.env.BASE_URL),
      routes,
      scrollBehavior() {  // <---
        // always scroll to top
        return { top: 0 }
      }
    })
    ...
    

Now when we navigate, we will be brought back up to the top of the page we navigated to:

![https://firebasestorage.googleapis.com/v0/b/vue-mastery.appspot.com/o/flamelink%2Fmedia%2F7.1622835314307.gif?alt=media&token=bdb01b2b-ece5-4337-938b-4174b50cc9d1](https://firebasestorage.googleapis.com/v0/b/vue-mastery.appspot.com/o/flamelink%2Fmedia%2F7.1622835314307.gif?alt=media&token=bdb01b2b-ece5-4337-938b-4174b50cc9d1)

Great, but there’s another behavior that we might want. On Google, when we scroll to the bottom of a page, click to go to the next page, and then use the back button, we’re brought back to where we just were (scrolled to the bottom of the page).

![https://firebasestorage.googleapis.com/v0/b/vue-mastery.appspot.com/o/flamelink%2Fmedia%2F8.gif?alt=media&token=6ba35db8-214c-436e-967f-a9673c5c8c7a](https://firebasestorage.googleapis.com/v0/b/vue-mastery.appspot.com/o/flamelink%2Fmedia%2F8.gif?alt=media&token=6ba35db8-214c-436e-967f-a9673c5c8c7a)

This is a behavior we don’t really think about, but we come to expect. When we hit the back button, we expect to be brought back to where we just left. However, we just told our Vue application to go to the top of the page on every navigation (even back).

To go back to the same part of the page we just left, it’s a small modification:

    ...
    const router = createRouter({
      history: createWebHistory(process.env.BASE_URL),
      routes,
      scrollBehavior(to, from, savedPosition) {
        if (savedPosition) { // <----
          return savedPosition
        } else {
          return { top: 0 }
        }
      }
    })
    ...
    

If there is a saved position for this page, now it will properly go back to where we just were. We can see this inside our event example:

![https://firebasestorage.googleapis.com/v0/b/vue-mastery.appspot.com/o/flamelink%2Fmedia%2F9.gif?alt=media&token=577f6e26-3f86-442b-9697-be68bd8cea0f](https://firebasestorage.googleapis.com/v0/b/vue-mastery.appspot.com/o/flamelink%2Fmedia%2F9.gif?alt=media&token=577f6e26-3f86-442b-9697-be68bd8cea0f)

* * *

Conclusion
----------

As you can see, Vue Router provides some very powerful tools for creating our Vue app. There are a few more topics that you can read about in the [Vue Router documentation](https://next.router.vuejs.org/), but I really hope this course gave you a strong base to continue building Vue applications.
