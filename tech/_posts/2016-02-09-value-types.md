---
layout: post
title: Value Types in Swift
---

Swift has reference types and value types. Even within the value types, there are different options to use! It can be kind of confusing to know when to use which. Knowing a little bit about what types work in which situations can really help out.

Here are some of the ways I differentiate the different value types in my own head, and some rules of thumb of when I use which.

<!--excerpt-->

## Tuples

Tuples are the simplest value types in swift. They are just a group of values bundled together. They aren't extensible in any way -- no extensions, no methods, no protocol conformance. 

~~~ swift
// We make protocol for people
protocol PersonType {
    var name: String { get }
    var age: Int { get }
}

// Give a name to our tuple type
typealias User = (name: String, age: Int)

// error: non-nominal type 'User' (aka '(user: String, age: Int)') cannot be extended
// extension User: PersonType { }

// Even though we can access it the same as a PersonType,
// We can't make a User be a PersonType
let caesar: User = (name: "Caesar", age: 26)
print(caesar.name) // "Caesar"
print(caesar.age) // 26
~~~

So it looks like tuples are pretty limited. There's still valid uses for them!
I'm sure there are more uses than just the ones I'm listing, but these are some places I've seen tuples used that made sense.

### Multiple Return Values 

One of the places tuples seem like a natural fit is when you want to return multiple values from a function. In languages that don't have tuples or multiple return values, the only type safe way to do so would be to create an object that only exists to hold the values you want to return.

Luckily in Swift, we don't have to do that. We can just create a new tuple that contains only the values we want.

~~~ swift
func findMinAndMax(values: [Int]) -> (min: Int, max: Int) {
    var minimum = 0
    var maximum = 0
    values.forEach { number in
        minimum = min(minimum, number)
        maximum = max(maximum, number)
    }
    return (min: minimum, max: maximum)
}
~~~

This has definitely been the most common use case for tuples in my code. It doesn't come up all the time, but when it does, it's a really nice feature to have.

### Stored Function Parameters

There's one cool feature about tuples that I think often gets overlooked. You can store a tuple somewhere and use it as all the parameters to a function.

~~~ swift
func printUserInfo(name: String, age: Int) {
    print("Name: \(name), Age: \(age)")
}

let caesar = (name: "Caesar", age: 26)
printUserInfo(caesar)
~~~

Sure, this example is a little contrived, but what does it tell us? Functions actually take tuples. This gives opens up some interesting options. 

Using this mixed with closures, we can chain functions together.

~~~ swift
func fetchUser() -> (name: String, age: Int) { ... }
func printUser(name: String, age: Int) { ... }

printUser(fetchUser())
~~~

Again, this example seems pretty trivial, but it can be useful when using some functional reactive library like [RxSwift](https://github.com/ReactiveX/RxSwift.git).

Some might argue that this isn't actually using tuples, and they might be right. But I think it's super neat that you can just pass in the tuple without having to break it up like so:

~~~ swift
printUser(user.name, age: user.age)
~~~

## Enums

Enums are probably my favorite value type in Swift. They are much more powerful than their Objective-C counterparts.

At their most basic, enums are just a type-safe way of specifying a set of mutually exclusive possible values in a certain category. With associated values, you can make them a little bit more dynamic than just static values.

I've found that in practice, I find myself enums in two ever-so-slightly different ways. 

One way is the traditional "different choices of the same thing," e.g. an enum of video quality settings with the choices High, Medium, and Low.

The other way is being completely different things, but in the same category. For example, analytics events that all have different meanings and associated values, but they are still all analytics events.

### Enums representing Choice

Enums whose possible values represent different values of the same thing is the most natural way to use enums. 

~~~ swift
enum VideoQuality {
    case High
    case Medium
    case Low

    var bitrate: Double {
        switch self {
        case .High: return 4200000
        case .Medium: return 1400000
        case .Low: return 300000
        }
    }
}
~~~

This is a very simple case where you can use enums to represent a choice. Instead of having a function or property that takes a `Double`, you can make it take a `VideoQuality`, and ensure that you only supported values get passed in.

Enums like this can also help increase the readability of your code. Ash Furrow recently posted [a blog post](https://ashfurrow.com/blog/the-wrong-binary/) about using enums in place of boolean values to more explicitly explain what parts of your code represent.

~~~ swift
enum VideoState {
    case Playing
    case Stopped
}

if case .Playing = video.state {
    print("Video is now playing")
}
~~~

Now it's easy to see that we mean that the video is playing. 

We could have also done this with an `isPlaying` field, and it would have worked fine. With an enum, we also leave it open to add new states in the future, like `Paused` or `Seeking`.

### Enums representing Different Things

When I way "different things," I still mean that they are in the same category of things. What I mean is that if we add associated values to our enums, each of our options can represent an entirely different thing. 

If we are watching a video, and we periodically get metadata that we need to monitor, we can use an enum to represent it, even though the individual values will change each time.

~~~ swift
enum VideoMetadata {
    case Program(channelId: String, programId: String, position: Int)
    case Ad(channelId: String, adId: String)
    case Filler(channelId: String)
}
~~~

This is the difference between enums as Choices vs Different Things. Each choice of `VideoState` is essentially equivalent, and represents the same thing. Each choice of `VideoMetadata` represents something completely different, although it is in the same category of metadata received during the video.

That's what makes enums so powerful! You can encapsulate different data within each state. Then you can pass that enum into a function, and you know you will only have valid values in there!

Or maybe you need to pass your enum into a function that takes a protocol type. No problem! Enums can have properties and conform to protocols just like any other type (sorry, tuples). 

One way I've found this useful is having a protocol for making API calls, and then having an enum that represents each endpoint of my API server. Then I can pass in the required parameters, and send it off to the server.

~~~ swift
// Enum of Choices to define the basic HTTP verbs
enum HTTPMethod { 
    case GET, POST, PUT, DELETE // Our API doesn't require the others
}

// Protocol that represents an API request
protocol APIRequestable {
    var method: HTTPMethod { get }
    var path: String { get }
    func createParameters() -> [String: Any]
}

// All the API Endpoints to our server are taken care of in here
// This makes it so I don't have to stringly-type my requests everywhere
enum APIEndpoint: APIRequestable {
    case GetUser(userId: String)
    case RegiterUser(username: String, password: String)
    
    var method: HTTPMethod {
        switch self {
        case .GetUser: return .GET
        case .RegisterUser: return .POST 
        }
    }

    var path: String {
        switch self {
        case .GetUser(userId: let id):
            return "/users/\(id)"
        case .RegisterUser:
            return "/users"
        }
    }

    func createParameters() -> [String, Any] {
        switch self {
        case .GetUser:
            return [:]
        case .RegisterUser(username: let name, password: let pass):
            return ["username": name, "password": pass]
        }
    }
}
~~~

One thing that can get a little annoying is having to have a `switch` statement in each parameter. This can make your code longer than it feels like it needs to be, and you can't find all the information about one endpoint in one glance.

## Structs

Structs are probably the easiest to understand. They are just like objects in other languages. Sure, they are values and not references, but that really doesn't change how they work.

It's kind of hard to come up with a rule of thumb for when structs are the appropriate value type -- the answer basically comes down to "when tuples are too simple, and enums are too specific." 

Most of the value types you write will be structs. Even types like `String` and `Int` are actually structs! 

What is a more difficult question is when you should use Structs vs Classes. It really all comes down to what your use case is. There's a lot of media online discussing the difference between value types and reference types, so I'm just going to briefly touch on the rules of thumb I keep in mind when creating a new type.

### Structs ARE Things

Values are copied on assignment, so they can't be used for shared state. Ideally they are just inert bits of data that represent something meaningful. 

~~~ swift
struct User {
    var name: String
    var age: Int
}
~~~

This is just a type that represents a user. There is no problem with passing it back and forth. It's just like passing around a `Int`.

This isn't to say that structs can't have methods or always be immutable. Even if your struct's properties are declared as `var`, you can still make the entire struct immutable by using `let` when instantiating it.

~~~ swift
struct User {
    var id: String
    var name: String
    var age: Int

    mutating func haveBirthday() {
        age += 1
    }
}

let caesar = User(name: "Caesar", age: 26)
// caesar.haveBirthday() // We can make Users immutable when we want them
var steve = User(name: "Steve", age: 25)
steve.haveBirthday() // Or we can make them mutable when we want
~~~

Even though this new `User` has a mutable function, it still only manipulates it's own private state, which is OK. It still just represents a user and doesn't change anything outside of its bounds.

### Classes DO Things

Sometimes you need objects that keep track of and manipulate other objects. You want to be able to share instances between different places. These objects don't really represent things, rather they manage things. 

A lot of times you will see the names of these types end in `Manager`, `Controller`, `Handler`, and many other "-er" prefixes. Not always, but it's a common trope in programming. 

As you can see, these classes are "do-ers" as opposed to the "be-ers" we had with structs.

One of the easiest to understand examples is a class that is your data layer. Maybe you have a class where you can fetch and save users. 

~~~ swift
class UserManager {
    private var users: [String: User] = [:]

    func fetchUser(userId: String) -> User? {
        return users[userId]
    }

    func saveUser(user: User) {
        users[user.id] = user
    }
}
~~~

If this was a value type instead, then everywhere we passed a `UserManager`, we would get a copy. That could cause big problems, because you if someone calls `saveUser`, none of the copies would have the change, and they would be left with stale data.

## In Closing

Swift gives us a lot of options when deciding on which types we want to use. Ultimately, the choice is up to you, but it can be nice to have some rules of thumb for what types work well in what situation.

I'm sure there are things I've forgotten here. Maybe you have a different way that you decide which type to use? Let me know in the comments, or send me a message on [Twitter](https://www.twitter.com/cjwirth)!

