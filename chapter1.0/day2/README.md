# Chapter 1 Day 2 - Private Capabilities



## What are They?

If you took our <a href="https://github.com/emerald-dao/beginner-cadence-course" target="_blank">Beginner Cadence Course</a>, you'll remember this picture of capabilities:

<img src="../images/capabilities.PNG" />

Quick review:
1. `/storage/` is only accessible to the account owner. We use `.save()`, `.load()` and `.borrow()` functions to interact with it.
2. `/public/` is available to everyone.
3. `/private/` is available to the account owner and people who the owner gives access to.

The `/private/` part is where private capabilities come in. **Private Capabilities make it so that an account owner can give people specific access to their account**. Think of private capabilities like a key that you pass around only to certain people so they can unlock functions on a resource in your account.

## When Would We Use Them?

We should use private capabilities when we want to give certain people or things special access to a resource in our account. Here are some scenarios:

- You want to give a marketplace contract the ability to withdraw NFTs from your storage when someone buys an NFT from you
- You want to give a fellow community manager Admin rights over your NFT Collection so they can also mint NFTs
- You want to give a friend the ability to withdraw NFTs from your NFT Collection because you trust them

We will walk through the second scenario in a later section since NFTs are usually pretty easy to understand if you have a grasp over Cadence code.

## Using Private Capabilities

As we learned in the previousd day, **you need a place to store the capability.** 