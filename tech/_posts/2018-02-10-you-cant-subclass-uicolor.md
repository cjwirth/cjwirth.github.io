---
layout: post
title: You Can't Subclass UIColor
---

I lied. You can. But it's not as straightforward as you might expect.

When implementing an iOS app, there are a lot of classes from UIKit that we can subclass: `UIView`, `UIViewController`, `UIButton`. This is probably familiar to any iOS developer. Some classes aren't as straightward to subclass. `UIColor` is one of them, and we're going to look into why.

<!--excerpt-->

## tl;dr

The reason is Class Clusters. You can do it by making your subclass a wrapper around a `UIColor` instance.

## Motivation

A class like `UIColor` seems to have everything you need, so why would you want to subclass it? The documentation even states:

> Most developers have no need to subclass `UIColor`. The only time doing so might be necessary is if you require support for additional color spaces or color models.

You can already create a color object with any RGB value you want, so what kind of functionality would you want to add?

One example is if you have a set number of colors in your palette, and you want to enforce that only those colors are used in custom UI elements. Each of these colors might have a shade (e.g. a dark, medium, and light variation). Since we're using UIKit, we'll have to deal with `UIColor` in the end, so it would also be great if we could just pass our objects into properties that normally take `UIColor` parameters.

So what we're looking for is a type that:

- Prevent use of arbitrary `UIColor` objects when we can
- Can be easily used as `UIColor`
- Have added properties or functionality (e.g. shades)

## First Attempt

My first thought was to do some protocol-oriented approach, where you would have a protocol that provided the shades, as well as properties to the underlying `UIColors` to use.

~~~ swift
// Anything that can be provide a color.
protocol ColorProvider {
    var color: UIColor { get }
}

// Colors in our palette. They have a set number of shades.
protocol MultiShadedColor {
    var light: ColorProvider { get }
    var medium: ColorProvider { get }
    var dark: ColorProvider { get }
}

// But by default, we can use the MultiShadedColor as a ColorProvider as well.
extension MultiShadedColor: ColorProvider {
    var color: UIColor {
        return medium.color
    }
}
~~~

We could create a class `Color`, and give it the necessary properties, and then make our custom UI elements use `ColorProvider` or `MultiShadedColor` whenever we wanted restrict it to only colors in our palette.

One other thing that I think could be better is that, since it's not `UIColor`, we can't use the type inference in places where would like to:

~~~ swift
let view = UIView()

// Using UIColor
view.backgroundColor = .red
// Using Color
view.backgroundColor = Color.specialRed.color
~~~

With this, it's also easy to add a protocol extension to `UIColor` which nullifies our hopes of restricting our types to `UIColor`:

~~~ swift
extension UIColor: ColorProvider {
    var color: UIColor {
        return self
    }
}

// No problem doing this now, even though we don't want arbitrary colors:
view.backgroundColor = UIColor.red.color
~~~

I don't think the protocol oriented approach is a bad way to go. It just seems like subclassing might be a good approach. Really, all I want to do is have a `UIColor` with a few properties added to it!

## Subclassing Roadblocks

It seems like a really easy task to do:

~~~ swift
class PaletteColor: UIColor {
    let light: UIColor
    let medium: UIColor
    let dark: UIColor

    init(light: UIColor, medium: UIColor, dark: UIColor) {
        self.light = light
        self.medium = medium
        self.dark = dark

        // Medium is our default color, so we will make self the same as `medium`.
        var r: CGFloat = 0, g: CGFloat = 0, b: CGFloat = 0, a: CGFloat = 0
        medium.getRed(&r, green: &g, blue: &b, alpha: &a)
        super.init(red: r, green: g, blue: b, alpha: a)
    }
}
~~~

Done! Well, not exactly. There are some required initializers that you have to implement. Adding those to the code, you see that one of the required initializers _can't_ implemented in Swift:

~~~ swift
// Declarations from extensions cannot be overridden yet
@nonobjc required convenience init(_colorLiteralRed red: Float, green: Float, blue: Float, alpha: Float) {
~~~

I don't plan on creating new instances once the palette is implemented, so I'm fine if we don't have an initializer. I'd be okay using a workaround with factory methods and private properties that are implicitly unwrapped.

~~~ swift
class PaletteColor: UIColor {

    // Defined colors in our palette.
    static let americaRed = PaletteColor.color(light: .red, medium: .white, dark: .blue)

    // Public, non-optional properties.
    var light: UIColor { return _light }
    var medium: UIColor { return _medium }
    var dark: UIColor { return _dark }

    // Private backing variables.
    private var _light: UIColor!
    private var _medium: UIColor!
    private var _dark: UIColor!

    // Private factory method so we can workaround compiler problems.
    private static func color(light: UIColor, medium: UIColor, dark: UIColor) -> PaletteColor {
        // Medium is our default color, so we will make self the same as `medium`.
        var r: CGFloat = 0, g: CGFloat = 0, b: CGFloat = 0, a: CGFloat = 0
        medium.getRed(&r, green: &g, blue: &b, alpha: &a)

        let color = PaletteColor(red: r, green: g, blue: b, alpha: a)
        color._light = light
        color._medium = medium
        color._dark = dark
        return color
    }
}
~~~

This compiles just fine. We will run into problems with this at runtime, though, and it's not due to those implicitly unwrapped optionals.

Upon starting the app, we will crash in the factory method:

~~~ swift
private static func color(light: UIColor, medium: UIColor, dark: UIColor) -> PaletteColor {
    // Medium is our default color, so we will make self the same as `medium`.
    var r: CGFloat = 0, g: CGFloat = 0, b: CGFloat = 0, a: CGFloat = 0
    medium.getRed(&r, green: &g, blue: &b, alpha: &a)

    let color = PaletteColor(red: r, green: g, blue: b, alpha: a)

    // 💥: EXC_BAD_ACCESS (code=EXC_I386_GPFLT)
    color._light = light
    color._medium = medium
    color._dark = dark
    return color
}
~~~

The backtrace isn't too helpful either, but it indicates that we're writing to some piece of memory that we shouldn't be. That seems weird; we initialized a new instance of our `PaletteColor` class, and now we're trying to set a property that we have...

Checking out the debugger gives us a little more information, albeit something unexpected:

~~~
(lldb) po color
UIExtendedSRGBColorSpace 1 1 1 1
~~~

`UIExtendedSRGBColorSpace`? I was expecting something more like `<PaletteColor: 0x604000445e50>`! I wonder if it's even the type I'm expecting?

~~~
(lldb) po type(of: color)
UIDeviceRGBColor
~~~

That's not even anything documented? How am I getting an instance of a private Apple API by instantiating a class that I created?

## Class Clusters

There are a lot of types in the Cocoa frameworks that are [Class Clusters](https://developer.apple.com/library/content/documentation/General/Conceptual/CocoaEncyclopedia/ClassClusters/ClassClusters.html). There's `NSNumber`, `UIButton`, and any of the classes that have mutable counterparts like `NSString` or `NSArray`.

The idea of class clusters is similar to protocols. You can have a high level concept that has multiple concrete implementations, depending on the type. For example, `NSNumber` has different implementations as to whether it is representing a `BOOL`, `int`, `double`, etc.

It's quite similar to the [abstract factory pattern](https://en.wikipedia.org/wiki/Abstract_factory_pattern), in how you just ask the abstract base class for an object, and it gives you the correct implementation.

This can be easily implemented in Objective-C, where initializers are functions that can return anything they want, even if it's not `self`!

Let's imagine we were creating Earth, and all the wildlife on it. There a lot of Animals, and for they most part they do similar things, but we need to implement their digestive systems differently based on what they prefer to eat.

We could create a Class Cluster of Animals, and categorize them by what they prefer to eat:

~~~ objc
// Publicly facing abstract object. All object creation goes through this.
@interface Animal: NSObject

// Returns an `Animal` object that is equipped to eat `meat`, `plants`, or both.
- (id)initWithFoodPreference:(NSString *)foodPreference;
@end
~~~

Then in the private implementations, we could have the initializer return the correct concrete instance based on the given input.

~~~ objc
- (id)initWithFoodPreference:(NSString *)foodPreference {
    if ([foodPreference isEqualToString:@"meat"]) {
        return [[Carnivore alloc] init];
    } else if ([foodPreference isEqualToString:@"plants"]) {
        return [[Herbivore alloc] init];
    } else {
        return [[Omnivore alloc] init];
    }
}
~~~

With this knowledge, we can now realize why we'd be getting an unexpected class when calling certain initializers on our class.

Luckily, there is still a way to subclass abstract classes like `UIColor`.

## Disclaimer: you probably don't want to add classes to class clusters

I just want to take a step back and stay that it's probably not a great idea to go around subclassing abstract classes of class clusters. Stick to the classes that are _meant_ to be subclassed.

Most classes aren't written with subclassing in mind, so the abstract base class might have some private logic that Apple provides to its concrete subclasses, and if changes are made, it might have unintended side effects on your subclass.

Apple _does_ document two examples of [creating classes within a class cluster](https://developer.apple.com/library/content/documentation/General/Conceptual/CocoaEncyclopedia/ClassClusters/ClassClusters.html#//apple_ref/doc/uid/TP40010810-CH4-SW76), but if you're at a point where you're thinking of subclassing `NSMutableArray`, you might try looking for another abstraction.

## So You Really Wanna Subclass `UIColor`

Okay, you've talked me into it. We're going to add a new class to the class cluster by subclassing `UIColor`. Fine.

The way Apple recommends doing it is by creating a _composite object._ This basically means taking private instance of an existing class from the class cluster, and then just forwarding all messages to it, intercepting it for the functionality we need to change.

This makes it really easy on us. All we need to do is copy all the methods from the `UIColor` header into our class, and forward them to an `innerColor` that we hold on to. That will be the "true" color of our subclass.

Since our spec is that the `medium` shade should be the default color, we can just add a new property:

~~~ swift
private var innerColor: UIColor {
    return medium
}
~~~

and then forward the `UIColor` methods to it:

~~~ swift
@objc(CGColor) override var cgColor: CGColor { return innerColor.cgColor }

@objc(CIColor) override var ciColor: CIColor { return innerColor.ciColor }

@objc override func set() { innerColor.set() }

@objc override func setFill() { innerColor.setFill() }

@objc override func setStroke() { innerColor.setStroke() }

@objc override func getWhite(_ white: UnsafeMutablePointer<CGFloat>?, alpha: UnsafeMutablePointer<CGFloat>?) -> Bool {
    return innerColor.getWhite(white, alpha: alpha)
}

@objc override func getHue(_ hue: UnsafeMutablePointer<CGFloat>?, saturation: UnsafeMutablePointer<CGFloat>?, brightness: UnsafeMutablePointer<CGFloat>?, alpha: UnsafeMutablePointer<CGFloat>?) -> Bool {
    return innerColor.getHue(hue, saturation: saturation, brightness: brightness, alpha: alpha)
}

@objc override func getRed(_ red: UnsafeMutablePointer<CGFloat>?, green: UnsafeMutablePointer<CGFloat>?, blue: UnsafeMutablePointer<CGFloat>?, alpha: UnsafeMutablePointer<CGFloat>?) -> Bool {
    return innerColor.getRed(red, green: green, blue: blue, alpha: alpha)
}

@objc(colorWithAlphaComponent:)
override func withAlphaComponent(_ alpha: CGFloat) -> UIColor {
    return innerColor.withAlphaComponent(alpha)
}
~~~

This solves _most_ of the problems. We can now use our previously declared `.americanRed` just how we want -- that's great!

## Caveats

We were able to create a subclass of `UIColor` that as properties for the different shades it can be. We can now have properties and methods that only accept a `PaletteColor`, and not just any old `UIColor`. We can use a `PaletteColor` in any place a `UIColor` can be used.

There's just one thing we forgot -- Since `PaletteColor` is a subclass of `UIColor`, it can be instantiated like any other `UIColor`, which gives us 2 problems:

- We can still instantiate `PaletteColor`s and get non-PaletteColor instances back
- We can still instantiate `PaletteColor`s with any old color

~~~ swift
let color = PaletteColor(red: 1, green: 0, blue: 0, alpha: 1)
type(of: color) // UIDeviceRGBColor

// 💥: CRASH, because it's not actually a PaletteColor
color.light
~~~

The way to fix this is to implement all the initializers that `UIColor` has. I would propose implementing those, and probably throwing an `assertionFailure()` to prevent "unauthorized" `PaletteColor`s from being instantiated.

To implement those initializers, we would run into the same problem we had earlier, where it tells us that "Declarations from extensions cannot be overridden yet."

This means our only options are to either just be diligent in our code writing and code review, or to implement our `PaletteColor` in Objective-C, where it's possible.

## The End

Once I realized I was dealing with a class cluster, everything became clear.

After going down this road, I think it might actually be better to go down a more explicit route of having `PaletteColor` be its own class or protocol that exposes a `color` property. Being explicit is almost always better than implicit, anyway.

This started as an exploration into why my app was crashing when trying to set a property on a class I was creating. I've known about class clusters for a long time, but I never tried adding a class to one, so it didn't cross my mind that that's why it would be crashing.

## Further Reading

- [Apple Docs on Class Clusters](https://developer.apple.com/library/content/documentation/General/Conceptual/CocoaEncyclopedia/ClassClusters/ClassClusters.html) - The Good Word
- [Cocoa with Love](https://cocoawithlove.com/2008/12/ordereddictionary-subclassing-cocoa.html) - Covered this topic _in 2008_
- [This Answer on Stack Overflow](https://stackoverflow.com/a/2459385) - Easy example and discussion on class clusters


