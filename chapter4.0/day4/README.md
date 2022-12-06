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

Optional Binding is another really useful strategy we can use with optionals. Using a new `if let` syntax, it lets you enter an if statement depending on whether or not a value is nil, and unwraps it if so:

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
  //
  // `var2` = 2
} else {
  // it never makes it in here
}
```

The above is just a simple example of what optional binding is doing. However, there are two main use cases I use optional binding for:
1. Indexing into a dictionary

```cadence
let dictionary: {String: Int} = {"One": 1, "Two": 2, "Three": 3}
if let number = dictionary["One"] { 
  // dictionary["One"] is a `Int?` type, 
  //and it gets unwrapped into `number` 
  //which has type `Int`
  log(number) // 1
}
```

2. Borrowing from someone's account storage

```cadence
let token: @ExampleNFT.NFT <- create NFT()
if let collection = getAccount(0x01).getCapability(/public/Collection).borrow<&ExampleNFT.Collection{NonFungibleToken.CollectionPublic}>() {
  // the user's collection is set up
  collection.deposit(token: token)
  return nil
}
// the collection was not set up because we didn't return in the if statement
return <- token
```

## Optional Chaining

Optional chaining is actually quite complicated to explain, but I will do my best here. Basically, optional chaining is useful for when we want to access a variable on a certain object that is an optional type, but don't want to panic if the object is `nil`.

For example:

```cadence
pub struct Test {
  pub let id: Int

  init() {
    self.id = 1
  }
}

pub fun main() {
  let test1: Test? = Test()
  let test2: Test? = nil

  let number1 = test1.id // ERROR: "Cannot access `id` on `Test?` type

  let number2: Int = test1!.id // 1
  let number3 = test2!.id // PANIC on the force-unwrap of an optional

  // Using Optional Chaining
  let number4: Int? = test1?.id // 1
  let number5: Int? = test2?.id // nil
}
```

Hopefully this script allows you to see the difference. When we have a struct or resource (in this case a struct `Test`), and want to access a field on it, we can actually use the `?` operator to *attempt to access the field* without `panic`-ing. If the struct is `nil`, it will simply return `nil`. If not, it will return the underlying value as an optional type.

## Conditional (Ternary) Operator

This one is actually quite easy, and is popular in many languages like Javascript.

Sometimes, we write code like this:

```cadence
var var1 = "one"
var var2 = 0

if (var1 == "one") {
  var2 = 1
} else if (var1 == "two") {
  var2 = 2
} else {
  var2 = 3
}
```

It's a really silly example, but is actually pretty annoying to write. Instead, we can use a "conditional operator," often written with a bunch of `?` and `:` symbols, like this:

```cadence
var var1 = "one"
var var2 = var1 == "one" ? 1 : var1 == "two" ? 2 : 3
```

These two mean the exact same thing. When you use conditional operators, it usually goes something like:

```
(boolean statement #1) ? 
  (if [boolean statement #1], this) : 
    (else, boolean statement #2) ? 
      (if [boolean statement #2], this) : 
        ...
```

> For a potentially better explanation, see <a href="https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Operators/Conditional_Operator#syntax">here</a>.

## Quests

1. Assuming the user's collection is set up properly, what will be returned from this function?

```cadence
import ExampleNFT from 0x01
import NonFungibleToken from 0x02

pub fun main(): [UInt64]? {
  let token: @ExampleNFT.NFT <- create NFT()
  if let collection = getAccount(0x01).getCapability(/public/Collection).borrow<&ExampleNFT.Collection{NonFungibleToken.CollectionPublic}>() {
    return collection.getIDs()
  }
  return nil
}
```

2. Assuming the user's collection was **not** set up properly, what will be returned from this function?

```cadence
import ExampleNFT from 0x01
import NonFungibleToken from 0x02

pub fun main(): [UInt64]? {
  let token: @ExampleNFT.NFT <- create NFT()
  if let collection = getAccount(0x01).getCapability(/public/Collection).borrow<&ExampleNFT.Collection{NonFungibleToken.CollectionPublic}>() {
    return collection.getIDs()
  } else {
    return []
  }
  return nil
}
```

3. Assuming the user's collection was **not** set up properly, what will be returned from this function?

```cadence
import ExampleNFT from 0x01
import NonFungibleToken from 0x02

pub fun main(): [UInt64]? {
  let token: @ExampleNFT.NFT <- create NFT()
  if let collection = getAccount(0x01).getCapability(/public/Collection).borrow<&ExampleNFT.Collection{NonFungibleToken.CollectionPublic}>() {
    return collection.getIDs()
  }
  return nil
}
```

4. What are the two outcomes that could happen when I run this script? Explain each.

```cadence
import ExampleNFT from 0x01
import NonFungibleToken from 0x02

pub fun main(user: Address): [UInt64] {
  let collection = getAccount(user).getCapability(/public/Collection).borrow<&ExampleNFT.Collection{NonFungibleToken.CollectionPublic}>()

  return collection!.getIDs()
}
```

5. What is wrong with the below script? 
- a) Please fix it (you are not allowed to modify this line: `return collection?.getIDs()`).
- b) After you fix it, what are the two possible outcomes that could happen when you run the script? Explain each.

```cadence
import ExampleNFT from 0x01
import NonFungibleToken from 0x02

pub fun main(user: Address): [UInt64] {
  let collection = getAccount(user).getCapability(/public/Collection).borrow<&ExampleNFT.Collection{NonFungibleToken.CollectionPublic}>()

  return collection?.getIDs()
}
```

6. Write the below code in `if-else` format instead.

```cadence
let vault = getAccount(user).getCapability(/public/Vault).borrow<&FlowToken.Vault{FungibleToken.Receiver}>()!
var status = vault.balance >= 10000 ? "Flow Legend" : vault.balance < 10 ? "Needs More" : vault.balance > 5000 ? "Flow Believer" : "Unknown"
```

7. Explain a potential benefit of using conditional chaining as opposed to force unwrapping an optional.