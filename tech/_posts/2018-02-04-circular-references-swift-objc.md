---
layout: post
title: Circular References Between Swift and Objective-C
---

Apple has put a lot of work into making it possible to use [Swift and Objective-C in the Same Project](https://developer.apple.com/library/content/documentation/Swift/Conceptual/BuildingCocoaApps/MixandMatch.html) without too much hassle. There are still some challenges that arise when working in a mixed codebase.

Some of these challenges arise due to circular dependencies between Swift and Objective-C. I run across some of these situations from time to time, so I thought it might be worth writing them down.

<!--excerpt-->

## Swift class used in ObjC class used in Swift

In Objective-C, we have the convention to prefix our type names, in order to prevent type name collisions. This isn't an issue in Swift, so I often like to drop the prefix when I'm writing classes in Swift.

Luckily, the `@objc` annotation gives us the ability to change the name that will be used when imported into Objective-C:

~~~ swift
// This class will be called `CJWUser` when being used from Objective-C.
// When using it from Swift, it will just be called `User`.
@objc(CJWUser)
class User { ... }
~~~

To use this class from Swift, we will need to import our project's Xcode-generated header into our file. We might need to add it to the header file, for example if we are adding it as a property to a class. In that case, we need to forward declare the class:

> However, to avoid cyclical references, don’t import Swift code from within the same module into an Objective-C header (.h) file. Instead, you can forward declare a Swift class or protocol to reference it in an Objective-C interface.

We should forward declare the class in the header, but then import the Swift file in the implementation:

~~~ objc
// UserProfileVC.h

// Tells the compiler that there will be class called `CJWUser`.
@class CJWUser;

@interface CJWUserProfileVC: UIViewController

/// Using the user class defined in Swift.
@property (nonatmoic, strong) CJWUser *user;

@end
~~~

~~~ objc
// UserProfileVC.m

// Import the auto-generated header with our Swift types.
#import "MyApp-Swift.h"

- (void)setUser:(CJWUser *)user {
    // Here we can use the normal properties of the user
}
~~~

Great! Now we can create our view controller and display the data to the user:

~~~ objc
CJWUserProfileVC *profileVC = [[CJWUserProfileVC alloc] init];
profileVC.user = user;
[presenter presentViewController:profileVC animated:YES completion:nil];
~~~

But what if we wanted to do this from Swift? It seems like we're doing everything right, but we get an error:

~~~ swift
let profileVC = CJWUserProfileVC()
// ERROR: Value of type 'CJWUserProfileVC' has no member 'user'
profileVC.user = user
presenter.present(profileVC, animated: true, completion: nil)
~~~

Just looking at the code, it seems like it should work. We are able to use the `CJWUser` from Objective-C. We are able to instantiate the `CJWUserProfileVC` from Swift. Why isn't it able to detect the `user` in Swift -- it's a class that we originally defined in Swift, after all!

But that's not entirely true is it? We defined the `User` type in Swift. We only _declared_ the `CJWUser` class in Objective-C. By declaring the `@class CJWUser;`, we didn't actually create type, but were telling the compiler that there _will be_ a class with that name. We can see the definition in the `MyApp-Swift.h` that Xcode automatically generates for us:

~~~ objc
SWIFT_CLASS_NAMED("User")
@interface CJWUser : NSObject
- (nonnull instancetype)init OBJC_DESIGNATED_INITIALIZER;
@end
~~~

So it seems like Swift just knows about a type called `User`, and Objective-C just knows about a type called `CJWUser`. I'm not entirely sure why the compiler can't connect these dots, or why this is the place it fails.

To fix this all we need to do is to not rename the type when we add the Objective-C alias:

~~~ swift
@objc
class CJWUser: NSObject { ... }
~~~

Then we'll be able to use the `CJWUser` class from both Swift and Objective-C.

But what about our convention of not using the prefix in Swift code? What if we already have a lot of references to `User`? Luckily, Swift provides typealiases that you can use to make new names for types:

~~~ swift
typealias User = CJWUser
@objc
class CJWUser: NSObject { ... }
~~~

Now you can still use `CJWUser` in Objective-C, while still using `User` in Swift:

~~~ objc
CJWUser *user = [[CJWUser alloc] init];
CJWUserProfileVC *profileVC = [[CJWUserProfileVC alloc] init];
profileVC.user = user;
~~~

~~~ swift
let user = User()
let profileVC = CJWUserProfileVC()
profileVC.user = user
~~~

You might be able to come up with a better solution to a problem like this by thinking about your architecture, but maybe you have some legacy code that you're not currently in a place to refactor. I've found this to be a pretty painless way to get around it.

Another approach might involve being a little more Swift-like and using protocols. One problem, however, is that it can be tricky to get an Objective-C class to conform to a Swift protocol.

## Objective-C Conformance to Swift Protocols

For the same reasons as above, if you need to include a type conforming to a Swift protocol to an Objective-C header file, you have to forward declare the type, rather than importing the Swift header:

~~~
// CJWUserProfileVC.h

// Don't import this, lest you create circular references!
// #import "MyApp-Swift.h"

@protocol CJWPerson;

@interface CJWUserProfileVC: UIViewController
@property (nonatomic, strong) id<CJWPerson> user;
@end
~~~

This works great, because now we can use a protocol type on an Objective-C class that we can use from both Swift and Objective-C.

But what happens if we want to have a class that conforms to that protocol? This poses a problem.

The documentation says:

> Forward declarations of Swift classes and protocols can only be used as types for method and property declarations.

> An Objective-C class can adopt a Swift protocol in its implementation (`.m`) file by importing the Xcode-generated header for Swift code and using a class extension.

What this is saying, which we can confirm in Xcode ourselves, is that we can't actually do this:

~~~ objc
@protocol CJWPerson;

// WARNING: Cannot find protocol definition for 'CJWPerson'
@interface CJWUser: NSObject <CJWPerson>
@end
~~~

I was _technically_ able to do this in my sample program, because it's just a warning and not an error, but this might not be true for larger, more complex situations.

Plus who likes having build warnings?

There are two fixes that we can do for this:

- Rewrite the protocol in Objective-C
- Add a function that returns `self` as the protocol type

Neither solutions feel completely satisfactory to me, but the second one feels more flexible.

To make it more concrete, this is how I've handled this situation:

~~~ objc
// CJWUser.h

@protocol CJWPerson;

@interface CJWUser: NSObject
- (id<CJWPerson>)asPerson;
@end
~~~

~~~ objc
// CJWUser.m
#import "MyApp-Swift.h"

@implementation CJWPerson

- (id<CJWPerson>)asPerson {
    return self;
}

@end
~~~

That way in Swift, I can just get a reference to the protocol type whenever I need it:

~~~ swift
let user: CJWUser = CJWUser()
let person: Person = user.asPerson()
let personProfileVC = PersonProfileVC()
personProfileVC.person = person
~~~

## In Closing

I'm grateful that Apple has put in so much work into making Objective-C and Swift work so nicely together. Even though there are a few things that can be difficult, there are usually ways to make it do what you want to do -- even if it involves a few workarounds sometimes!

If you know more about the Swift compilation workflow, and why some of these things work, please let me know! You can find me on [Twitter @cjwirth](https://twitter.com/cjwirth)