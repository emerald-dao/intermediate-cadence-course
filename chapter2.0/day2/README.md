# Chapter 2 Day 2 - access(account)

Hey folks! Welcome back to another Day. We are making our way back to the grand topic of **access control!** "Yaaaaaaaay," said nobody. But don't worry, this one will actually be quite short, because this is a relatively simple concept to understand. 

## Access Control

As you already know, access control is Cadence's fun way of allowing us to specifically say where certain variables and functions are allowed to be *read* and *written*. More specifically, each "access modifier" (like `pub`, `access(contract)`, etc) each have a *read* and *write* scope.

Take a look at the image below to recap what each access modifier means:

<img src="../images/access_modifiers.png" />

## So what is `access(account)`?

Now, you may have already learned what `access(account)` means. But I wanted to include it in the intermediate course because it actually doesn't get used as much as it should...

For those that don't know, `access(account)` allows you to:
- *Read* that variable (or call that function) in the current, inner, and other contracts in the same account
- *Write* that variable in the current and inner scope

The interesting part is obviously the read element. What can we do with something like this? Well, let's say we deploy this contract to account `0x01`:

```cadence
pub contract Greeting {
  access(account) fun sayHello() {
    log("Hello!")
  }
}
```

We can also deploy another contract to account `0x01` that does this:

```cadence
import Greeting from 0x01

pub contract Person {

  pub resource Human {
    pub fun speak() {
      Greeting.sayHello()
    }
  }

}
```

Pretty simple right? We are calling a function from a different contract, but still in the same account.

## Actually Useful Stuff

This should all make sense, but let's see some interesting reasons why we'd use `access(account)`. 

The most important reason is when we want to cleanly handle Admin rights. Let's say we have a whole bunch of NFT Contracts inside account `0x01`:

```cadence
import NonFungibleToken from 0x02

pub contract Sword: NonFungibleToken {

  pub resource NFT: NonFungibleToken.INFT {
    pub let id: UInt64

    init() {
      self.id = self.uuid
    }
  }

  // ... more code here ...

}
```

```cadence
import NonFungibleToken from 0x02

pub contract Shield: NonFungibleToken {

  pub resource NFT: NonFungibleToken.INFT {
    pub let id: UInt64

    init() {
      self.id = self.uuid
    }
  }

  // ... more code here ...

}
```

```cadence
import NonFungibleToken from 0x02

pub contract Bow: NonFungibleToken {

  pub resource NFT: NonFungibleToken.INFT {
    pub let id: UInt64

    init() {
      self.id = self.uuid
    }
  }

  // ... more code here ...

}
```

We just defined 3 NFT Contracts: `Sword`, `Shield`, and `Bow`. Now, for each, let's say we only want the Admin to be able to mint the NFTs. Let's add some code to our contracts:

```cadence
import NonFungibleToken from 0x02

pub contract Sword: NonFungibleToken {

  pub resource NFT: NonFungibleToken.INFT {
    pub let id: UInt64

    init() {
      self.id = self.uuid
    }
  }

  // ... more code here ...

  pub resource Admin {
    pub fun mint(): @NFT {
      return <- create NFT()
    }
  }

  init() {
    self.account.save(<- create Admin(), to: /storage/SwordAdmin)
  }

}
```

```cadence
import NonFungibleToken from 0x02

pub contract Shield: NonFungibleToken {

  pub resource NFT: NonFungibleToken.INFT {
    pub let id: UInt64

    init() {
      self.id = self.uuid
    }
  }

  // ... more code here ...

  pub resource Admin {
    pub fun mint(): @NFT {
      return <- create NFT()
    }
  }

  init() {
    self.account.save(<- create Admin(), to: /storage/ShieldAdmin)
  }

}
```

```cadence
import NonFungibleToken from 0x02

pub contract Bow: NonFungibleToken {

  pub resource NFT: NonFungibleToken.INFT {
    pub let id: UInt64

    init() {
      self.id = self.uuid
    }
  }

  // ... more code here ...

  pub resource Admin {
    pub fun mint(): @NFT {
      return <- create NFT()
    }
  }

  init() {
    self.account.save(<- create Admin(), to: /storage/BowAdmin)
  }

}
```

Hmm, this is starting to get repetitive right? We have 3 different `Admin` resources, each defined in their respective NFT Contracts, just so we can mint NFTs. On top of that, we have to store the 3 Admin resources at different storage paths, which is pretty annoying.

What if we wanted instead to just have 1 Admin that could mint from each? Well, let's start by making a fourth contracts that is deployed to the same account:

```cadence
pub contract Admin {

  pub resource NFTMinter {
    // how do we mint NFTs 
    // from a different contract?
  }

  init() {
    self.account.save(<- create NFTMinter(), to: /storage/AdminNFTMinter)
  }

}
```

The question now is: "How do we mint NFTs from a different contract?" Of course, resources can *only* be created in their own contract, so lets try to define a function in each contract with `access(account)` to mint NFTs:

```cadence
import NonFungibleToken from 0x02

pub contract Sword: NonFungibleToken {

  pub resource NFT: NonFungibleToken.INFT {
    pub let id: UInt64

    init() {
      self.id = self.uuid
    }
  }

  // ... more code here ...

  access(account) fun mint(): @NFT {
    return <- create NFT()
  }

}
```

```cadence
import NonFungibleToken from 0x02

pub contract Shield: NonFungibleToken {

  pub resource NFT: NonFungibleToken.INFT {
    pub let id: UInt64

    init() {
      self.id = self.uuid
    }
  }

  // ... more code here ...

  access(account) fun mint(): @NFT {
    return <- create NFT()
  }

}
```

```cadence
import NonFungibleToken from 0x02

pub contract Bow: NonFungibleToken {

  pub resource NFT: NonFungibleToken.INFT {
    pub let id: UInt64

    init() {
      self.id = self.uuid
    }
  }

  // ... more code here ...

  access(account) fun mint(): @NFT {
    return <- create NFT()
  }

}
```

We can now call this function from our `Admin` contract:

```cadence
import Sword from 0x01
import Shield from 0x01
import Bow from 0x01

pub contract Admin {

  pub resource NFTMinter {
    pub fun mintSword(): @Sword.NFT {
      return <- Sword.mint()
    }

    pub fun mintShield(): @Shield.NFT {
      return <- Shield.mint()
    }

    pub fun mintBow(): @Bow.NFT {
      return <- Bow.mint()
    }
  }

  init() {
    self.account.save(<- create NFTMinter(), to: /storage/AdminNFTMinter)
  }

}
```

Now, our code is much more efficient, and we only have to handle 1 `NFTMinter` resource instead of storing 3 resources at totally different storage paths.

## Quests

1. Design your own contracts such that you use at least one `access(account)` function that gets called in a contract within the same account. Explain why you had to use `access(account)`.

2. Starting from this contract: https://flow-view-source.com/mainnet/account/0x921ea449dffec68a/contract/Flovatar
- Find 1 variable that uses `access(account)`
- Find 1 function that uses `access(account)`
- Using the function you found, explain why it uses that access modifier and where it gets called in a different contract in that same account