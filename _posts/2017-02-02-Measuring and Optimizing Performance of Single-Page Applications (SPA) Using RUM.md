---  
layout: post  
title:  "Measuring and Optimizing Performance of Single-Page Applications (SPA) Using RUM"  
date:   2017-02-02 09:16:58  
categories: performance  
author: liyan  
---  

### Introduction  

Improving site speed is one of the major technology initiatives at LinkedIn because it is highly correlated with the engagement members have on our website. Real User Monitoring (RUM) is an approach where we use data from real users, instead of a synthetic lab environment, to measure performance, and this is the primary way we measure site speed at LinkedIn. In this blog post, we present how we measure performance of single-page applications at LinkedIn using RUM, and how we used RUM data to make our new [LinkedIn](https://blog.linkedin.com/2017/january/19/linkedin-desktop-redesign-puts-conversations-and-content-at-the-center) web application faster by 20%.  

Page Load Time (PLT) is the key metric that we measure for every page at LinkedIn in order to capture the user’s perception of when the page is ready. It is not easy to measure this metric uniformly across all pages because it is highly subjective depending on the content of the page and on the end user’s perception. [Speed Index](https://sites.google.com/a/webpagetest.org/docs/using-webpagetest/metrics/speed-index) is a good indicator for the user’s perception of when the page is rendered, and we use it for measuring performance in synthetic environments. But it is not possible to calculate this metric [accurately](https://github.com/WPO-Foundation/RUM-SpeedIndex) using RUM. So, we needed to find a good proxy for PLT that could be easily measured using RUM. For traditional web applications, which are mostly server-side rendered, window.onload() event is a reasonably good proxy for PLT. Most traditional RUM libraries use the [Navigation Timing API](https://www.w3.org/TR/navigation-timing/) to detect when window.onload() event is fired and thereby measure PLT for the page. However, as we discuss below, we ran into several obstacles when trying to use window.onload() as a proxy for PLT for single-page applications.  
  
  
  
### Single-page applications  

Single-page applications (SPA) are web applications built using a Javascript MVC framework, like [Ember](https://engineering.linkedin.com/blog/2016/12/pemberly-at-linkedin), to deliver a rich, app-like experience. The HTML for these pages is mostly built on the client browser instead of the web server. Another thing to note is that any page/URL in these applications can be visited in two modes:  

- **App launch**: This occurs when the app is initially loaded by entering the URL in the browser or by clicking on an email link. “App launch” mode is typically slow, as the application (JS/CSS) needs to be downloaded and booted before doing the work to render the page.  
- **Subsequent**: This occurs when the app has already been loaded and the page is visited by clicking on a link within the app. “Subsequent” mode is typically fast because the application is already downloaded and booted, and we just need to fetch new data for the page and render it.  

As LinkedIn web applications moved from traditional server-side rendered web pages to modern single-page applications, we faced many challenges in using the window.onload() event for measuring performance of these applications using RUM.  

When a page is visited in “app launch” mode, the window.onload() event is fired too early relative to when the user sees meaningful content on the page, as illustrated in the image below. window.onload() event is geared more towards resource download times instead of rendering times, and when most of the rendering happens on the client side, it does not represent PLT accurately.  

![](/image/170202_SRA1.jpg)  

When a page is visited in “subsequent” mode, the window.onload() event is not fired at all because there is no new HTML document being downloaded. We cannot track the performance of these page views if we rely on the window.onload() event.  

![](/image/170202_SRA2.jpg)  
  
  
  
### RUM for single-page applications  

We need to automatically detect the start of a “subsequent” visit and detect when the page has been rendered to measure the PLT for both “app launch” and “subsequent” navigations. When considering our options, we looked at the different ways to do so:  

- Using [Resource Timing API](https://www.w3.org/TR/resource-timing/) to detect when an AJAX call has been made to identify the start of a “subsequent” visit.  
- Using [MutationObserver](https://developer.mozilla.org/en-US/docs/Web/API/MutationObserver) to detect changes in DOM and detecting end of network activity using Resource Timing API.  

These approaches were either unreliable or inaccurate for our use case. For example, we make AJAX requests to pre-fetch some data without the user initiating a “subsequent” visit. Also, there might be network activity happening that does not impact user experience. We want PLT to closely match when the user has seen content on the screen, instead of network activity.  

We figured out that the most reliable way we could detect these events is by letting the application tell us when they’ve happened by using a simple API. Below is an example of how the application would use the API to mark these events.  

```javascript  
var rumObj = new RumTracking({  
  'web-ui-framework': 'EMBER'  
});  

// App Launch - window.performance.timing.navigationStart is the start marker  
rumObj.setPageKey('feed_page_key');  
// Do rendering  
rumObj.appRenderComplete();  
  
// Subsequent navigation  
rumObj.appTransitionStart();  
rumObj.setPageKey('profile_page_key');  
// Do rendering  
rumObj.appRenderComplete();  
```  

The main problem with this approach is that each page would have to write the instrumentation code, which is very tedious. Thankfully, some SPA frameworks, like Ember, have rich [lifecycle hooks](http://emberjs.com/api/classes/Ember.Router.html) that we can use to add this instrumentation automatically.  

Below is an example of how we automate the instrumentation for most cases in our Ember addon.  

```javascript  
// Add instrumentation for subsequent navigation start  
router.on('willTransition', () => {  
    rumObj.appTransitionStart();  
});  
  
// Add instrumentation for rendering is complete  
router.on('didTransition', () => {  
    Ember.run.scheduleOnce('afterRender', () => {  
        rumObj.appRenderComplete();  
   });  
});  
```  

We know the start of “app launch” navigation based on [navigationStart](https://www.w3.org/TR/navigation-timing/#dom-performancetiming-navigationstart) from Navigation Timing API. We listen to the router’s willTransition event to detect the start of a “subsequent” navigation. We also listen to the router’s didTransition event and add work in the afterRender queue to notify that the page has been rendered for both “app launch” and “subsequent” navigations. We now have the start and end times of page renderings for both “app launch” and “subsequent” navigations and can therefore calculate PLT based off of these times.  

With this approach, we have automated the measurement of PLT for many single-page applications within LinkedIn that are built on top of [Ember](http://emberjs.com/), [AngularJS](https://angularjs.org/), and [Marionette](http://marionettejs.com/) frameworks.  
  
  
  
### Granular metrics for finding optimization opportunities  

If we want to debug any performance issues, we need to collect more granular metrics to breakdown the PLT and understand where the [bottlenecks](https://engineering.linkedin.com/blog/2017/01/boss--automatically-identifying-performance-bottlenecks-through-) are. Traditional RUM libraries would rely on [Navigation Timing API](https://www.w3.org/TR/navigation-timing/) and [Resource Timing API](https://www.w3.org/TR/resource-timing/) to provide these granular metrics about the HTML and resource download times. But single-page applications spend a significant amount of time on the browser doing JavaScript execution after the main HTML/JS/CSS are downloaded. In order to identify the key milestones or phases during this JavaScript execution, we added granular metrics using the [User Timing API](https://www.w3.org/TR/user-timing/). Once we added these granular metrics, we were able to construct the waterfall below, which helps us visualize bottlenecks easily.  

![](/image/170202_SRA3.jpg)  
*Waterfall for a typical “app launch” page in a single-page application*  

When we launched our new LinkedIn web application built on Ember, we [analyzed](https://engineering.linkedin.com/performance/monitor-and-improve-web-performance-using-rum-data-visualization) the RUM data to find optimization opportunities. Below are a couple of optimizations that we implemented based on RUM data.  
  
  
  
### Occlusion culling (lazy rendering)  

When we looked at the granular metrics charts, we found that about 30% of the page load time was being spent in the “render” phase. The render phase is where we build the DOM for the components on the page after the data is available. We also noticed that the browser’s main thread is not yielded for a paint until the DOM is created for all the components on the page.  

![](/image/170202_SRA4.jpg)  

If we yield the browser main thread earlier, after the DOM is created for only components that are in the viewport, we can have a much faster user experience, as we do not need to render components below the viewport before the browser does a paint. We refer to this optimization technique, where we avoid/defer rendering of content outside the viewport, as “occlusion culling.”  

At a high level, the major performance issue here is that all the components on the page have the same priority and all of them are rendered/painted at the same time. So we developed a solution where we can give different priorities to components based on when they need to be rendered. This is illustrated in the image below:  

![](/image/170202_SRA5.jpg)  

The first category is the blue-colored components, which are marked as “non occludable.” They have the highest priority and are always rendered on the page irrespective of the viewport height. We keep all the components that fit in the typical viewport in this category.  

The second category is the green-colored components, which are marked as “occludable” but have lesser priority and might be rendered incrementally in the next animation frame, depending on these two conditions:  

- If the viewport height is longer than a typical viewport, these components will be rendered;  
- If the component is configured to pre-render for smooth scrolling experience, even if it is outside the viewport.  

The third category is the red-colored components, which are marked as “occludable” and are not rendered (culled) initially because they are outside the viewport. They will be rendered as we scroll and they enter into the viewport.  

Using this approach, we incrementally render/paint components and can have a faster [First Meaningful Paint](https://developers.google.com/web/tools/lighthouse/audits/first-meaningful-paint) of the content in the viewport. Additionally, since we avoid doing the work to render some components that are outside the viewport, we can also have a faster [Time To Interactive](https://developers.google.com/web/tools/lighthouse/audits/time-to-interactive). When we implemented this optimization for some pages, we noticed that the time spent in render phase improved by 50% at both the 50th and 90th percentile.  

![](/image/170202_SRA6.jpg)  
  
  
  
### Lazy data fetching  

After we improved the render phase, we noticed that 20% of the page load time was being spent in the “Transition” phase, according to the granular metrics charts. The transition phase is where we wait for the data to arrive, normalize the data, and then push it to Ember data store. This work is heavily dependent on the amount of data that needs to be processed. If we lazily fetch data that is not needed for First Meaningful Paint, we can reduce the amount of data that needs to be processed and have a faster First Meaningful Paint. So we split the data fetching process into two calls at a high level: one to fetch data needed for First Meaningful Paint and the other to fetch the remaining data needed for the page. We applied this optimization for some pages and noticed that the transition phase improved by up to 40%.  

Another advantage of the optimizations described above is that they apply to both “app launch” and “subsequent” navigations because these optimizations are in the time spent on the client browser after the application has been downloaded and booted.  
  
  
  
### Lessons learned  

These are just some of the optimizations which were implemented and validated using RUM data as we ramped our new LinkedIn web application. Some best practices and lessons learned as we went through this journey are:  

- Defer work not needed for First Meaningful Paint. This principle yielded us good results in many cases, including the optimizations described above. There were many other optimizations, like lazily loading application code (JS/CSS) not needed for First Meaningful Paint, that also fall under this category.  

- Analyze RUM data thoroughly in addition to performance data obtained from synthetic and development environments. There are a wide variety of devices, networks, and users in the real world, and it is not easy to simulate all of them in a synthetic environment. In many cases, the optimizations in the real world were significantly higher than what we noticed in a synthetic environment.  

- Always do an A/B experiment to accurately measure the gains from a given optimization. Since we are running many optimization experiments in parallel, we cannot attribute the gains accurately to a specific experiment without putting the optimization behind an A/B experiment. This also helps us in informing how much gain the same experiment would give in other pages and applications. It is also a good idea to continue to do these experiments periodically to see how they are performing, as bottlenecks shift over time.  

- To iterate quickly on new ideas, we should validate and refine them in a synthetic performance testing environment. We have a synthetic performance testing framework where we can do many runs of a test in an isolated environment to validate and refine the idea before pushing out to production. This framework has been very useful for iterating quickly on new ideas.  


（ 原文 [Measuring and Optimizing Performance of Single-Page Applications (SPA) Using RUM](https://engineering.linkedin.com/blog/2017/02/measuring-and-optimizing-performance-of-single-page-applications)）  
