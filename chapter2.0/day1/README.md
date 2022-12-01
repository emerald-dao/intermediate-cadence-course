# Chapter 2 Day 1 - access(contract) Pattern

Hey hey, and welcome to the next Chapter of this course. You survived Chapter 1, and are hopefully masters of capabilities. In this Chaper, we will be talking a lot about Admin rights, and how to navigate giving Admins abilities in your contracts.

## What is `access(contract)`?

As you may recall, `access(contract)` is what you call an access modifier. You place it on variables or functions when you only want them to be accessed inside the contract.

For example, declaring:

```cadence
access(contract) var greeting: String
```

makes `greeting` only readable inside of the contract. Similarly,

```cadence
access(contract) fun changeGreeting(newGreeting: String) {
  self.greeting = newGreeting
}
```

makes `changeGreeting` only callable inside the contract.

## Describing the Problem

Okay Jacob, what are you getting at?

Well, there often arrises a scenario where an Admin wants to be able to call a function on someone else's resource. However, in order for that to happen, the function must be publicly accessible or the Admin won't be able to call it. The issue, though, is that if the funciton is public, then *anyone* can call it. 

## The Solution

The solution is to make a function the Admin can call, that itself calls a function (that is restricted to `access(contract)`) through a public interface.

## Example Problem

For example, let's say we want an Admin to be able to prevent a user from transferring their NFTs. Let's look at a (slimmed down) NFT contract:

```cadence
pub contract ExampleNFT {

  pub resource Collection {
    pub var locked: Bool
    pub var ownedNFTs: @{UInt64: NFT}

    pub fun withdraw(withdrawID: UInt64) {
      pre {
        !locked: "You cannot withdraw NFTs."
      }

      let nft <- self.ownedNFTs.remove(at: withdrawID) 
            ?? panic("This NFT doesn't exist in this Collection.")
      return <- nft
    }

    pub fun lock() {
      self.locked = true
    }

    init() {
      self.locked = false
      self.ownedNFTs <- {}
    }
  }

  pub resource Admin {
    // How will the Admin lock the Collection?
  }

}
```

In this case, we must make the `lock` function public so that the Admin can use a function to lock the Collection. Let's see what that looks like below:

```cadence
pub contract ExampleNFT {

  // We added an interface to call the `lock` function publicly
  pub resource interface CollectionPublic {
    pub fun lock()
  }

  pub resource Collection: CollectionPublic {
    pub var locked: Bool
    pub var ownedNFTs: @{UInt64: NFT}

    pub fun withdraw(withdrawID: UInt64) {
      pre {
        !locked: "You cannot withdraw NFTs."
      }

      let nft <- self.ownedNFTs.remove(at: withdrawID) 
            ?? panic("This NFT doesn't exist in this Collection.")
      return <- nft
    }

    pub fun lock() {
      self.locked = true
    }

    init() {
      self.locked = false
      self.ownedNFTs <- {}
    }
  }

  pub resource Admin {
    pub fun lockUserCollection(collection: &Collection{CollectionPublic}) {
      collection.lock()
    }
  }

}
```

Great! Now the Admin can successfully lock the user's collection. But we now have a problem: anyone can lock a user's collection, because the `lock` function is public.

In order to fix that, let's make the `lock` function have `access(contract)` inside the public interface:

```cadence
pub contract ExampleNFT {

  pub resource interface CollectionPublic {
    // This is now `access(contract)`
    access(contract) fun lock()
  }

  pub resource Collection: CollectionPublic {
    pub var locked: Bool
    pub var ownedNFTs: @{UInt64: NFT}

    pub fun withdraw(withdrawID: UInt64) {
      pre {
        !locked: "You cannot withdraw NFTs."
      }

      let nft <- self.ownedNFTs.remove(at: withdrawID) 
            ?? panic("This NFT doesn't exist in this Collection.")
      return <- nft
    }

    pub fun lock() {
      self.locked = true
    }

    init() {
      self.locked = false
      self.ownedNFTs <- {}
    }
  }

  pub resource Admin {
    pub fun lockUserCollection(collection: &Collection{CollectionPublic}) {
      collection.lock()
    }
  }

}
```

Perfect! Simple, isn't it? Now, the public is not able to call `lock` through the public interface anymore, but the Admin is able to because we made an additional function inside the `Admin` resource to call the `lock` function within the contract.

The only remaining question is: do we want the user to be able to lock their own collection? In the above contract, the Collection owner can still lock their collection because the `lock` function is still `pub` inside the actual resource. Usually, if the user wants to lock their own collection, we can let them do that. But let's say for some reason you don't want to let the user lock their own collection. Well, the solution is also pretty easy:

```cadence
pub contract ExampleNFT {

  pub resource interface CollectionPublic {
    access(contract) fun lock()
  }

  pub resource Collection: CollectionPublic {
    pub var locked: Bool
    pub var ownedNFTs: @{UInt64: NFT}

    pub fun withdraw(withdrawID: UInt64) {
      pre {
        !locked: "You cannot withdraw NFTs."
      }

      let nft <- self.ownedNFTs.remove(at: withdrawID) 
            ?? panic("This NFT doesn't exist in this Collection.")
      return <- nft
    }

    // This is now `access(contract)`
    access(contract) fun lock() {
      self.locked = true
    }

    init() {
      self.locked = false
      self.ownedNFTs <- {}
    }
  }

  pub resource Admin {
    pub fun lockUserCollection(collection: &Collection{CollectionPublic}) {
      collection.lock()
    }
  }

}
```

We simply make the function `access(contract)` inside the resource, so now the only person who can ever lock the collection is the Admin itself.

## Quests

1. Describe the `access(contract)` pattern in words.

2. Come up with your own unique example of when the `access(contract)` pattern could be used.

3. Based on the following diagram, do you think this pattern could also be used with `access(account)`?
<img src="../images/access_modifiers.png" />

4. Using the <a href="https://flow-view-source.com/mainnet/account/0x2d4c3caffbeab845/contract/FLOAT">FLOAT Contract</a>, find at least one example of the `access(contract)` pattern being used (hint: if your answer to quest #3 is correct, you should be able to find one by searching all the public interfaces for a certain function that has a specific access modifier).