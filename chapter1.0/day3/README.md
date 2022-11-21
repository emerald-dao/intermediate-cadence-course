# Chapter 1 Day 3 - Poor Capability Links

This is a fun chapter for me because I get to flame all of you. Yes, you. Poor capability links are probably my #1 pet peeve that most Cadence developers do, and is also a very common mistake. Even the Flow team does it! Ugh.

## What Are You Talking About?

Poor Capability links are when Cadence developers set up users' account storage wrong (most often NFT Collections) such that the publicly linked type is not correct. I will give you an example.

Let's say you have a standard NFT Contract called `ExampleNFT` in which the `Collection` resource implements the following interfaces: `NonFungibleToken.Receiver`, `NonFungibleToken.Provider`, `NonFungibleToken.CollectionPublic`, `MetadataViews.ResolverCollection`, and a custom interface named `ExampleNFT.CollectionPublic`. Of course, `NonFungibleToken.Provider` should not be linked to the public because it allows you to withdraw from the collection. However, the other 4 should certainly be linked. 

Here is how the public link *should* be made:

```cadence
import ExampleNFT from 0x01
import NonFungibleToken from 0x02
import MetadataViews from 0x03

transaction() {
  prepare(signer: AuthAccount) {
    signer.save(<- ExampleNFT.createEmptyCollection(), to: /storage/Collection)
    signer.link<&ExampleNFT.Collection{NonFungibleToken.Receiver, NonFungibleToken.CollectionPublic, MetadataViews.ResolverCollection, ExampleNFT.CollectionPublic}>(/public/Collection, target: /storage/Collection)
  }
}
```

Seems easy, right? Apparently, not for everyone...

## Common Mistakes

There are two very common mistakes that mess up users' collections:
1. Forgetting an Interface
2. Using `&AnyResource`

### Forgetting an Interface

It is very common that developers only link the NFT Contract's custom interface instead of also linking the standard `NonFungibleToken` interfaces, like so:

```cadence
import ExampleNFT from 0x01

transaction() {
  prepare(signer: AuthAccount) {
    signer.save(<- ExampleNFT.createEmptyCollection(), to: /storage/Collection)
    signer.link<&ExampleNFT.Collection{ExampleNFT.CollectionPublic}>(/public/Collection, target: /storage/Collection)
  }
}
```

This is bad. Although it is may be true that the `ExampleNFT.CollectionPublic` interface may include all the same functions that the `NonFungibleToken` interfaces include, **that does not mean you should not link the `NonFungibleToken` interfaces.** 

If you took the <a href="https://github.com/emerald-dao/beginner-cadence-course" target="_blank">Beginner Cadence Course</a>, you would know that I emphasize conforming to standards. It is very important to allow DApps to read your users' collections without putting the berdon on them to figure out *how* to read the collection.

Everyone knows that `NonFungibleToken.CollectionPublic` will allow you to call the `getIDs` function inside the `Collection` resource. However, not everyone knows that your custom `ExampleNFT.CollectionPublic` interface even exists. So when someone like me goes to read your collections by doing this:

```cadence
import ExampleNFT from 0x01

pub fun main(user: Address): [UInt64] {
  let collection = getAccount(user).getCapability(/public/Collection)
              .borrow<&ExampleNFT.Collection{NonFungibleToken.CollectionPublic}>()
              ?? panic("This link does not exist.")

  return collection.getIDs()
}
```

... it would fail if you never linked `NonFungibleToken.CollectionPublic`, and instead I would have to look into your contract code to know to use `ExampleNFT.CollectionPublic` instead.

Besides pure annoyance, there are actually important reasons why standard interfaces must be linked. Marketplace contracts like <a href="https://github.com/onflow/nft-storefront/blob/78e6df21959cc872569a54c6a232b603c622955e/contracts/NFTStorefrontV2.cdc#L263">NFTStorefront</a> rely on `NonFungibleToken.Receiver` and `NonFungibleToken.Provider` to be linked. If they are not, the user won't be able to interact with the contract properly.

There are also many other examples of this. One of them being Emerald City's <a href="https://bot.ecdao.org/">Emerald bot</a>, which attempts to identify user NFT holdings to give users roles in Discord. Because the Emerald bot cannot know about every single NFT Contract in existance, it must rely on the `NonFungibleToken` standard to browse user's collections. If a user's standard interfaces are not linked, the bot cannot read your collection and know you have certain NFTs!

Long story short: **Always link all the proper interfaces to the public, even if you don't think you need to.**

### Using `&AnyResource`

This mistake is also very common. Let's take a look at another bad way to set up a user's collection:

```cadence
import ExampleNFT from 0x01
import NonFungibleToken from 0x02
import MetadataViews from 0x03

transaction() {
  prepare(signer: AuthAccount) {
    signer.save(<- ExampleNFT.createEmptyCollection(), to: /storage/Collection)
    signer.link<&{NonFungibleToken.Receiver, NonFungibleToken.CollectionPublic, MetadataViews.ResolverCollection, ExampleNFT.CollectionPublic}>(/public/Collection, target: /storage/Collection)
  }
}
```

Well Jacob... what's wrong with this? All the interfaces are linked! 

Yes, they are. But notice that `ExampleNFT.Collection` is not specified, rather we use an implicit `&AnyResource` type. When you write something like this:

```cadence
&{NonFungibleToken.Receiver}
```

That is actually the same thing as:

```cadence
&AnyResource{NonFungibleToken.Receiver}
```

The issue here is that you do not link the base resource type to the public. While this may still work correctly sometimes, you will run into trouble later down the road when you want to borrow this collection. For example:

```cadence
import ExampleNFT from 0x01
import NonFungibleToken from 0x02

pub fun main(user: Address): [UInt64] {
  let collection = getAccount(user).getCapability(/public/Collection)
              .borrow<&ExampleNFT.Collection{NonFungibleToken.CollectionPublic}>()
              ?? panic("This link does not exist.")

  return collection.getIDs()
}
```

This script will fail. Why? Because you did not link the `ExampleNFT.Collection` to the public. You would have to borrow it like this:

```cadence
import ExampleNFT from 0x01
import NonFungibleToken from 0x02

pub fun main(user: Address): [UInt64] {
  let collection = getAccount(user).getCapability(/public/Collection)
              .borrow<&{NonFungibleToken.CollectionPublic}>()
              ?? panic("This link does not exist.")

  return collection.getIDs()
}
```

... which is not only really improper, but much more scary, will not guarantee that the collection at that public path is even an `ExampleNFT.Collection`!

In this case, the user could technically link a completely different collection (that also implements `NonFungibleToken.CollectionPublic`) to that public path on their own, potentially an NFT Collection that they created that holds no value. So when you try to find all the `id`s of the NFTs in their `ExampleNFT.Collection` by running the above script, you may get `id`s that are from an entirely different collection. Scary stuff!

## A MainNet Example

Let's look at a real-life example of setting up a user's NFT Collection really bad.

Click <a href="https://github.com/dapperlabs/nba-smart-contracts/blob/master/transactions/user/setup_account.cdc#L23">here</a> to see the official NBATopShot transaction to set up a user's account. Notice anything fishy?

Yep! They use `&AnyResource` *AND* don't link `NonFungibleToken.Receiver`!. Let's see why this is bad.

Head over to https://runflow.pratikpatel.io/ and make sure it is toggled to MainNet. Paste in the below transaction:

```cadence
import TopShot from 0x0b2a3299cc857e29
import NonFungibleToken from 0x1d7e57aa55817448

pub fun main(user: Address): [UInt64] {
  let collection = getAccount(user).getCapability(/public/MomentCollection)
              .borrow<&TopShot.Collection{NonFungibleToken.CollectionPublic}>()
              ?? panic("Your TopShot Collection is not set up correctly.")

  return collection.getIDs()
}
```

In the box that says "User", paste in your Dapper Wallet address. If you don't have one, you can put in mine: `0x84efe65bd9993ff8`. Execute the script.

You'll notice you get an error:

<img src="https://i.imgur.com/kqQB66i.png" alt="error in script" />

This is because `TopShot.Collection` isn't linked! Try the following instead:

```cadence
import TopShot from 0x0b2a3299cc857e29
import NonFungibleToken from 0x1d7e57aa55817448

pub fun main(user: Address): [UInt64] {
  let collection = getAccount(user).getCapability(/public/MomentCollection)
              .borrow<&{NonFungibleToken.CollectionPublic}>()
              ?? panic("Your TopShot Collection is not set up correctly.")

  return collection.getIDs()
}
```

For me, even this didn't work! Wow, what a mess. In order for me to read my NFT ids, I had to run this script:

```cadence
import TopShot from 0x0b2a3299cc857e29

pub fun main(user: Address): [UInt64] {
  let collection = getAccount(user).getCapability(/public/MomentCollection)
              .borrow<&{TopShot.MomentCollectionPublic}>()
              ?? panic("Your TopShot Collection is not set up correctly.")

  return collection.getIDs()
}
```

Although we were eventually able to read out ids, hopefully this has shown you how inconvenient poor links are. It requires the consumer (us) to weed our way through interfaces to figure out how to actually get to the data, which should have been very straightforward if done correctly.

And even worse, with the above script, **I have no guarantee that the collection I'm reading from is even a `TopShot.Collection`.** For all I know, it could be some other random NFT Collection that also implements the `TopShot.MomentCollectionPublic` interface and is linked to the same public path. This could be a very severe bug in our code. We will talk more about this in the next day.

## Quests

1. Explain the two most common mistakes Cadence developers make regarding poor capability links.

2. Rewrite <a href="https://github.com/dapperlabs/nba-smart-contracts/blob/master/transactions/user/setup_account.cdc">this script</a> from the NBATopShot official repo to link users' collections properly.

3. Let's say this script fails when trying to read a user's NFTs:

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

...but this script works:

```cadence
import ExampleNFT from 0x01

pub fun main(user: Address): [UInt64] {
  let collection = getAccount(user).getCapability(/public/Collection)
              .borrow<&ExampleNFT.Collection{ExampleNFT.CollectionPublic}>()
              ?? panic("Your TopShot Collection is not set up correctly.")

  return collection.getIDs()
}
```

What do you think the transaction looked like to set up this user's collection?

4. Let's say this script fails when trying to read a user's NFTs:

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

...but this script works:

```cadence
import NonFungibleToken from 0x02

pub fun main(user: Address): [UInt64] {
  let collection = getAccount(user).getCapability(/public/Collection)
              .borrow<&{NonFungibleToken.CollectionPublic}>()
              ?? panic("Your TopShot Collection is not set up correctly.")

  return collection.getIDs()
}
```

What do you think the transaction looked like to set up this user's collection?