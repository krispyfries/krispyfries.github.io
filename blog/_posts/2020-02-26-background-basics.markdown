---
layout: post
title:  "Short guide: Using XPCOM's background thread pool"
date:   2020-02-27 05:00:00 -0800
categories: blog
---

I decided to take my blurb about `BackgroundEventTarget` and make a guide about it. This post will walk you through the basics of how to use the background event target. This post assumes you're working on some part of gecko, you already understand the basics of working in async, and you want to convert or add new background work that you can send to the existing background threads.

## Why background thread pool?
The background event target was created to take on a lot of the async jobs that currently run as one-off threads or use the StreamTransport service for work unrelated to stream transport. The goal is to create a sort of "funnel" for runnables which improves the uniformity of async work and centralizes how we handle background work. Not to mention it's a lot easier to use the existing thread pool instead of maintaining your own thread. :)

## Determine what kind of dispatch you need
There are two ways to send jobs to the thread pool: `NS_DispatchBackgroundTask` and `NS_CreateBackgroundTaskQueue`. The kind of use you have and how you use it depends on what your specific needs are for the background pool. A good way to figure it out is like this:  
* If you don't care about order of dispatch or when it finishes and just want to throw background work onto something that can do it, you want `NS_DispatchBackgroundTask`.
* If you want to manipulate your own task queue to do things like make `IsOnCurrentThread` calls, dispatch serially, or do some other work that requires you to have an `nsISerialEventTarget` of your own, you want to use `NS_CreateBackgroundTaskQueue`. It returns a `TaskQueue` that you can manipulate which will dispatch to the background thread pool.

### The `NS_DISPATCH_EVENT_MAY_BLOCK` flag
Along with the new background pool we have a new dispatch flag which sends some extra information to the event target. Essentially, we want to split up potentially blocking I/O and regular background work. The general reasoning is simple: it's easier to control how many threads can block on I/O while still having background threads available for regular background work. So, you can send with your dispatch the `NS_DISPATCH_EVENT_MAY_BLOCK` flag.

## Example 1: `NS_DispatchBackgroundTask`
Say you have some `nsIRunnable*` called `aOffthreadTask` that you want to dispatch. It's going to do some non-blocking background for you. Sending it to the background event target is easy and would look something like this:

    NS_ENSURE_SUCCESS(NS_DispatchBackgroundTask(aOffthreadTask));

This sends the work to dispatch on a background thread.

## Example 2: `NS_CreateBackgroundTaskQueue`
Now, say we have some background IO in `aOffthreadIO` that must be followed by `aClosingIO`. Order matters, so we want to get a background task queue to put it all on:

    nsCOMPtr<nsISerialEventTarget> mBackgroundET;
    NS_CreateBackgroundTaskQueue("My Task Queue", getter_AddRefs(mBackgroundET));

To use the task queue, we just have to call `dispatch` on it with the right flags.

    mBackgroundET->Dispatch(aOffthreadIO, NS_DISPATCH_EVENT_MAY_BLOCK);
    mBackgroundET->Dispatch(aClosingIO, NS_DISPATCH_EVENT_MAY_BLOCK);

There's no need to shut down task queues - `nsThreadManager` handles shutdown for you. So that's all there is to it!

## Further reading
You can check out the source code for background dispatch: [Searchfox](https://searchfox.org/mozilla-central/rev/b2ccce862ef38d0d150fcac2b597f7f20091a0c7/xpcom/threads/nsThreadUtils.h#1732-1759) | [hg web](https://hg.mozilla.org/mozilla-central/file/tip/xpcom/threads/nsThreadUtils.h#l1733)  
Tracking bug for one-off thread conversion:  [bugzilla](https://bugzilla.mozilla.org/show_bug.cgi?id=1595241)
