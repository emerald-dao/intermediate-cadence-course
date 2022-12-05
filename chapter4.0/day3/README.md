# Chapter 4 Day 3 - Structuring Transactions Correctly

Hi! In today's lesson, you won't be learning anything entirely new per say... but I will be showing you something that allows me to immediately tell the difference between a noobie and advanced Cadence developer. In addition, today's lesson will tie in closely to the greater security / readability of our code.

## Up Until Now

If you took our <a href="https://github.com/emerald-dao/beginner-cadence-course" target="_blank">Beginner Cadence Course</a>, you will remember that I briefly mentioned the "prepare" phase and "execute" phase of transactions. Let me paste a part from that:

> You can see this being done in the prepare portion of the transaction, and that's the whole point of the prepare phase: to access the information/data in your account. On the other hand, the execute phase can't do that. But it can call functions and do stuff to change the data on the blockchain. NOTE: In reality, you never actually need the execute phase. You could technically do everything in the prepare phase, but the code is less clear that way. It's better to separate the logic.

While it is entirely true you could perform all code in the prepare phase without problems, this is not an encouraged way to do things. So, if you've been wondering what to put in the prepare phase and what to put in the execute phase, this is for you.

## prepare vs. execute

I think we all know the difference between these two: the "prepare" phase has access to the signer's `AuthAccount`, and the "execute" phase doesn't.

To make our transactions more organized, and to improve the readability of our code, follow this rule of thumb:

1. In the prepare phase, access all the data inside the user's `AuthAccount` and store them as local variables
2. In the execute phase, do everything else (utilizing the local variables from the prepare phase)

## Example

Let's say we want to run a transaction that transfers $FLOW from the signer to a recipient. You could write the transaction like so:

```cadence
import FlowToken from 0x02
import FungibleToken from 0x03

transaction(recipient: Address, amount: UFix64) {
  prepare(signer: AuthAccount) {
    let flowVault = signer.borrow<&FlowToken.Vault>(from: /storage/flowTokenVault)!
    let tokens <- flowVault.withdraw(amount: amount)

    let recipientVault = getAccount(recipient).getCapability(/public/flowTokenReceiver)
              .borrow<&FlowToken.Vault{FungibleToken.Receiver}>()!
    recipientVault.deposit(from: <- tokens)
  }

  execute {

  }
}
```

This will work totally fine, but it screams one thing: NOOB! Don't be a noob. After all, I am your instructor, and you can't embarass me like this.

Instead, you can structure your code like so:

```cadence
import FlowToken from 0x02
import FungibleToken from 0x03

transaction(recipient: Address, amount: UFix64) {

  // Local variables
  let FlowVault: &FlowToken.Vault
  let RecipientVault: &FlowToken.Vault{FungibleToken.Receiver}

  prepare(signer: AuthAccount) {
    // Borrow signer's `&FlowToken.Vault`
    self.FlowVault = signer.borrow<&FlowToken.Vault>(from: /storage/flowTokenVault)!
    // Borrow recipient's `&FlowToken.Vault{FungibleToken.Receiver}`
    self.RecipientVault = getAccount(recipient).getCapability(/public/flowTokenReceiver)
              .borrow<&FlowToken.Vault{FungibleToken.Receiver}>()!
  }

  // All execution
  execute {
    let tokens <- self.FlowVault.withdraw(amount: amount)
    self.RecipientVault.deposit(from: <- tokens)
  }
}
```

Now this... this screams "I AM SUPER COOL AND KNOW WHAT I'M DOING!" It's also super attractive. 

As you can see, the difference is that we use the prepare phase to access everything inside of accounts. In a way, it's like the "setup" of the transaction. We store all of the variables/references inside local variables that get defined before the prepare phase.

The execute phase then takes those local variables and performs actions on them.

## Who Do This?

Of course, there are reasons to do this besides looking cool.

As developers, it is our absolutely duty to make code as readable and clear as possible. This is not only for ourselves and other developers, but for users to be able to develop trust with our applications and feel safe. Asking users to manipulate their assets by trusting our code is a major ask that often gets taken for granted. If at any point we can make code a little cleaner, and a little easier to track, we can help users be safe in an increasingly complicated technical world.

By structuring our code properly, users will be able to worry about only the prepare phase to see what is being accessed on their account. They can completely ignore the execute phase (and all the code with it) if they notice no scary references are being borrowed from their account in the prepare phase.

## Quests

1. Using the <a href="https://flow-view-source.com/mainnet/account/0x2d4c3caffbeab845/contract/FLOAT">FLOAT contract</a>, write a properly structured transaction on Mainnet that transfers a FLOAT with a specific `id` to a `recipient`.

2. Using the <a href="https://flow-view-source.com/mainnet/account/0x921ea449dffec68a/contract/Flovatar">Flovatar contract</a>, write a transaction on Mainnet that properly sets up a user's NFT Collection.