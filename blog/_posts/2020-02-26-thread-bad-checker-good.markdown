---
layout: post
title:  "Deprecating NS_NewNamedThread"
date:   2020-02-28 10:00:00 -0800
categories: blog
---

For the past year or so I've spent a lot of time thinking about how we use threads in Firefox. There are a few initiatives to improve our threading picture - [Moving the Spidermonkey helper thread jobs into xpcom](https://bugzilla.mozilla.org/show_bug.cgi?id=1501438), using a [general-purpose background thread pool](https://bugzilla.mozilla.org/show_bug.cgi?id=1595241), and so on. We still run a lot of one-off threads in every content process. Don't get me wrong - async is good! The problem lies in when you have a *lot* of async and every single thread has its own way of doing things. Specifically, we do a lot of one-off threads that have one purpose and may even stick around forever after doing their job.

There are other initiatives to address other threading issues, but my goal here is to create *process*. A lack of process means there are likely over a hundred one-off threads in our code, and every single one has its own way of doing async. Putting a cap on threads and encouraging the use of a general-purpose background thread pool can help us reduce our overall thread count and prevent starvation along with creating a regular standard for how we do our async work.

A process to control the addition of new threads may look something like this:

* Identify existing instances of `NS_NewNamedThread`
* Prevent the addition of new ad-hoc threads
* Create an approval system for threads that absolutely must be added

The [no-new-threads](https://bugzilla.mozilla.org/show_bug.cgi?id=1613440) checker prevents new uses of `NS_NewNamedThread` in builds to address these cases. It's runnable as a part of the clang static analyzer checks enabled with the `--enable-clang-plugin` flag, which is enabled in automation. When your build fails on a thread, it looks something like this:

     0:15.75 error: Thread name not recognized. Please use the background thread pool.
     0:15.75   mozilla::Unused << NS_NewNamedThread("a bad name", getter_AddRefs(thread),
     0:15.75                      ^
     0:15.75 note: NS_NewNamedThread has been deprecated in favor of background task dispatch via NS_DispatchBackgroundTask and NS_CreateBackgroundTaskQueue. If you must create a new ad-hoc thread, have your thread name added to ThreadAllows.txt.

Of course, if we just run a checker like this with no knowledge of what uses of `NS_NewNamedThread` are already out there, we'll run into errors from the existing thread names. The checker therefore adds two *allow lists* to allow certain files and thread names to make use of `NS_NewNamedThread`.

### Adding to the allow lists
The goal of the checker is not to cut down on uses of `NS_NewNamedThread` by restricting who can use it to those whose thread names are in the allow list. The checker introduces two new .txt files - `ThreadAllows.txt` and `ThreadFileAllows.txt` - which the checker parses to generate the lists of threads and entire files to let through. The former contains names of threads which it will ignore, while the latter contains entire files to overlook. **You only need to add to one list** - the reason why we have two lists is to keep track of files where the *entire* file should be ignored by the checker, rather than just thread names. This is necessary for cases where the checker might not completely understand a thread name.

If you aren't sure which of the above lists your thread falls into, consider the following:

* If your thread name is a string literal, add the name to `ThreadAllows.txt`. For example, if your thread was named `"Awesome Thread"`, then you would add `Awesome Thread` to the text file.
* If your thread name is anything else (e.g. a member variable or method) then the checker may not be able to work out what it is. Add your file name to `ThreadFileAllows.txt`. Don't worry about the full path - the checker only looks at the file name. It will now ignore the entire file. For example, you want anything you do in `path/to/Foo.h` to be ignored. You would add `Foo.h` to `ThreadAllows.txt`.

The checker reads in a generated file built off of these .txt files, and uses the lists to determine which names and files to ignore.

#### Possible issues with the name lists
The checker is intended to be simple and easy to use, but it's possible that it returns a build error unexpectedly. If this is the case, check the following things:

**Does your thread name end in `.cpp` or `.h`?**  
When the program generating the allow lists builds the list, it recognizes anything ending in `.cpp` or `.h` as a *c++ file* and puts it in the file list. In practice, threads probably shouldn't be named after files.

**Does the checker understand the name of your thread?**  
It is completely possible that the AST matcher couldn't get a string literal out of the name args. If this is the case, you're going to get an error on the name. Some examples the checker may not understand:

    NS_NewNamedThread(runnable->GetThreadName(),
                      getter_AddRefs(newThread), runnable);
    NS_NewNamedThread(mName, getter_AddRefs(mThread));

In these cases, when the checker doesn't get it, it's better to just add the file name to `ThreadFileAllows.txt` instead. That doesn't mean it's encouraged to look over whole files, so if it's possible to feed the checker something it can understand then the practice is much preferred. For example:

    // The checker interprets this thread name as "Checker Test".
    NS_NewNamedThread("Checker Test", getter_AddRefs(aThread));

If your issue seems unrelated to or these don't work, it doesn't hurt to [file a bug](https://bugzilla.mozilla.org/enter_bug.cgi?product=Firefox%20Build%20System&component=Source%20Code%20Analysis) against the checker.

### Some fun technical details
*Note: If you don't care to hear about how the checker works, go ahead and skip this section.*  
The `no-new-threads` checker uses a custom matcher and a generated header file to read in and match thread names to instances of `NS_NewNamedThread`. It looks specifically for first-party code that contains a `CallExpr` with the name `NS_NewNamedThread`, but is *not* on the thread allowlist, like so:

    allOf(isFirstParty(),
          callee(functionDecl(hasName("NS_NewNamedThread"))),
          unless(isInAllowlistForThreads()))

To verify whether a thread name is allowed, it looks to see if some instance of `NS_NewNamedThread` that it found in this step is acceptable against its allowlists. It does this by getting the first arg (the name) of a `CallExpr` and casting it to a `StringLiteral` that it will be able to compare to a static array of names. It can accomplish this by using a generated header file:

    // This file was generated by generate_thread_allows.py. DO NOT EDIT.
    static const char *allow_thread_files[] = {
      "FooBar.cpp",
      "Foo.h"
    };
    static const char *allow_thread_names[] = {
      "Foo",
      "Bar"
    };

By iterating over the lists in the header file, the matcher can determine first if it cares about the file the associated `CallExpr` appears in, and if it does, if the name of the thread is on its list of allowed threads. Doing this is not necessarily the *fastest* way to find an error, but with only a little over a 100 instances of NS_NewNamedThread (a number that will be shrinking) I don't expect the checker to produce any significant slowdown on builds.

When the matcher does find an issue, the checker will raise an error that fails the build. It contains the location of the function call and a note explaining the error (see above).

### Future work
There's a lot of potential in a thread deprecator! I can see this expanded into limiting creation of new `LazyIdleThread`s (usually this work can just go to a background thread), uses of `StreamTransport` that don't really need calls into StreamTransport's api (essentially, using it as a background thread pool, which we did frequently before the introduction of `BackgroundEventTarget`), and limiting the creation of non-beneficial thread pools. In the mean time, let's focus on [converting some threads](https://bugzilla.mozilla.org/show_bug.cgi?id=1595241) and making the most out of our background threads. :D

### See also:
Checker source code: [1](https://hg.mozilla.org/mozilla-central/file/tip/build/clang-plugin/NoNewThreadsChecker.h) [2](https://hg.mozilla.org/mozilla-central/file/tip/build/clang-plugin/NoNewThreadsChecker.cpp) [3](https://hg.mozilla.org/mozilla-central/file/tip/build/clang-plugin/ThreadAllows.py) [4](https://hg.mozilla.org/mozilla-central/file/tip/build/clang-plugin/ThreadAllows.txt) [5](https://hg.mozilla.org/mozilla-central/file/tip/build/clang-plugin/ThreadFileAllows.txt)  
Convert one-off threads to the background pool: [Bugzilla](https://bugzilla.mozilla.org/show_bug.cgi?id=1595241)  
Bug for the new checker: [Bugzilla](https://bugzilla.mozilla.org/show_bug.cgi?id=1613440)
