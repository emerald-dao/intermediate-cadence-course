# Chapter 1 Day 4 - Working with Poor Links

Yeah yeah we're back again for more content. If your brain hasn't turned to mush already, congrats! If it has, keep up! We're gonna keep moving along.

## Preventing Bugs from Poor Links

In the last day, we talked a lot about why poor links may be inconveniant. But there's actually much worse problems that can arise from poor links, potentially critical bugs. I will show you an example below.

Let's say you write a contract that defines a new type of NFT: `ExampleNFT.NFT`. And after, you want to write a function that checks if a user has more than 3 of your `Example.NFT`s, and if so, gives them $FLOW token. Let's see what that function would look like:

```cadence
pub fun giveFlowIfUserHas3NFTs(userCollection: &{ExampleNFT.CollectionPublic}) {
  if userCollection.getIDs() > 3 {
    // deposit $FLOW to the user here
  }
}
```

Although I left out the actual implementation of this function, it is pretty simple. It takes in a reference to a user's NFT Collection, and if it has more than 3 NFTs inside of it, it will give you $FLOW. 

The trick here is looking at the type of the NFT collection. What does `&{ExampleNFT.CollectionPublic}` mean? Well, an idio- ... I mean most people would say, "Oh, this is a reference to a user's ExampleNFT collection! *buzzer noise*. Wrong! Actually, this is a reference to ANY resource that implements the `ExampleNFT.CollectionPublic` interface.

The critical bug here is that I could deploy my own contract that has a resource that implements `ExampleNFT.CollectionPublic`, mint fake NFTs to that account, and then pass in the reference to the function above and still claim the $FLOW token. Uh oh!

### So What's the Fix?

Well, the fix is actually very simple. You could rewrite the function to be:

```cadence
pub fun giveFlowIfUserHas3NFTs(userCollection: &ExampleNFT.Collection{ExampleNFT.CollectionPublic}) {
  if userCollection.getIDs() > 3 {
    // deposit $FLOW to the user here
  }
}
```

Now, the user **must** pass in the correct Collection to the function.

However, there's an alternative way to go about this.

## Resource Identifier

There's this awesome thing in Cadence called an `identifier`. Every single type in Cadence has an identifier, including resource types. This means we can check the underlying type of a resource (or its reference) without requiring you to completely specify its type up-front.

Let's take a look at the script below:

```cadence
import Flovatar from 0x921ea449dffec68a

pub fun main() {
  let newCollection: @Flovatar.Collection <- Flovatar.createEmptyCollection()

  let collectionType: Type = newCollection.getType()
  log(collectionType) // Type<@Flovatar.Collection>()

  let collectionIdentifier: String = collectionType.identifier
  log(collectionIdentifier) // A.921ea449dffec68a.Flovatar.Collection

  destroy newCollection
}
```

You can sort of get a sense for how to retrieve the `identifier` from a resource in the above script. 

An `identifier` is basically a string description of the type of something in Cadence. More specifically, the format is:

<img src="https://i.imgur.com/lUlrXTw.png" alt="identifier description" />

### Back to Our Example

So, if we wanted to make sure only the correct reference was being passed in, and for some other reason wanted to do it the complicated way, we could use resource identifier's instead. Let's assume our ExampleNFT contract was deployed to an account with address `0x2d4c3caffbeab845`.

```cadence
pub fun giveFlowIfUserHas3NFTs(userCollection: &{ExampleNFT.CollectionPublic}) {
  pre {
    userCollection.getType().identifier == "A.2d4c3caffbeab845.ExampleNFT.Collection":
      "You are passing in the incorrect type!"
  }
  if userCollection.getIDs() > 3 {
    // deposit $FLOW to the user here
  }
}
```

You may be wondering... when is this useful? Why would we ever want to use identifiers in our contract? Well, a perfect example is when we're trying to deal with user collections that are set up really poorly.

Let's say a user's `ExampleNFT.Collection` was linked like so:

```cadence
import ExampleNFT from 0x01
transaction() {
  prepare(signer: AuthAccount) {
    signer.save(<- ExampleNFT.createEmptyCollection(), to: /storage/Collection)
    signer.link<&{NonFungibleToken.Receiver, NonFungibleToken.CollectionPublic, MetadataViews.ResolverCollection, ExampleNFT.CollectionPublic}>(/public/Collection, target: /storage/Collection)
  }
}
```

Let's also say that we want to verify that a user holds 3 `Example.NFT`s. Well, we can't do it like this:

```cadence
import ExampleNFT from 0x01
import NonFungibleToken from 0x02

pub fun main(user: Address): [UInt64] {
  let collection = getAccount(user).getCapability(/public/Collection)
              .borrow<&ExampleNFT.Collection{NonFungibleToken.CollectionPublic}>()
              ?? panic("Your TopShot Collection is not set up correctly.")

  return collection.getIDs()
}
```

... because as we learned in the last day, this isn't possible given how we linked the path (using `&AnyResource`). So, we'd have to check it like this:

```cadence
import ExampleNFT from 0x01
import NonFungibleToken from 0x02

pub fun main(user: Address): [UInt64] {
  let collection = getAccount(user).getCapability(/public/Collection)
              .borrow<&{NonFungibleToken.CollectionPublic}>()
              ?? panic("Your TopShot Collection is not set up correctly.")

  return collection.getIDs()
}
```

However, as we also learned in the last day, this provides no certainty that the ids we're returning are actually from an `ExampleNFT.Collection`. It could be any resource that implements `NonFungibleToken.CollectionPublic`.

The way to verify this is with identifiers. We can say...

```cadence
import NonFungibleToken from 0x02

pub fun main(user: Address): [UInt64] {
  let collection = getAccount(user).getCapability(/public/Collection)
              .borrow<&{NonFungibleToken.CollectionPublic}>()
              ?? panic("Your TopShot Collection is not set up correctly.")

  assert(
    collection.getType().identifier == "A.01.ExampleNFT.Collection",
    message: "This is not the correct type. No hacking me today!"
  )

  return collection.getIDs()
}
```

Now, we are protected :)

Notice, by the way, that we didn't even have to import `ExampleNFT`, but we were still able to verify the type of the reference. Pretty interesting... I will leave it to all of you to figure out some cool tricks you can do with this in your own contracts. We may discover more throughout this course.

## Quests

1. Explain what a resource identifier is.

2. Is it possible for two different resources, that have the same type, to have the same identifier?

3. Is it possible for two different resources, that have a different type, to have the same identifier?

4. Based on the included comments:
- What is wrong with the following script? 
- Explain in detail how someone hack this script to always return `true`. 
- Then, what are two ways we could fix this script to make sure it is safe?

```cadence
import NonFungibleToken from 0x02

// Return true if the user should be given a prize for holding
// more than 5 TopShot NFTs
pub fun main(user: Address): Bool {
  let collection = getAccount(user).getCapability(/public/Collection)
              .borrow<&{NonFungibleToken.CollectionPublic}>()
              ?? panic("Your TopShot Vault is not set up correctly.")

  if collection.getIDs().length > 5 {
    return true
  }

  return false
}
```

5. Let's say we have the following script on Mainnet...

```cadence
import FungibleToken from 0xf233dcee88fe0abe

pub fun main(user: Address): UFix64 {
  let vault = getAccount(user).getCapability(/public/Vault)
              .borrow<&{FungibleToken.Receiver}>()
              ?? panic("Your Vault is not set up correctly.")

  return vault.balance
}
```

This script was originally written to read the balance of a user's Flow Token Vault, but for some reason we were not able to borrow the full type, so we had to use `&AnyResource` like we did above.

Rewrite this script such that we verify we are reading a balance from a FlowToken Vault. Hint: Use this page to find the Mainnet addresses of the contracts you will need: https://developers.flow.com/flow/core-contracts