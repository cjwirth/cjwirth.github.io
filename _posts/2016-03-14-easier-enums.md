---
layout: post
title: Easier Enums with Private Types
---

In Swift, I try to use protocols as often as I can. I also like to use enums as often as is appropriate. Often times I find myself with an enum conforming to some protocol or another. For every property I implement, I have to `switch self` to get the right value -- and this can get _really_ old, _really_ fast. 

Luckily, Swift lets you have private types that you can use to make your enums a whole lot shorter! This is just a quick tip that I came across the other day. You won't believe what comes next!

<!--excerpt-->

## My Enum is Too Big!

To set the stage, let's imagine we are making a Pokédex app, so we'll need to create a protocol for the Pokémon we will have in our app.

~~~swift
protocol Pokemon {
    var name: String { get }
    var element: Element { get } // `Element` defined elsewhere. You know: Fire, Water, Electric, etc...
    var info: String { get }
}
~~~

Now we can display any type that conforms to the Pokémon protocol in our Pokédex view.

~~~swift
class PokedexView: UIView {
   ...
   func showPokemon(pokemon: Pokemon) {
       nameLabel.text = pokemon.name
       infoLabel.text = pokemon.info
       backgroundColor = pokemon.element.color
   }
}
~~~

Excellent, we are off to a good start. Now to actually define the Pokémon to use in our app. 

Since each generation of Pokémon game has a different set of Pokémon available, and because it vaguely fits with the example I am putting together, we will make a different enum for each generation of Pokémon game.

~~~ swift
enum RedBluePokemon: Pokemon {
    case Bulbasaur
    case Charmander
    case Squirtle
    ...
}
~~~

But wait! To conform to the `Pokemon` protocol, we need to define the 3 properties necessary. Doing so, we end up with something like this:

~~~swift
enum RedBluePokemon: Pokemon {
    ...
    var name: String {
        switch self {
        case .Bulbasaur: return "Bulbasaur"
        case .Charmander: return "Charmander"
        case .Squirtle: return "Squirtle"
        ...
        }
    }
    
    var element: Element {
        switch self {
        case .Bulbasaur: return .Grass
        case .Charmander: return .Fire
        case .Squirtle: return .Water
        ...
        }
    }
    
    var info: String {
        switch self {
        case .Bulbasaur: return "A strange seed was planted on its back at birth. The plant sprouts and grows with this Pokémon."
        case .Charmander: return "Obviously prefers hot places. When it rains, steam is said to spout from the tip of its tail."
        case .Squirtle: return "After birth, its back swells and hardens into a shell. Powerfully sprays foam from its mouth."
        ...
        }
    }
}
~~~

As we can tell, this will get pretty repetitive. Since enums don't have stored properties, in each property we have to `switch self` and go through all the options to pass back the correct value! 

This isn't so bad with only 3 properties and 3 possibilities it's not so bad. But what about when we add all 151 Pokémon from Red and Blue! What a monstrosity this will be!

We'll end up with a minimum of `(number of properties) * (options in enum)` lines of code! And it'll keep on getting bigger if we add another property! Yuck! 

And this is just for the `RedBluePokemon`, we still have to implement `YellowPokemon`, `GoldPokemon`, `SilverPokemon`...

## Instantiatable Values

What's the easiest way to have a type that conforms to a protocol with properties? By making a struct that has each of the properties. 

~~~swift
private struct PokemonValues: Pokemon {
   var name: String
   var element: Element   
   var info: String
}
~~~

Now we can instantiate a new Pokémon by passing in the values that we need.

~~~swift
let pikachu = PokemonValues(name: "Pikachu", element: .Electric, info: "When several of these Pokémon gather, their electricity could build and cause lightning storms.")
~~~

But did you notice that we made it a `private` struct? The reason for that is that we don't want to be able to instantiate bogus Pokémon. We already have a list of `RedBluePokemon` that we want to be able to use, and we want to make sure we don't end up with phony data like this:

~~~swift
let caesar = PokemonValues(name: "Caesar", element: .Fire, info: "The very best, like no one ever was")
~~~

By making it a `private struct` we can ensure that we only create `PokemonValue` instances within this constrained file. 

## Value Aggregation!

Now we're going to bring the two together so we can cut down the size of our enum. 

What we're going to do is have a private property on the enum that returns a `PokemonValues` for each of the possible Pokémon. Then, when we need to get the value in the protocol method, we will just reference the private `PokemonValues` property.

~~~swift
enum RedBluePokemon: Pokemon {

    var name: String { return values.name }
    var info: String { return values.info }
    var element: Element { return values.element }

    private var values: PokemonValues {
        switch self {
        case .Bulbasaur: return PokemonValues(name: "Bulbasaur", element: .Grass, info: "A strange seed was planted on its back at birth. The plant sprouts and grows with this Pokémon.")
        case .Charmander: return PokemonValues(name: "Charmander", element: .Fire, info: "Obviously prefers hot places. When it rains, steam is said to spout from the tip of its tail.")
        case .Squirtle: return PokemonValues(name: "Squirtle", element: .Water, info: "After birth, its back swells and hardens into a shell. Powerfully sprays foam from its mouth.")
        ...
        }
    }
}
~~~

Much better! Not only were we able to _drastically_ cut down the line count, but now we also have all the values for each Pokémon together. We don't have to scroll the page to see that Charmander is a Fire-type and then again to see what his Pokédex entry says.

## In Parting

It's not perfect -- you have to instantiate an entire new `PokemonValues` struct every time you need a value; since you're using protocols, you can't completely prevent someone from creating their own `PokemonValues` struct to create bogus values; there is probably a way better way to store Pokémon data than in an enum like this! (How many are there by now anyway?)

This is just a neat way I found to make my enums shorter and easier to read. I'm sure there's different approaches, and I'm always looking forward to hearing more.

And yes, there is probably a better way to store Pokémon than in the way I presented above, but I hope it worked for an example 😜