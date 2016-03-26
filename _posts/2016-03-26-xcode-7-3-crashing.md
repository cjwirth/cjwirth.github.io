---
layout: post
title: Xcode 7.3 Crashes With Non-ObjC Properties in Classes
---

On March 21, 2016, the world celebrated the release of Xcode 7.3 Official! 🎉

If you're anything like me, you quickly downloaded the DMG ([not from the AppStore](http://ericasadun.com/2016/03/22/xcode-upgrades-lessons-learned/)), updated all your code, your dependencies, excitedly ran your app for the first time in this bright, new world, and...

It crashed. Not only that, but the stack trace was anything but helpful. What even _is_ `realizeClass(objc_class*)`? 

I'm going to show you how to fix this, without having to resort to going back to Xcode 7.2.

<!--excerpt-->

## MVP to Crash and Burn

To illustrate the problem, I'm going to be using a very small program that still crashes when run. The entire source code fits inside a [tweet](https://twitter.com/cjwirth/status/713629205636878336)!

Here it is, in all its less-than-140-characters-glory:

~~~ swift
enum Enum {
    case Foo
}

class Class {
    var bar = Enum.Foo
}

let object = Class() // 💥 EXC_BAD_ACCESS
~~~

> "But Caesar, that looks like perfectly valid Swift code! And it's not even doing anything, what could possibly go wrong?"
>
> -- <cite>You, Dear Reader</cite>

I know, right?! That was my reaction too! Except I was dealing with my entire app, and not this small little MVP. I had literally no idea what was going on.

So, what was the first thing I did to see what was going wrong? print out the backtrace!

~~~
(lldb) bt
* thread #1: tid = 0x2c7f64, 0x00007fff9aa1765d libobjc.A.dylib`realizeClass(objc_class*) + 1147, queue = 'com.apple.main-thread', stop reason = EXC_BAD_ACCESS (code=2, address=0x1002431d8)
    frame #0: 0x00007fff9aa1765d libobjc.A.dylib`realizeClass(objc_class*) + 1147
    frame #1: 0x00007fff9aa18efd libobjc.A.dylib`_class_getNonMetaClass + 127
    frame #2: 0x00007fff9aa18d00 libobjc.A.dylib`lookUpImpOrForward + 171
    frame #3: 0x00007fff9aa13591 libobjc.A.dylib`objc_msgSend + 209
    frame #4: 0x0000000100233b9c MyCrashingApp`type metadata accessor for Class + 44 at main.swift:0
  * frame #5: 0x00000001002339db MyCrashingApp`main + 75 at main.swift:17
    frame #6: 0x00007fff9db745ad libdyld.dylib`start + 1
    frame #7: 0x00007fff9db745ad libdyld.dylib`start + 1
~~~

Wow! That is incredibly confusing and unhelpful to someone like me. It's only touching my app in 2 frames, where I am creating the `object` object. There shouldn't be anything wrong with that?

# Pinpointing the Problem Class

At this point I didn't really know what to do. So what did I do? I gave up and complained about it on Twitter, like any self-respecting developer would, of course. 😉

Luckily, it wasn't long until help was on the way! 

[![Christopher Rogers teaches me about OBJC_PRINT_CLASS_SETUP](/public/images/20160326/crogers-twitter-1.png)](https://twitter.com/christorogers/status/712597337361649664)

This is a great help! If we do something with `OBJC_PRINT_CLASS_SETUP`, then we can find out what class is actually causing trouble in our system. 

`OBJC_PRINT_CLASS_SETUP` is an Objective-C environment variable. You can find it online [where Apple keeps its open source stuff available](http://www.opensource.apple.com/source/objc4/objc4-646/runtime/objc-env.h). The comment next to it is "log progress of class and category setup." 

We can turn on this environment flag by editing our project's scheme. Once in the scheme editing screen, we go into the _Run_ action, and over to the _Arguments_ tab.

Once there, we can add an environment variable with the name `OBJC_PRINT_CLASS_SETUP` and value of `YES`.

[![Adding OBJC_PRINT_CLASS_SETUP to Scheme](/public/images/20160326/editing-scheme.png)](/public/images/20160326/editing-scheme.png)

Now we can run our app one more time, and see if we can find our problematic class! 

~~~
objc[5399]: CLASS: found 1568 classes during launch
objc[5399]: CLASS: realizing class 'OS_dispatch_queue'  0x1004d12a8 0x1004cfbc8
objc[5399]: CLASS: realizing class 'OS_dispatch_object'  0x1004d11d0 0x1004cfa80
objc[5399]: CLASS: realizing class 'OS_object'  0x1004d16c8 0x1004cf638
objc[5399]: CLASS: realizing class 'NSObject'  0x7fff7c73a0f0 0x7fff7c739b18
objc[5399]: CLASS: realizing class 'NSObject' (meta) 0x7fff7c73a118 0x7fff7c739710
objc[5399]: CLASS: methodizing class 'NSObject' (meta)
objc[5399]: CLASS: methodizing class 'NSObject' 
...
-- many, many, lines later --
...
objc[5399]: CLASS: realizing class 'MyCrashingApp.Class' (meta) 0x100282f68 0x10026ee30
objc[5399]: CLASS: realizing class 'SwiftObject' (meta) 0x10026f790 0x10026e170
objc[5399]: CLASS: realizing class 'SwiftObject'  0x10026f768 0x10026e5c8
objc[5399]: CLASS: methodizing class 'SwiftObject' 
objc[5399]: CLASS: methodizing class 'SwiftObject' (meta)
objc[5399]: CLASS: methodizing class 'MyCrashingApp.Class' (meta)
objc[5399]: CLASS: realizing class 'MyCrashingApp.Class'  0x100282fa0 0x10026eea0
~~~

Wow! That's a whole lot of output! It's all pretty repetitive, too. I'm no expert on the Objective-C runtime, but it looks like it's probably loading all the classes when they are needed.

The last one it loaded before it crashed? Yes, that's our `Class` class that we defined up above. Well... of course it is.

![I don't know what I expected](/public/images/20160326/idk-expected.jpg)

So what have we learned so far? When Objective-C is setting up our classes or something, it fails to load our bad class for some reason. That must mean that if we just fix our class, then things should be back to normal right!

## But How Do I Fix The Class?

Luckily, I had another tweet waiting for me that taught the exact problem.

[![Christopher Rogers Teaches Me The Real Problem](/public/images/20160326/crogers-twitter-2.png)](https://twitter.com/christorogers/status/712990266609676290)

> Hey! This Caesar guy didn't do anything! [@christorogers](https://twitter.com/christorogers) did all the work and actually figured everything out!
> 
> <cite>-- Dear Reader, Truly Correct</cite>

Yeah, I know. I owe this entire blog post to Christopher Rogers. He saved me many hours of confused debugging and searching. He's truly a legend, and I'm super grateful! 🚀

Let's take a closer look at our MVP app:

~~~ swift
enum Enum {
    case Foo
}

class Class {
    var bar = Enum.Foo
}
~~~

The first stored property on `Class` is of type `Enum`, which is an `enum` -- this is a type that isn't visible from Objective-C!

We'll go an move the non-Objective-C property out of the first position. We only have one property on our class, but that probably won't happen in a real app. 

Since we only have one property, we'll have to add some unused property to our class.

~~~ swift
class Class {
    let bugfix = ""
    var bar = Enum.Foo
}
~~~

Now, if we try running our app...

~~~
Program ended with exit code: 0
~~~

Success!

## Summary

If you're having this problem in your project after upgrading to Xcode 7.3, there are just a few steps you have to take to fix it:

1. Enable the `OBJC_PRINT_CLASS_SETUP` environment variable
1. Find the problematic class by finding the last class it loaded before crashing
1. Move properties not visible to Objective-C out of the first position
1. Rinse and Repeat until all classes are fixed

## Let's Get This Fixed!

This is pretty clearly a bug in the development tools. There shouldn't be anything wrong with having whatever you want as your first stored property in your class!

To help get this fixed, file a bug report! Duplicate the bug report below (get the info you need from the Open Radar report if necessary). 

Or if you are feeling ambitious, take a look at the Swift compiler source code and see if you can fix it! There's a bug already filed on the Swift JIRA.

- ⭐️ [rdar://25330746](rdar://25330746) - Official Radar - Duplicate This!
- [Swift Bug Report](https://bugs.swift.org/browse/SR-1055) - Check on progress
- [Open Radar Report](http://www.openradar.me/25330746) - Unofficial bug track

