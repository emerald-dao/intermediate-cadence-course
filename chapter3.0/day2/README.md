# Chapter 3 Day 2 - AuthAccount in Scripts

Alright, let's get to the more complicated stuff. In today's lesson, we'll be learning about fetching an AuthAccount inside of a script.

## `AuthAccount` Review?

To review, read this section from the Beginner Cadence Course:

> On Flow, accounts can store their own data. What does this mean? Well, if I own an NFT (NonFungibleToken) on Flow, that NFT gets stored in my account. This is very different than other blockchains like Ethereum. On Ethereum, your NFT gets stored in the smart contract. On Flow, we actually allow accounts to store their own data themselves, which is super cool. But how do we access the data in their account? We can do that with the `AuthAccount` type. Every time a user (like you and me) sends a transaction, you have to pay for the transaction, and then you "sign" it. All that means is you clicked a button saying "hey, I want to approve this transaction." When you sign it, the transaction takes in your `AuthAccount` and can access the data in your account.

> You can see this being done in the prepare portion of the transaction, and that's the whole point of the prepare phase: to access the information/data in your account. On the other hand, the execute phase can't do that. But it can call functions and do stuff to change the data on the blockchain. NOTE: In reality, you never actually need the execute phase. You could technically do everything in the prepare phase, but the code is less clear that way. It's better to separate the logic.

## `AuthAccount` in a Script? What?!

Yeah I know, it's weird. Up to this point, all I told you (or all that you probably learned) was that you could only ever fetch an `AuthAccount` type inside the `prepare` statement of a transaction. But what if I told you that you could actually fetch an `AuthAccount` inside of a script?

Let's look at an example...

```cadence
pub fun main(user: Address) {
  let account: AuthAccount = getAuthAccount(user)
}
```

Pretty cool, right?

But wait... isn't this extremely dangerous? If I can access *anyones* `AuthAccount`, can't I just manipiulate everyone's account?

Umm, yeah! You can. Inside the script, if you fetch someone's `AuthAccount`, you can do whatever you want to it. You can withdraw their NFTs, tokens, manipulate their account storage, and whatever else a normal `AuthAccount` would let you do.

But here's the catch... it's a script. Scripts cannot manipulate data. Anything you do will revert after the script is over. So it's safe.

## Why Use It?

There are two main reasons to use `AuthAccount` inside of a script. I will cover each of them separately here.

### Re-linking to Discover Data

Sometimes, as we discussed in Chapter 1 Days 3-4, users' resources are not linked properly. This is usually due to developers messing up and not linking certain interfaces to the public when setting up users' accounts. For example, what if `NonFungibleToken.CollectionPublic` was not linked on a user's collection? 

```cadence
import ExampleNFT from 0x01
import NonFungibleToken from 0x02

pub fun main(user: Address): [UInt64] {
  // THIS LINE WILL PANIC because CollectionPublic is not linked.
  let collection = getAccount(user).getCapability(/public/Collection)
              .borrow<&ExampleNFT.NFT{NonFungibleToken.CollectionPublic}>()
              ?? panic("Your ExampleNFT Collection is not set up correctly.")

  return collection.getIDs()
}
```

The above script would fail because `CollectionPublic` is not linked on the user's account. To fix this, we can actually get the user's `AuthAccount`, link `CollectionPublic`, and then fetch the data:

```cadence
import ExampleNFT from 0x01
import NonFungibleToken from 0x02

pub fun main(user: Address): [UInt64] {
  // Fetch AuthAccount
  let account: AuthAccount = getAuthAccount(user)
  // First unlink the public path so we can relink it again soon
  account.unlink(/public/Collection)
  // Don't forget `Receiver`!
  account.link<&ExampleNFT.NFT{NonFungibleToken.CollectionPublic, NonFungibleToken.Receiver}>(/public/Collection, target: /storage/Collection)

  // Now this will work
  let collection = getAccount(user).getCapability(/public/Collection)
              .borrow<&ExampleNFT.NFT{NonFungibleToken.CollectionPublic}>()
              ?? panic("Your ExampleNFT Collection is not set up correctly.")

  return collection.getIDs()
}
```

Pretty cool, right?! But remember: **the user's collection will still be poorly linked after this script is over, since scripts cannot actually make any state changes.**

### Finding Data Not Displayed to the Public

Of course, since we have the `AuthAccount`, we can find any data we want inside a user's account.

Let's say there is an NFT Contract in which the `CollectionPublic` interface that the `Collection` resource implements allows you to borrow a `&NonFungibleToken.NFT` type by calling the `borrowNFT` function (the standard function that every Collection has). However, it does not allow you to borrow a full `&NFT` reference using the `borrowFullNFT` function (a custom function I just made up).

Well, we can still fetch the `&NFT` type by using `AuthAccount`:

```cadence
import ExampleNFT from 0x01

pub fun main(user: Address, id: UInt64): &ExampleNFT.NFT {
  // Fetch AuthAccount
  let account: AuthAccount = getAuthAccount(user)

  // Borrow directly from storage
  let collection = account.borrow<&ExampleNFT.NFT>(from: /storage/Collection)
              ?? panic("Your ExampleNFT Collection is not set up correctly.")

  // Not restricted by `CollectionPublic` interface, so can call this custom function.
  return collection.borrowFullNFT(id: id)!
}
```

## Quests

1. Let's say there exists a custom fungible token, $JACOB, that exists in a `JacobToken` contract. It is an official fungible token contract, so it implements the `FungibleToken` contract defined <a href="https://flow-view-source.com/mainnet/account/0xf233dcee88fe0abe/contract/FungibleToken">here</a>. We want to run a script to read the balance of someone's Vault, but it doesn't seem to be working:

```cadence
import JacobToken from 0x01
import FungibleToken from 0x02

pub fun main(user: Address): UFix64 {
  let vault = getAccount(user).getCapability(/public/JacobTokenBalance)
          .borrow<&JacobToken.Vault{FungibleToken.Balance}>()
              ?? panic("Your JacobToken Vault was not set up properly.")

  return vault.balance
}
```

Every time we run the script, it panics and says: "Your JacobToken Vault was not set up properly." Assuming the public path is correct, change this script to be able to read the balance by working around the poor link.