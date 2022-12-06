# Chapter 4 Day 4 - Miscellaneous

Heeeeello again and welcome back to the last lesson in this course. Although you have probably gotten sick of me, I know you're going to miss me.

In this lesson, I'm going to be throwing a lot of random, smaller topics at you just for the heck of it. These things are not crazy important, but will help you write your programs more efficiently.

## Nil-Coalescing Operator

You know how when we write `panic` statements sometimes, we use the `??` symbol? Well, what *actually* is that?

The `??` allows you to perform some action if the thing to the left of it is equal to `nil`. Let's look at an example:

```cadence
var var1 = nil
var var2 = var1 ?? 1

log(var2) // 1
```

In this case, because `var1` is `nil`, it will instead be 1. Let's look at another example:

```cadence
var var1 = 2
var var2 = var1 ?? 1

log(var2) // 2
```

Here, beacuse `var1` is not `nil`, it will just equal 2. Doesn't it make sense now why we can use `??` after we borrow references from account storage? Something like this:

```cadence
pub fun main(user: Address) {
  let attemptBorrow: &ExampleNFT.Collection{NonFungibleToken.CollectionPublic? = getAccount(user).getCapability(/public/Collection)
          .borrow<&ExampleNFT.Collection{NonFungibleToken.CollectionPublic}>()
  
  let collection = attemptBorrow ?? panic("The user does not have a Collection set up.")
}
```

## Optional Binding

Optional Binding is another really useful strategy we can use with optionals. Essentially, it lets you enter an if statement depending on whether or not a value is nil, and unwraps it:

```cadence
let var1: Int? = nil
if let var2 = var1 {
  // it never makes it in here
} else {
  // it will go to here
}
```

```cadence
let var1: Int? = 2
if let var2 = var1 {
  // it will go in here, and `var2` will now have 
  // type `Int`, not `Int?`. So it unwrapped the
  // optional.
} else {
  // it never makes it in here
}
```