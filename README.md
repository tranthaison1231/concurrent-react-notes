## ⚛️ Concurrent-React-Notes

Welcome to `concurrent-react-notes` - a great place to learn about Concurrent React!

> If you are looking for notes from before the launch of Concurrent React at ReactConf 2019, see the [/legacy](/legacy/README.md) folder.

Everything here is information since launch, assuming you have seen the official introductory talks from the React team and [read the official docs](https://reactjs.org/docs/concurrent-mode-intro.html).

These are personal notes, so they _will_ have an editorial bias. But you are welcome to open issues, contribute, and discuss with me.

## Concurrent

- React Podcast: Andrew Clark on Concurrent Mode ([podcast](https://reactpodcast.com/70))

Suspense is declarative loading states. Concurrent Mode is a way to coordinate them in a more intentional way.

(source: [Dan](https://mobile.twitter.com/dan_abramov/status/1189276304232865793))

The point of Concurrent Mode is it's easy to choose what's required and what's deferred. You wrap the deferred stuff into its own <Suspense> and you're good to go. So whenever you add a new widget, you can decide whether it should delay page transition, or if should load after.

(source: [Dan](https://mobile.twitter.com/dan_abramov/status/1189275628677357569))

"Concurrent Mode" is ok but I prefer calling it "React on Acid" - [Dan](https://mobile.twitter.com/dan_abramov/status/1189235810337525767)

In fact, Concurrent Mode is aimed to enable _better_ offscreen culling because it can optimistically “warm up” and pre-render next likely items without blocking and jank.

(source: [Dan](https://mobile.twitter.com/dan_abramov/status/1187920574926016513))

another thing enabled by Concurrent Mode is partial and progressive hydration. Which directly benefits low end.

(source: [Dan](https://mobile.twitter.com/dan_abramov/status/1169364691044491264))

You may see unexpected issues in Concurrent Mode because a render can be aborted.

(source: [Dan](https://mobile.twitter.com/dan_abramov/status/1161973331979132929))

Is Concurrent Mode just a workaround for “virtual DOM diffing” overhead? Some people got that impression. Let me clarify why we’re working on it:

1. Time slicing keeps React responsive while it runs your code. Your code isn’t just DOM updates or “diffing”. It’s any JS logic you do in your components! Sometimes you gotta calculate things. No framework can magically speed up arbitrary code.
2. Making updates really fast is a great goal. However, how many of the interactions in apps you use are “very fast updates to existing nodes”, as opposed to “replacing a part of the screen with new content”? Go ahead and count them.
3. When you replace a part of screen with new content (like when you click on my tweet or scroll it down), there’s fewer shortcuts a library can do. You gotta create those DOM nodes, possibly transform the data, and run some calculations. This is CPU work.
4. You can optimize it somewhat. But this work has to be done. What’s interesting though — is _when_ you do it. Traditional model is “fetch data, then mount”. This means you’re stuck wasting CPU cycles not doing anything useful while waiting for data and more code to arrive.
5. No amount of “reactivity” solves that. It’s not a problem of handling new inputs — it’s a scheduling problem. Concurrent React starts rendering “in memory” immediately, even while code and data for some components is still loading.
6. The goal is to be responsive regardless of whether CPU or IO is lagging behind. So you want to _interleave_ CPU and IO work. Let components render “in memory” while data for others is still streaming in, and show the final result when it’s ready. Not “fetch and mount”.
7. Showing updates as fast as possible seems like an obvious goal. But is it, always? I don’t think it is when you fetch (IO). User perception research shows that a fast succession of loading states (flashing and hiding spinners) makes the transition feel _slower_.
8. So you wanna remove “virtual”. But if a UI library can’t start rendering code “in memory” and its every “render” has to produce an immediate visible UI update, it loses the ability to coordinate screen updates and optimize them for human perception.
9. You can’t be faster than “done”. Rendering “in memory” before all the data is ready is faster by definition than waiting for the whole thing. You can try to fix it by rendering to screen early — but showing loading states too fast feels janky and you get too many reflows.
10. CPU and IO are two sides of the same coin. You have to solve both. Removing “in memory” virtual representation means that for one of most common transitions (replacing part of a screen) you have to choose between janky loading sequence or starting work too late. Both suck.
11. What if there was a layer that, due to “virtual” component output, can start rendering as soon as you click (rather than when you finish fetching), continue in background as more code arrives, and coordinate screen paints for minimal jank and flicker? That’s Concurrent React.
12. When we started working on Concurrent React, we had no idea about the IO side of this question or coordinating loading states. But if you think about how to bring best experience to the user regardless of their network and device, you’re gonna have to think about IO a lot.
13. Concurrent React is still in development. It was a multi-year project. We are actively dogfooding it now, and there’s still work to polish the APIs and ensure common UI patterns are covered well. We want to make sure it’s super solid before it’s marked as stable.
14. I can’t resist some demo time. We’re currently working on new React DevTools. One of the ways it improves performance is by only serializing props for selected element. But do you see the downside of adding asynchronous data? Note the flashing “Loading...” in the right pane.
15. We can’t fetch that data “faster”. It’s asynchronous by nature. But what if we can let Concurrent React coordinate the screen updates for minimal jank? It looks like this. Right pane updates are slightly delayed but you can hardly perceive that. So smooth!
16. **Our goal with Concurrent React is to make this experience the default.** You don’t need to coordinate loading sequences for minimal jank — React does that. Computations don’t need to stall the thread either. And we can start work as early as possible thanks to being “virtual”.

(source: [Dan](https://mobile.twitter.com/dan_abramov/status/1120971795425832961))

Concurrent mode lets us yield to network responses and process them earlier than if we blocked. If that processing needs to send another request, it gets sent out earlier for better parallelism.

(source: [Seb](https://mobile.twitter.com/sebmarkbage/status/1189009831417405440))

## Suspense

Here are the concerns solved by Suspense:

- Single declarative way to specify loading states decoupled from _what_ is loading (GraphQL, REST, JS bundle, images) and where in the tree
- Graceful orchestration of those loading states (control over reveal order, avoiding flicker)
- Suspense also offers some new capabilities that data sources can take advantage of. For example, a response can gradually “unlock” deeper levels of data as it streams. That’s not new... but with declarative loading states, it means the app can also “unlock” UI in coordinated way.
- I like to think of Suspense as a way to find balance between technological and UX extremes. Technologically, streaming data and rendering immediately as it comes is fastest. UX-wise, it would be terrible to see every component load separately and shift layout every few ms.
- Suspense lets us choose well-defined boundaries where we’re willing to show loading states. That lets us stream data as it comes (and start rendering immediately) but only show result to the user in places we agreed to, in order we agreed to, and with a frequency that feels good.
- And let’s not forget it’s not just about data. Suspense doesn’t care what we’re waiting for. It uses the same mechanism for code, data, and any other async things you need. So you can stream code and data in parallel, and the app can “unlock” deeper loading states as we fetch.

(source: [Dan](https://twitter.com/dan_abramov/status/1190275834961178624))

Suspense decouples three things:

1. Visual presentation of loading states
2. Where data is being read
3. How data is requested and streamed in

“You can show something before the whole response comes in?” Yep 👍

“You can force different sections of UI to reveal in a top-down order even if their code or data loads in a different sequence” Yep 👍

“You can kick off a fetch early but wait before transition?” Yep 👍

This is counter-intuitive because we’re used to those things being coupled together. We’re used to manually orchestrating them. To me this is as big a mental shift as React was from manually managing the DOM.

(source: [Dan](https://mobile.twitter.com/dan_abramov/status/1189884884543774721))

The whole point of Suspense is you can pass the data down and read() it where needed. So optional data doesn't block the required parts from rendering.

(source: [Dan](https://mobile.twitter.com/dan_abramov/status/1189229077703614464))

Suspense doesn't handle:

- Retry patterns
- Throttling
- Error handling
- Sequential fetching
- Circuit breaker patterns
- Interruption/cancellation
- Deterministically testing it all!

(source: [David](https://mobile.twitter.com/DavidKPiano/status/1190267697327677445))

## useTransition

useTransition() lets us skip/delay the "recessed" state. That's when we had to hide some _existing_ content and show a spinner instead. Delaying that is usually better.

However, we want to get to the "skeleton" state as soon as possible. We don't want to wait for _everything_.

So useTransition() lets us wait for all the boundaries _outside_ to be ready. But once they're ready, we show the new page, and let the rest of the content in their own <Suspense> boundaries load incrementally. Potentially with <SuspenseList> to coordinate their reveal order.

When you setState(), some components stay on the page, and some components get unmounted or newly mounted, right?

"Recessed" boundaries already exist on the page. Hiding them is bad because you hide _existing_ content.

"Skeleton" are new boundaries. You haven't seen them yet.

For example, if you navigate from Feed to Profile, hiding the whole top-level app (including tabbar) would be bad. Because we already showed it before. It's bad to temporarily hide existing content.

But it's ok that on Profile page, "Photos" section might still be loading.

(source: [Dan](https://mobile.twitter.com/dan_abramov/status/1189561524718899200))

## Relay

- Joe Savona: Data Fetching With Suspense In Relay ([Youtube](https://www.youtube.com/watch?v=Tl0S7QkxFE4&list=PLPxbbTqCLbGHPxZpw4xj_Wwg8-fdNxJRh&index=15&t=0s))

## SSR, Progressive and Selective Hydration

"We’re actively investing into SSR but started from the client side (progressive hydration of Suspense boundaries). Unfortunately we can’t squeeze complex features into the existing SSR due to its architecture. We’re starting work on a new one."

(source: [Dan](https://mobile.twitter.com/dan_abramov/status/1189730578050011136))

## Batched Mode

“Batched” mode is like a limited version of concurrent mode that enables batching but none of the other features (time slicing, priorities, delayed Suspense, etc).

(source: [Andrew](https://mobile.twitter.com/acdlite/status/1175084727348412419))

## Meta

- Reflections on the lead up to Suspense https://mobile.twitter.com/dan_abramov/status/1184987041676845056
- Suspense and Hooks https://mobile.twitter.com/dan_abramov/status/1154501240019197952
- credits for people who worked on Concurrent https://twitter.com/dan_abramov/status/1187434146844553218
- the prerequisites for Concurrent https://mobile.twitter.com/acdlite/status/1187440140018282497
