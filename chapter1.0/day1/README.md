# Chapter 1 Day 1 - More on Capabilities

Welcome to your first lesson. That's right, it's time to start your painful adventure into learning the extremely weird nuances of Cadence crap. What better subject to start on capabilities, the most painful of them all!

## What are Capabilities?

You should already know what capabilities are. But here is a summary for you...

If you took our <a href="https://github.com/emerald-dao/beginner-cadence-course" target="_blank">Beginner Cadence Course</a>, you'll remember this picture of capabilities:

<img src="../images/capabilities.PNG" />

Quick review of capabilities:
1. `/public/` is available to everyone.
2. `/private/` is available to the account owner and people who the owner gives access to.

It's totally fine if you don't understand `/private/` capabilities yet. We'll get to that in the next day.

## Actually Understanding Capability Types

Often times, beginner Cadence coders are so used to borrowing capabilities immediately that they don't realize capabilities can be passed around and stored just like any other type (`String`, `Int`, `@AnyResource`, etc). 

Let's look at an example of borrowing capabilities immediately, like this:

```cadence
import Test from 0x01

// A script to get all the `ids` in a user's NFT Collection
pub fun main(user: Address): [UInt64] {
  let collection = getAccount(user).getCapability(/public/Test)
            .borrow<&Test.Collection{Test.CollectionPublic}>()!
  return collection.getIDs()
}
```

You should understand that this is:
1. Getting the public account of the user
2. Fetching a public capability
3. Borrowing the actual collection reference
4. Calling `getIDs()` and returning the array of ids.

But, do you *really* understand the 2nd step? Let's break the code down into how it's written in steps:

```cadence
import Test from 0x01

// A script to get all the `ids` in a user's NFT Collection
pub fun main(user: Address): [UInt64] {
  // 1. Getting the public account of the user
  let account: PublicAccount = getAccount(user)

  // 2. Fetching a public capability
  let capability: Capability<&Test.Collection{Test.CollectionPublic}> = publicAccount.getCapability<&Test.Collection{Test.CollectionPublic}>(/public/Test)

  // 3. Borrowing the actual collection reference
  let collection: &Test.Collection{Test.CollectionPublic} = capability.borrow()!

  // 4. Calling `getIDs()` and returning the array of ids.
  return collection.getIDs()
}
```

In this script, you can clearly see that a capability type, more specifically a `Capability<&Test.Collection{Test.CollectionPublic}>` type, is being stored inside the `capability` variable. We can actually *store* this capability in places, like dictionaries, resources, structs, and more. Most often, we store capabilities in resources.

## Storing Capabilities

Let's say we have a contract to allow users to create their own profiles, where each profile has an `id`, `name`, and a reference to a user's Flow Token vault so we can read their balance.

```cadence
import FlowToken from 0x02
import FungibleToken from 0x03

pub contract Identity {

  pub resource Profile {
    pub let id: UInt64
    pub let name: String
    pub let vaultCapability: &FlowToken.Vault{FungibleToken.Balance}

    init(name: String, vaultCapability: &FlowToken.Vault{FungibleToken.Balance}) {
      self.id = self.uuid
      self.name = name
      self.vaultCapability = vaultCapability
    }
  }

  pub fun createProfile(name: String, vaultCapability: &FlowToken.Vault{FungibleToken.Balance}): @Profile {
    return <- create Profile(name: name, vaultCapability: vaultCapability)
  }

}
```

This contract is all good right? 

...

NO! If you said yes, you are an idiot. Just kidding, you're not an idiot. But why is this not right?

Well, references are not `storable` types. You cannot store references somewhere. Why? Well, because that'd be pretty dangerous for a multitude of reasons.

> For more on storable types, see [here](https://developers.flow.com/cadence/language/composite-types#composite-type-fields).

In order to solve this problem, we can actually use capabilities. Let's rewrite our contract:

```cadence
import FlowToken from 0x02
import FungibleToken from 0x03

pub contract Identity {

  pub resource Profile {
    pub let id: UInt64
    pub let name: String
    pub let vaultCapability: Capability<&FlowToken.Vault{FungibleToken.Balance}>

    init(name: String, vaultCapability: Capability<&FlowToken.Vault{FungibleToken.Balance}>) {
      self.id = self.uuid
      self.name = name
      self.vaultCapability = vaultCapability
    }
  }

  pub fun createProfile(name: String, vaultCapability: Capability<&FlowToken.Vault{FungibleToken.Balance}>): @Profile {
    return <- create Profile(name: name, vaultCapability: vaultCapability)
  }

}
```

Now, our goal is achieved and we are legendary Cadence coders. Let's see how we can first set up this profile, and then let's see how to actually read the balance from our profile.

### Storing a Capability Inside Our Profile

Let's look at a transaction to set up our profile by storing our capability inside of it.

```cadence
import Identity from 0x01
import FlowToken from 0x02
import FungibleToken from 0x03

transaction(name: String) {
  
  prepare(signer: AuthAccount) {
    // Fetch capability from our account
    let vaultCapability: Capability<&FlowToken.Vault{FungibleToken.Balance}> = signer.getCapability<&FlowToken.Vault{FungibleToken.Balance}>(/public/flowTokenBalance)
    
    // Create a `@Profile` type
    let profile: @Identity.Profile <- Identity.createProfile(name: name, vaultCapability: vaultCapability)

    // Store `@Profile` type
    signer.save(<- profile, to: /storage/Profile)
    // Link `@Profile` type to the public
    signer.link<&Identity.Profile>(/public/Profile, target: /storage/Profile)
  }

  execute {

  }
}
```

You can see in the first part, we first fetched our capability (that has a type of `Capability<&FlowToken.Vault{FungibleToken.Balance}>`) and then we passed it into our `profile` to be stored.

### Reading Our Profile

Now, let's run a script to actually read our profile, and further, our flow token balance.

```cadence
import Identity from 0x01
import FlowToken from 0x02
import FungibleToken from 0x03

// A script to read the Flow Token balance from a user's profile
pub fun main(user: Address): UFix64 {
  // 1. Getting the public account of the user
  let account: PublicAccount = getAccount(user)

  // 2. Fetching a public capability (the profile)
  let capability: Capability<&Identity.Profile> = publicAccount.getCapability<&Identity.Profile>(/public/Profile)

  // 3. Borrowing the profile reference
  let profile: &Identity.Profile = capability.borrow()!

  // 4. Get the `vaultCapability` capability inside our profile
  let vaultCapability: Capability<&FlowToken.Vault{FungibleToken.Balance}> = profile.vaultCapability

  // 5. Borrowing the actual reference to the vault
  let flowTokenVault: &FlowToken.Vault{FungibleToken.Balance} = vaultCapability.borrow()!

  // 6. Returning the balance on the vault
  return flowTokenVault.balance
}
```

Broken down into steps, you can see how to read our flow token balance from our profile. Of course, you can also write this script like this:

```cadence
import Identity from 0x01
import FlowToken from 0x02
import FungibleToken from 0x03

// A script to read the Flow Token balance from a user's profile
pub fun main(user: Address): UFix64 {
  let profile: &Identity.Profile = getAccount(user).getCapability(/public/Profile)
              .borrow<&Identity.Profile>()!

  let flowTokenVault: &FlowToken.Vault{FungibleToken.Balance} = profile.vaultCapability.borrow()!

  return flowTokenVault.balance
}
```

This script does the exact same thing, but it much more condensed. 

## Valid/Invalid Capabilities

It is important to note that when we fetch capabilities from an account, or use capabilities at all, it is possible that they are totally invalid. Remember, capabilities are merely an arrow, or a pointer, to an actual thing in account storage. But what if that arrow is pointing to something that no longer exists?

When we use capabilities, we usually `borrow` on them to get the actual underlying reference. For example, like we did in the above script:

```cadence
let account: PublicAccount = getAccount(user)

let capability: Capability<&Identity.Profile> = publicAccount.getCapability<&Identity.Profile>(/public/Profile)

// WE ARE BORROWING THE CAPABILITY HERE
let profile: &Identity.Profile = capability.borrow()!
```

Notice that when we borrow, we use the "force-unwrap operator" `!` to unwrap the capability and get the underlying reference, which in this case is `&Identity.Profile`. This is because our capability might be invalid, or nil.

In fact, when we initially fetched our `vaultCapability` from our account, like so:

```cadence
// Fetch capability from our account
let vaultCapability: Capability<&FlowToken.Vault{FungibleToken.Balance}> = signer.getCapability<&FlowToken.Vault{FungibleToken.Balance}>(/public/flowTokenBalance)
```

... the `vaultCapability` here may be invalid. 

> Of course, because `FlowToken.Vault` is properly setup and linked on every Flow account in existence, this is sort of a bad example because this capability will *always* be correct. But just ignore that for now.

### `.check()` Function

To make sure the capability is actually valid, and won't panic when we try to borrow it in the future, we can use the `.check()` function. Let's rewrite our transaction from earlier: 

```cadence
import Identity from 0x01
import FlowToken from 0x02
import FungibleToken from 0x03

transaction(name: String) {
  
  prepare(signer: AuthAccount) {
    let vaultCapability: Capability<&FlowToken.Vault{FungibleToken.Balance}> = signer.getCapability<&FlowToken.Vault{FungibleToken.Balance}>(/public/flowTokenBalance)

    assert(vaultCapability.check(), message: "This capability is invalid!")
    
    // ... continue on here ...
  }
}
```

You can see that we added an `assert` and used the `.check()` function to make sure our capability is correct before moving on with the transaction. This way, we don't store an invalid capability in our profile.

### Valid Capabilities Becoming Invalid

Let's look at an example of when a capability would become invalid.

Using a script similar to the one we saw above...

```cadence
import Identity from 0x01

// A script to read the Flow Token balance from a user's profile
pub fun main(user: Address): String {
  // 1. Getting the public account of the user
  let account: PublicAccount = getAccount(user)

  // 2. Fetching a public capability (the profile)
  let capability: Capability<&Identity.Profile> = publicAccount.getCapability<&Identity.Profile>(/public/Profile)

  // 3. Borrowing the profile reference
  let profile: &Identity.Profile = capability.borrow()!
  
  return profile.name
}
```

... we see an example where we are first fetching a capability to our public profile, borrowing the underlying reference and then returning the name. This works great.

But what happens when the account who owns the Profile resource we're fetching `unlink`s the resource from the public path?

```cadence
transaction(name: String) {
  
  prepare(signer: AuthAccount) {
    // Unlink your public profile
    signer.unlink(/public/Profile)
  }

  execute {

  }
}
```

When you `unlink` something from a /public/ or /private/ path, you are essentially saying that capability no longer exists.

So when we try to run our script again...

```cadence
import Identity from 0x01

pub fun main(user: Address): String {
  let account: PublicAccount = getAccount(user)

  let capability: Capability<&Identity.Profile> = publicAccount.getCapability<&Identity.Profile>(/public/Profile)

  // PANIC HAPPENS HERE
  let profile: &Identity.Profile = capability.borrow()!
  
  return profile.name
}
```

... it doesn't work correctly. Instead, it panics on the force-unwrap of our capability.

This same exact logic can be applied to the `Capability<&FlowToken.Vault{FungibleToken.Balance}>` inside of our Profile resource. If someone were to unlink their flow vault to the public (*you should never do this*), that capability would also become invalid.

## Quests

Consider a scenario where we have two contracts:
1) An NFT Contract representing music records and a collection to store those records
2) An artist's profile that has a link to the artist's music collection

Contract #1:

```cadence
import NonFungibleToken from 0x03

pub contract Record: NonFungibleToken {

    pub var totalSupply: UInt64

    pub event ContractInitialized()
    pub event Withdraw(id: UInt64, from: Address?)
    pub event Deposit(id: UInt64, to: Address?)

    pub let CollectionStoragePath: StoragePath
    pub let CollectionPublicPath: PublicPath
    pub let MinterStoragePath: StoragePath

    pub resource NFT: NonFungibleToken.INFT {
        pub let id: UInt64
        pub let songName: String
    
        init(songName: String) {
            self.id = self.uuid
            self.songName = songName
            Record.totalSupply = Record.totalSupply + 1
        }
    }

    pub resource interface CollectionPublic {
        pub fun deposit(token: @NonFungibleToken.NFT)
        pub fun getIDs(): [UInt64]
        pub fun borrowNFT(id: UInt64): &NonFungibleToken.NFT
        pub fun borrowRecordNFT(id: UInt64): &Record.NFT? {
            post {
                (result == nil) || (result?.id == id):
                    "Cannot borrow the reference: the ID of the returned reference is incorrect"
            }
        }
    }

    pub resource Collection: CollectionPublic, NonFungibleToken.Provider, NonFungibleToken.Receiver, NonFungibleToken.CollectionPublic {
        pub var ownedNFTs: @{UInt64: NonFungibleToken.NFT}

        pub fun withdraw(withdrawID: UInt64): @NonFungibleToken.NFT {
            let token <- self.ownedNFTs.remove(key: withdrawID) ?? panic("missing NFT")
            emit Withdraw(id: token.id, from: self.owner?.address)
            return <- token
        }

        pub fun deposit(token: @NonFungibleToken.NFT) {
            let token <- token as! @Record.NFT
            emit Deposit(id: token.id, to: self.owner?.address)
            self.ownedNFTs[token.id] <-! token
        }

        pub fun getIDs(): [UInt64] {
            return self.ownedNFTs.keys
        }

        pub fun borrowNFT(id: UInt64): &NonFungibleToken.NFT {
            return (&self.ownedNFTs[id] as &NonFungibleToken.NFT?)!
        }
 
        pub fun borrowRecordNFT(id: UInt64): &Record.NFT? {
            if self.ownedNFTs[id] != nil {
                let ref = (&self.ownedNFTs[id] as auth &NonFungibleToken.NFT?)!
                return ref as! &Record.NFT
            }

            return nil
        }

        init () {
            self.ownedNFTs <- {}
        }

        destroy() {
            destroy self.ownedNFTs
        }
    }

    pub fun createEmptyCollection(): @NonFungibleToken.Collection {
        return <- create Collection()
    }

    pub fun createRecord(songName: String): @Record.NFT {
      return <- create Record.NFT(songName: songName)
    }

    init() {
        self.totalSupply = 0

        self.CollectionStoragePath = /storage/RecordCollection
        self.CollectionPublicPath = /public/RecordCollection
        self.MinterStoragePath = /storage/Minter

        emit ContractInitialized()
    }
}
```

Contract #2:

```cadence
import Record from 0x01

pub contract Artist {

  pub resource Profile {
    pub let id: UInt64
    pub let name: String
    pub let recordCollection: Capability<&Record.Collection{Record.CollectionPublic}>

    init(name: String, recordCollection: Capability<&Record.Collection{Record.CollectionPublic}>) {
      self.id = self.uuid
      self.name = name
      self.recordCollection = recordCollection
    }
  }

  pub fun createProfile(name: String, recordCollection: Capability<&Record.Collection{Record.CollectionPublic}>): @Profile {
    return <- create Profile(name: name, recordCollection: recordCollection)
  }

}
```

Then, do the following:

1) Write a transaction to save a `@Record.Collection` to the signer's account, making sure to link the appropriate interfaces to the public path.

2) Write a transaction to mint some `@Record.NFT`s to the user's `@Record.Collection`

3) Write a script to return an array of all the user's `&Record.NFT?` in their `@Record.Collection`

4) Write a transaction to save a `@Artist.Profile` to the signer's account, making sure to link it to the public so we can read it

5) Write a script to fetch a user's `&Artist.Profile`, borrow their `recordCollection`, and return an array of all the user's `&Record.NFT?` in their `@Record.Collection` from the `recordCollection`

6) Write a transaction to `unlink` a user's `@Record.Collection` from the public path

7) Explain why the `recordCollection` inside the user's `@Artist.Profile` is now invalid

8) Write a script that proves why your answer to #7 is true by trying to borrow a user's `recordCollection` from their `&Artist.Profile`
