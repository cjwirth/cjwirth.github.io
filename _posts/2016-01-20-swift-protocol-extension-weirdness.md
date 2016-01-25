---
layout: post
title: Swift Protocol Extension Weirdness
---

Swift is a [protocol oriented language](https://developer.apple.com/videos/play/wwdc2015-408/), and I've found I can really provide a lot of power as well as flexibility by making good use of protocols. 

The other day I was chasing down a confusing bug. For some reason my protocol methods _weren't being called!_

It turns out protocol extension methods can be unintuitive sometimes.


<!--excerpt-->

### Protocols Acting According to Protocol

What makes protocols great is that you separate declaration from definition -- you can have different types conform to the same protocol, but they might implement the functions in their own way.

Swift 2 gave us _protocol extensions_, and we can add some default behavior to our protocols.

```Swift
protocol Ferocious {
    func roar()
}

extension Ferocious {
    func roar() {
        print("ROOOAAARRR!!!")
    }
}

struct Dinosaur: Ferocious {
    // We are fine with the default roar
}

struct Dog: Ferocious {
    // We don't want the default, so we define our own
    func roar() {
        print("woof!")
    }
}

let dinosaur: Dinosaur = Dinosaur()
dinosaur.roar() // "ROOOAAARRR!!!"

let dog: Dog = Dog()
dog.roar() // "woof!"

let ferociousDog: Ferocious = Dog()
ferociousDog.roar() // "woof!"
```

In this case, it acts exactly as we expect. A _Dinosaur_ will be called with the default implementation. A _Dog_ will call _Dog_'s implementation of it, whether or not it is being treated as a _Dog_ or a _Ferocious_. It is a _Dog_, after all.

### Protocols Breaking Protocol

But then what if we wanted to add some more functionality to _Ferocious_? What would happen if we added a `bite()` method to our protocol extension?

```Swift
protocol Ferocious {
	func roar()
}

extension Ferocious {
	func roar() {
		print("ROOOAAARRR!!!")
	}
	
	func bite() {
	    print("BITE!!!")
	}
}
```

But we don't want our _Dog_ to bite in capital letters! Our app's only requires cute puppies.

![Puppy in Mug Cup](/public/images/20160120/ferocious-puppy.jpg)

<center>_Watch out, he's actually got quite a bite_</center>

So we'll just make him nom a bit:

```Swift
struct Dog: Ferocious {
    // We don't want the default, so we define our own
    func roar() {
        print("woof!")
    }
   	 
    func bite() {
        print("nom nom nom")
    }
}
```

Alright, let's check out the new functionality we added to our _Ferocious_ creatures.

```Swift
let dinosaur: Dinosaur = Dinosaur()
dinosaur.bite() // "BITE!!!"

let dog: Dog = Dog()
dog.bite() // "nom nom nom"

let ferociousDog: Ferocious = Dog()
ferociousDog.bite() // "BITE!!!"
```
Okay, it seems like something is wrong here. The _Dinosaur_ is alright, because it's just using the default implementation. But our two _Dog_s are behaving differently... That's odd...

Did you catch the difference? The `dog` is defined as a _Dog_ type, whereas `ferociousDog` is defined only as a _Ferocious_ type. 

However, if we move the `bite` function out of the _extension_ and into the _protocol_, we can see that it behaves as we originally wanted it to:

```Swift
protocol Ferocious {
	func roar()
	func bite()
}

...

let dinosaur: Dinosaur = Dinosaur()
dinosaur.bite() // "BITE!!!"

let dog: Dog = Dog()
dog.bite() // "nom nom nom"

let ferociousDog: Ferocious = Dog()
ferociousDog.bite() // "nom nom nom!!!"
```

There we go, that's more like what we expected.

### Why??

While confusing at first, there is some logic to this behavior. 

If you have the method in the protocol section, then it is clear that there is a method that the protocol provides. 

If you only have it in the protocol extension and your object's implementation, it's just an overloaded method. Whichever one is appropriate will run. 

That's why when we declared a _Dog_ instance, we called _Dog_ methods, and when we declared a _Ferocious_ instance, we called _Ferocious_ instances.

### Conclusion

Put all your protocol methods in the protocol section. If you need some helper extension functions, make them private so you don't accidentally call the wrong implementation.

That's my recommendation, but if you can think of any situation where it makes sense to use those extension methods, leave a comment! I'd love to hear it.
