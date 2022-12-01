# Chapter 3 Day 1 - Organizing Information in a Script

This chapter is going to focus mostly on scripts. To start, let's do something easy: organizing information we return back to our client.

## What's the Problem?

Let's say we have a script like this:

```cadence
pub fun main() {
  let var1: String = "Hello"
  let var2: Int = 3

  // return both var1 and var2 to the client
}
```

There's no easy way to return both of these to a client. Technically, we could use `AnyStruct` to return something like this:

```cadence
pub fun main(): [AnyStruct] {
  let var1: String = "Hello"
  let var2: Int = 3

  let array: [AnyStruct] = [var1, var2]
  return array
}
```

...but that's super messy. Now our client will have to iterate over an array with no context as to what those values are.

## Using Structs

Using the above example, what if we were able to create a struct to group these two things together?

```cadence
pub fun main(): Data {
  let var1: String = "Hello"
  let var2: Int = 3

  return Data(var1, var2)
}

pub struct Data {
  pub let var1: String
  pub let var2: Int

  init(_ var1: String, _ var2: Int) {
    self.var1 = var1
    self.var2 = var2
  }
}
```

Now, our client will receive some data that looks like:

```json
{
  "var1": "Hello",
  "var2": 3
}
```

## Real-World Scenario

Using a more complex example, let's say we wanted to run a script to borrow a FLOAT NFT reference from someone's account:

```cadence
import FLOAT from 0x2d4c3caffbeab845

pub fun main(account: Address, id: UInt64): &FLOAT.NFT? {
  let floatCollection = getAccount(account).getCapability(FLOAT.FLOATCollectionPublicPath)
                        .borrow<&FLOAT.Collection{FLOAT.CollectionPublic}>()
                        ?? panic("Could not borrow the Collection from the account.")
  let float = floatCollection.borrowFLOAT(id: id)
  return float
}
```

This is simple enough. But what if we also wanted to return the `totalSupply` of all FLOATs in the same transaction? Because we only have one return value, this would be hard. But we know we can use structs to group our data:

```cadence
import FLOAT from 0x2d4c3caffbeab845

pub fun main(account: Address, id: UInt64): Data {
  let floatCollection = getAccount(account).getCapability(FLOAT.FLOATCollectionPublicPath)
                        .borrow<&FLOAT.Collection{FLOAT.CollectionPublic}>()
                        ?? panic("Could not borrow the Collection from the account.")
  let float = floatCollection.borrowFLOAT(id: id)
  return Data(float, FLOAT.totalSupply)
}

pub struct Data {
  pub let float: &FLOAT.NFT?
  pub let totalSupply: UInt64

  init(_ float: &FLOAT.NFT?, _ totalSupply: UInt64) {
    self.float = float
    self.totalSupply = totalSupply
  }
}
```

Now our client can successfully have both pieces of data :)

## Quests

1. Come up with your own script that groups data together.

2. Use a real-world example (on Mainnet) grouping data together.

3. Write a script that returns:
- a FLOAT NFT's reference
- FLOAT's total supply
- a Flovatar NFT's reference
- Flovatar's total supply
- Flow Token's total supply
- a reference to your public `&FlowToken.Vault{FungibleToken.Receiver}`
- a capability to your public `Capability<&FlowToken.Vault{FungibleToken.Receiver}>`

Hints:
- FLOAT's contract address: 0x2d4c3caffbeab845
- Flovatar's contract address: 0x921ea449dffec68a
- Find all the core contract's addresses here: https://developers.flow.com/flow/core-contracts

You can use the <a href="https://flow-view-source.com/">Flow View Source</a> to discover contracts and learn how to read from them.