---
layout: post
title: Hurdles Migrating to SPM
---

I was recently working on a project that was divided into modular frameworks by having a framework target for each module in the project. In order to take some of these modules and share it between projects, I thought I would break it out and make a Swift Package out of it. It didn't go quite as smoothly as I would have hoped. Let's take a look at some of the hurdles I ran into along the way.

*I may continue adding to this document as I progress along the project.*

<!--excerpt-->

## Background

The setup of this project is fairly straightforward. There are frameworks for UI components, Models, and a Base framework that is shared throughout everything. It's set up something like this:

```
App/
  Sources/
Libraries/
  Base/
  Models/
  UI/
```

If it helps to visualize, the dependency graph looks something like this:

<center><img src="/public/images/20210703/graph.png" style="width: 33%;" /></center>

It's this Base framework that we are trying to break out into a Package using the Swift Package Manager.

It's also worth noting that this is an existing project with a mixed codebase with a large majority written in Swift, but some important components still written in Objective-C, so some of the code in the frameworks has to be written in a way that it can be used from Objective-C.

## Hurdle 1: SPM Doesn't Handle Mixed-Language Codebases

While trying to move the Base framework into SPM, it quickly became apparent that I wouldn't be able to keep it the same as it existed as the Xcode framework, because SPM doesn't handle a codebase that has both Swift *and* Objective-C (or C, or C++, or...). 

It was easy to break it up into two different frameworks (which SPM *can* handle), but it felt weird to have BaseSwift and BaseObjC. The Objective-C code was enough of a minority that it was easy enough to rewrite it in Swift without too much trouble.

This was pretty easy in my situation, but I can imagine it would become a lot harder with larger codebases.

## Hurdle 2: SPM Only Has "Release" and "Debug" Configurations

When you use SPM, it will only build your libraries in the "Release" or "Debug" configuration. When it's done being built, it's added to your Derived Data in a `$CONFIG-$TARGET` directory, for example `Release-iphoneos`. 

This can be an issue if your project has different configuration names such as `AppStore` or `Beta`. The Derived Data for these will be stored in `AppStore-iphoneos` and `Beta-iphoneos` respectively, and you will run into and error where Xcode will not be able to find the SPM Package.

As a workaround, [some people have suggested a workaround](https://stackoverflow.com/a/58307605) of copying the Derived Data for the SPM package into the Derived Data for your config as a build phase before compiling your code:

```shell
if [ -d "${SYMROOT}/Release${EFFECTIVE_PLATFORM_NAME}/" ] && [ "${SYMROOT}/Release${EFFECTIVE_PLATFORM_NAME}/" != "${SYMROOT}/${CONFIGURATION}${EFFECTIVE_PLATFORM_NAME}/" ] 
then
  cp -f -R "${SYMROOT}/Release${EFFECTIVE_PLATFORM_NAME}/" "${SYMROOT}/${CONFIGURATION}${EFFECTIVE_PLATFORM_NAME}/"
fi
```

To be clear, I didn't run into this problem directly, but I stumbled upon this [StackOverflow post](https://stackoverflow.com/questions/57165778/getting-no-such-module-error-when-importing-a-swift-package-manager-dependency) a number of times while trying to solve problems. Apparently it's a [known issue](https://forums.swift.org/t/custom-build-configuration-names/29167), and will hopefully be fixed soon.

## Hurdle 3: Module Import Issues and Exposing Objective-C Types

Our Base framework has some types that are exposed to Objective-C using `@objc` on the types.

The Base framework was migrated to SPM.

Our UI components framework was able to import the Base framework and use types from it. It also contains some files written in Objective-C, and so it has an umbrella header (the Objective-C header with the same name as the framework). 

Our UI components framework also exposes new things to Objective-C based on types in the Base framework. 

With this setup, anything that depends on the UI component framework will not be able to use it, and you'll be shown some confusing error messages:

```
<module-includes>:2:9: note: in file included from <module-includes>:2:
#import "Headers/UIComponents-Swift.h"
        ^
/Users/caesar/Library/Developer/Xcode/DerivedData/MyApp-amrpwkdqvxmjaebtbsshwivyfauk/Build/Products/Debug-iphonesimulator/UIComponents.framework/Headers/UIComponents-Swift.h:196:9: error: module 'BaseLibrary' not found
@import BaseLibrary;
        ^
<unknown>:0: error: could not build Objective-C module 'UIComponents'
```

"module 'BaseLibrary' not found" and "could not build Objective-C module 'UIComponents'", two errors that you *don't* see when building the UI framework alone.

The UI framework was certainly able to fine the Base framework, so then why does this error only appear when building a target that *depends on* the UI framework?

I think the key to the answer lies in *where* the error is appearing.

Clicking "Module 'BaseLibrary' not found" error brings you to the `UIComponents-Swift.h` header, the auto-generated header for the UI components framework that exposes the Swift types to Objective-C.

I also get the same error if I `@import BaseLibrary;` in any of my Objective-C header files in the UI components framework. So I can't be importing it in any of my headers included in the umbrella header, so where is this rogue import coming from?

As mentioned above, the UI components framework is exposing new things to Objective-C from Swift code based on things that happened in the Base framework.

This can be, for example, adding a type that conforms to a protocol defined in the Base framework:

```swift
// Base
@objc public protocol BaseThing {
    func doTheThing()
}

// UI
import BaseLibrary
@objc public class UIThing: NSObject, BaseThing {
   ...
}
```

I believe that since some of my Swift code here relies on something defined in the Base framework, the auto-generated `UIComponents-Swift.h` header *had to* include the import to the Base framework in order for the system to understand what it's supposed to be.

This only applies if it's expose *new things* in Swift and exposing it to Objective-C, because the import becomes required. It won't apply if you're just returning a type that already exists in the Bases library. The auto-generated header will just forward declare the type it needs. 

I don't have a full list of what it means to "expose new things" to Objective-C, but I have found that it will apply to at least these cases, provided that they have a `@objc` to expose them:

* Creating a new type that conforms to a protocol in the Base framework
* Creating a new type that subclasses a class from the Base framework
* Adding a method as an extension to a type that exists in the Base framework

It also only applies if you have an umbrella header in your framework. Delete that, and the issue goes away.

## Conclusion

While it seems like the Swift Package Manager has made a lot of progress, it seems like there are still some hurdles integrating it into existing codebases. Not to say that it can't be done, but there may be some hurdles along the way. It's not terribly surprising that one of the difficult points has to do with multilingual codebases. It's not the [first time that I have encountered it myself](https://cjwirth.com/tech/circular-references-swift-objc), either.

I will say, though, that there has been a lot of progress done on SPM since the [last time I looked into using it](https://cjwirth.com/tech/using-xcode-and-spm-together). I'm glad to know that it keeps getting better, and that there is an Apple-provided solution for dependency management, even if it's not the easiest tool for me to use at the moment.

