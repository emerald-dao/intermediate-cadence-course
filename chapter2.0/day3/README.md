# Chapter 2 Day 3 - Borrowing Contracts

Heyo! This lesson will be pretty short, but it is a very new feature of Cadence that I think is pretty cool, and potentially has some really cool use cases that are undiscovered!

## Video

If you'd like to watch a video, you can here: https://www.youtube.com/watch?v=fM7UYt9p7ik

## The 'Why?'

Why would we want to borrow a contract in Cadence? Well, let's say we want to access some functions inside of a contract but we don't import that contract itself:

```cadence
pub fun main(): UInt64 {
  // Not possible, because we don't import `ExampleNFT`
  return ExampleNFT.totalSupply
}
```

This is sort of a weird use case because you might ask, "Well, why not just import the contract then?" You would be right, we most certainly could.

But there are times where we actually can't import a contract. For example, let's say you have contracts A and B, where B imports contract A:

<img src="../images/contracts.png" />

Now, let's say you want to access some function defined in contract B from contract A. In order to do that, contract A would have to import contract B, but you are not allowed to do that because of circular dependence:

<img src="../images/bad.png" />

What you could do instead is "borrow" contract B inside of contract A, and you wouldn't have to import it anymore.

## Contract Interfaces

At this point, you should know what a contract interface is. One example is `NonFungibleToken`, which is an interface that all NFT contracts must implement on the Flow blockchain.

When you attempt to borrow a contract in Cadence, you have to borrow on a contract interface.

## Example

Let me stop writing and show you an example instead:

```cadence
import NonFungibleToken from 0x02

pub fun main(): UInt64 {
  let account: PublicAccount = getAccount(0x01)
  let borrowedContract = account.contracts.borrow<&NonFungibleToken>(name: "ExampleNFT") ?? panic("contract not found")
  return borrowedContract.totalSupply
}
```

In the example above, we did a few things:
1. Got the public account of a certain address
2. Looked inside the public account for a contract named "ExampleNFT" that implements the `NonFungibleToken` contract interface
3. Accessed the `totalSupply` on the `NonFungibleToken` reference and returned it

## Quests

1. Define your own contract interface and a contract that implements it. Inside the contract, have one function that mutates data and one function that reads that data.

2. In a script, attempt to call the read function inside the contract without importing it.

3. In a transaction, attempt to call the mutate function inside the contract without importing it.