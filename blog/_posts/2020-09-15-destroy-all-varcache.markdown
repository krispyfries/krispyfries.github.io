---
layout: post
title:  "Removal of VarCache preferences"
date:   2020-09-15 05:00:00 -0800
categories: blog
---

To conclude an multi-year effort to remove legacy preference mechanisms, the last traces of [VarCache prefs](https://bugzilla.mozilla.org/show_bug.cgi?id=1448219) were finally removed this month and the [VarCache machinery](https://bugzilla.mozilla.org/show_bug.cgi?id=1642727) was removed. A lot of people don't know what these were (I didn't) but it's a pretty big deal to see them gone!

## What is VarCache?
VarCache was the old way of defining prefs in c++ code. You'd pass it a mirror variable and a pref name, which looked something like this:

    Preferences::AddBoolVarCache(&mPrefValue, A_PREF_NAME, true);

There are a few problems with this mechanism, though.
  * **It's slow**. When you have hundreds of these, and they're all looking up pref names, you can expect a performance hit compared to using something built statically into the binary.
  * **The mechanism was too general.** One of the problems with VarCache prefs was how many ways they could be used wrong. This led to a lot of duplicate logic, lots of prefs that were defined but never accessed, lots of prefs whose mirror variables were manually changed. Plus, if you did something like made a typo in the pref name, you'll just create a *new pref entirely*, which isn't very good when, for example, you've got prefs spelled with both ['referer'](https://searchfox.org/mozilla-central/rev/30e70f2fe80c97bfbfcd975e68538cefd7f58b2a/modules/libpref/init/StaticPrefList.yaml#7996) and ['referrer'](https://searchfox.org/mozilla-central/rev/30e70f2fe80c97bfbfcd975e68538cefd7f58b2a/modules/libpref/init/StaticPrefList.yaml#3295).
  * **No checks for thread and type safety.** I spend a lot of time [thinking about threads](https://krispyfries.github.io/blog/2020/02/28/thread-bad-checker-good.html) and in this case the best policy around threadsafe programming is abstraction - never expect every single user of a mechanism to handle their own thread safety. Suffice to say there were a few data races in the VarCache code that needed to be cleaned up.
  * **Instructions unclear, can't find my mirror variable.** Like most legacy mechanisms, VarCache documentation wasn't thorough enough to cover a lot of the ways people want to use it. The new [static prefs](https://searchfox.org/mozilla-central/rev/30e70f2fe80c97bfbfcd975e68538cefd7f58b2a/modules/libpref/init/StaticPrefList.yaml) are much better documented.

## Replacing VarCache
We have a way now to define prefs [entirely in the binary](https://bugzilla.mozilla.org/show_bug.cgi?id=1436655). It's faster, easier to use, and (most importantly to me) thread safe. While replacing a single VarCache with a Static Pref wouldn't have an especially noticeable benefit to performance, replacing all of them would add up. (I don't have the exact numbers on hand, but more of a "static prefs are faster than using a dynamic VarCache lookup for obvious reasons". Additionally, static prefs can be used from Rust.)

When static prefs were introduced, we had *hundreds* of VarCache prefs to get rid of. It would be easy if these could just be replaced with a script; but since VarCache was using mirror variables in the c++ code, each one had to be examined individually and in its own context to determine how to handle it. That meant some were straightforward (delete the VarCache call, replace `mPrefValue` with a call into static prefs) but there were just as many that had special cases where some of the existing calls had to be refactored. Some were turned into constants and some were turned into callbacks. It was certainly nice to see so many static `init()` functions removed from startup, because all they did was init VarCache prefs!

It took a while to whittle away at them, but now that they're gone I'm happy to say that the machinery doesn't exist anymore. If you want to use prefs in c++, I'd recommend using static prefs. :)

## What now?
Well, you can't use VarCache anymore, so what do you do? For the most part we replaced them with static prefs, but there are still the [Preferences::Get](https://searchfox.org/mozilla-central/rev/f4b4008f5ee00f5afa3095f48c54f16828e4b22b/modules/libpref/Preferences.cpp#4560-4596) methods and the [callback mechanism](https://searchfox.org/mozilla-central/rev/f4b4008f5ee00f5afa3095f48c54f16828e4b22b/modules/libpref/Preferences.cpp#4953) for cases that simply do not apply to static prefs.

## Some strange highlights
  * Some prefs weren't even being used anymore and [could be removed entirely](https://hg.mozilla.org/mozilla-central/rev/0c7b5a9964dd).
  * Sometimes, [we don't actually want the user to access a pref](https://bugzilla.mozilla.org/show_bug.cgi?id=1572534#c3), but we didn't have any other kind of persistent storage to take advantage of at the time.
  * This pref had a note to [keep it in sync with some macros](https://hg.mozilla.org/mozilla-central/rev/a03ca1bbfefa) - manually!

While each individual pref conversion doesn't feel like a big change, it's so nice to see it all come together at the end. Sometimes improving Firefox takes little steps which all add up to make something much better.
