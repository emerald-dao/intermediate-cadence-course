# Chapter 4 Day 1 - Proving Identity

Helloooooo, and welcome to Chapter 4! This Chapter is going to be filled with a bunch of really useful things that you should absolutely know, but don't necessarily relate to each other.

In today's lesson, I'm going to teach you probably one of the most important things I've learned from developing on Cadence: proving you own an address.

## The Problem

Cadence is very unique compared to other smart contract languages. However, where I would argue it differs most is in its ability to prove who is signing the current transaction.

In Solidity (the smart contract language for EVM chains), you can easily access the signer of a transaction inside the smart contract by doing `msg.sender`. This makes your life very easy. In Cadence, it's actually quite different.

Usually, you have a transaction where you get passed the signer in the prepare phase, like this:

```cadence
transaction() {
  prepare(signer: AuthAccount) {

  }
}
```

This is great, but the `AuthAccount` here only exists within the transaction. This means we have no way of knowing who the signer is inside the contract itself.

For example, let's look at this contract:

```cadence
pub contract Profiles {
  pub let profiles: @{Address: Profile}

  pub resource Profile {
    pub let address: Address
    pub let name: String

    init(_ address: Address, _ name: String) {
      self.address = address
      self.name = name
    }
  }

  // the `address` here should be the person calling the function
  pub fun createProfile(address: Address, name: String) {
    pre {
      self.profiles[address] == nil: "A Profile already exists for your address."
    }
    let profile: @Profile <- create Profile(address, name)
    self.profiles[address] <- profile
  }
}
```

We want users to be able to create `Profile` resources by passing in their address and their name. However, you quickly realize that there's a big problem here. Anyone can call `createProfile`, which means I can just pass in *your* address with *my* name. Then, you won't be able to create a profile anymore because of the pre-condition.

## What About `AuthAccount`?

You may immediately wonder, "Well... can't we just pass in the `AuthAccount` object and access the address from that?"

If you thought that, I applaud you for your creative thinking. After all, an `AuthAccount` can only be fetched if someone signs the transaction. So if the contract is given an `AuthAccount`, that means we have the signer's object.

We could rewrite the contract like so:

```cadence
pub contract Profiles {
  pub let profiles: @{Address: Profile}

  pub resource Profile {
    pub let address: Address
    pub let name: String

    init(_ address: Address, _ name: String) {
      self.address = address
      self.name = name
    }
  }

  // the account is confirmed to be the signer because it is an
  // `AuthAccount`, and there's no other way to fetch it besides
  // someone signing the transaction.
  pub fun createProfile(account: AuthAccount, name: String) {
    pre {
      self.profiles[account.address] == nil: "A Profile already exists for your address."
    }
    let address: Address = account.address
    let profile: @Profile <- create Profile(address, name)
    self.profiles[address] <- profile
  }
}
```

This could work, because we are able to fetch the address by doing `account.address`. 

However, **you should never do this.** There is a rule among Cadence programmers that you should never ever do this. Not because it won't work, but simply because it is almost like dark magic to pass an `AuthAccount` anywhere. The rule is you should keep the `AuthAccount` inside the prepare phase of a transaction. Otherwise, it would encourage users passing around an `AuthAccount`, which is extremely dangerous.

## Using `self.owner!.address`

There is another solution we can use, and one that I use practically all the time.

When a resource is stored inside of account storage, you can use `self.owner` to access the `PublicAccount` of the user who owns that resource. Let's look at an example:

```cadence
pub contract Greeting {

  pub resource Hello {

    pub fun sayHello(): String {
      let ownerAddress: Address = self.owner!.address
      return "Hello! From ".concat(ownerAddress.toString())
    }

  }

  pub fun createHello(): @Hello {
    return <- create Hello()
  }
}
```

What we can do is:
1. Run a transaction that calls `createHello` and stores a `Hello` resource in account storage

```cadence
import Greeting from 0x01

transaction() {
  prepare(signer: AuthAccount) {
    signer.save(<- Greeting.createHello(), to: /storage/Hello)
    signer.link<&Greeting.Hello>(/public/Hello, target: /storage/Hello)
  }
}
```

2. Borrow the `Hello` resource from account storage and call `sayHello`, which will return "Hello! From [owner's address]"

```cadence
// Let's say `user` == 0x01
pub fun main(user: Address): String {
  let hello = getAccount(user).getCapability(/public/Hello)
            .borrow<&Greeting.Hello>()!
  return hello.sayHello() // "Hello! From 0x01"
} 
```

I realize now this is a stupid example, but I hope it helps you understand what `self.owner!.address` is actually doing.

With this new understanding, we can rewrite our profiles example to look something like this:

```cadence
pub contract Profiles {
  pub let profiles: @{Address: Profile}

  pub resource Profile {
    pub let address: Address
    pub let name: String

    init(_ address: Address, _ name: String) {
      self.address = address
      self.name = name
    }
  }

  pub resource Identity {
    pub let name: String 

    pub fun createProfile() {
      pre {
        Profiles.profiles[self.owner!.address] == nil: "A Profile already exists for your address."
      }
      let address: Address = self.owner!.address
      let profile: @Profile <- create Profile(address, name)
      Profiles.profiles[address] <- profile
    }

    init(_ name: String) {
      self.name = name
    }
  }

  pub fun createIdentity(name: String): @Identity {
    return <- create Identity(name)
  }
}
```

In order:
1. Create an `Identity` and store it in our account storage
2. In another transaction, borrow `Identity` and call `createProfile`, which will create a `Profile` with your correct address.

## Quests

1. Explain the `self.owner!.address` pattern.

2. You may be wondering: "Why do we have to add a force-unwrap operator (`!`) after `self.owner`?" Good question! Can you make a guess as to when `self.owner` would be `nil`?

3. Come up with your own example where you utilize `self.owner!.address`, and explain why you had to use it there.

4. Take a look at the FLOAT contract on Mainnet <a href="https://flow-view-source.com/mainnet/account/0x2d4c3caffbeab845/contract/FLOAT">here</a>. Find an example of `self.owner!.address` and explain what it is doing there (hint: look at the `createEvent` function).