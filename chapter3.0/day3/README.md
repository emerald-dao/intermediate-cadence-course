# Chapter 3 Day 3 - Storage Iteration

Yo yo! Today, we'll be talking about a feature actually quite new to Cadence: Storage Iteration! It may look hard at first, but it's actually pretty easy.

## Video

Check out a video of this lecture here: https://www.youtube.com/watch?v=bwtQ0ettqLU

## What is Storage Iteration?

Storage Iteration basically allows you to loop over a user's account to view everything in their account storage. For example, all their NFTs, all their tokens, all their resources, etc. 

## Basic Functions

There are actually a few functions we can use off the bat to query information about the paths inside of an account. Let's look at them here:

**PublicAccount.publicPaths**
Returns an array of all the public paths inside of a user's account.

ex.

```cadence
pub fun main(user: Address): [PublicPath] {
  let publicAccount: PublicAccount = getAccount(user)
  return publicAccount.publicPaths
}
```

**AuthAccount.storagePaths**
Returns an array of all the storage paths inside of a user's account.

ex.

```cadence
pub fun main(user: Address): [StoragePath] {
  let authAccount: AuthAccount = getAuthAccount(user)
  return authAccount.storagePaths
}
```

**AuthAccount.privatePaths**
Returns an array of all the private paths inside of a user's account.

ex.

```cadence
pub fun main(user: Address): [PrivatePath] {
  let authAccount: AuthAccount = getAuthAccount(user)
  return authAccount.privatePaths
}
```

## Looping Over an Account

More interesting, and also more complicated, is the ability to actually loop over every path and discover some information.

In order to do that, I will teach you this in steps:
1. If we want to loop over public paths, we need a `PublicAccount`. If we want to loop over private/storage paths, we need an `AuthAccount`.
2. We need to define an "iteration function" to tell Cadence what to do on each path.
3. In the function, on each iteration, we return `true` if we want to continue looping, or `false` if we want to stop.
4. Then, we pass the "iteration function" to a different function that let's us loop over paths. For public paths, `forEachPublic`. For storage paths, `forEachStored`. For private paths, `forEachPrivate`.

Let's look at a very simple example. Let's say we ran a transaction that looked like this:

```cadence
transaction() {
  prepare(signer: AuthAccount) {
    signer.save(1, to: /storage/one)
    signer.save(2, to: /storage/two)
    signer.save("hello", to: /storage/hello)
    signer.save(3, to: /storage/three)
  }
}
```

It may look weird because we usually save resources to account storage, but I'm making this example very simple for you.

Now, let's loop over the storage paths inside of an account:

```cadence
pub fun main(user: Address) {
  // 1. Get the AuthAccount
  let authAccount: AuthAccount = getAuthAccount(user)

  // 2. Define an iteration function
  let iterationFunction = fun (path: StoragePath, type: Type): Bool {
    if type == Type<String>() {
        log("I found a string! Done looping.")
        // 3. Return false to stop iterating
        return false
    }
    // 3. Return false to stop iterating
    return true
  }

  // 4. Pass the iteration function to `forEachStored`
  authAccount.forEachStored(iterationFunction)
}
```

The script will loop over all of the storage paths and, on each loop, give your iteration function:
1. The path you are currently on (`path`)
2. The type of the thing at that path (`type`)

On the first loop:
- `path` = /storage/one
- `type` = Int

On the second loop:
- `path` = /storage/two
- `type` = Int

On the third loop:
- `path` = /storage/hello
- `type` = String

... and then it stops. Why? Because during that iteration, we said "if it's a String type, return false." Which stops the iteration.

**NOTE: The iteration function *does not* maintain ordering. This means it is very possible that we actually did iterate over `/storage/three`. I was simply just trying to make an example.**

## Real Example

So, why would we use this? Well, what if we wanted to loop over all of the user's NFT collections, and return the `id`s the user has for each. Let's try it below:

```cadence
import NonFungibleToken from 0x02

pub fun main(user: Address): {Type: [UInt64]} {
  let answer: {Type: [UInt64]} = {}
  let authAccount: AuthAccount = getAuthAccount(user)
  
  let iterationFunction = fun (path: StoragePath, type: Type): Bool {
    // `isSubtype` is a function we can call on a Type to check if its parent
    // type is what we provide as the `of` parameter. In this case, we're essentially
    // saying, "if the current type of what we're looking at is a `@NonFungibleToken.Collection`,
    // then...
    if type.isSubtype(of: Type<@NonFungibleToken.Collection>()) {
        // We can borrow a broader `&NonFungibleToken.Collection` type here because we know
        // the type actually stored here is, in fact, a `@NonFungibleToken.Collection`. We will
        // be restricted to the functions defined inside of the `NonFungibleToken` contract, but
        // that's okay.
        let collection = authAccount.borrow<&NonFungibleToken.Collection>(from: path)! // we can force-unwrap here because we know it exists
        let collectionIDs: [UInt64] = collection.getIDs()
        answer[type] = collectionIDs
    }

    return true
  }

  authAccount.forEachStored(iterationFunction)

  return answer
}
```

Please make sure to read the comments I left on the script to get a better understanding of what's happening.

After running this script, we should get a dictionary that maps a NFT's type to an array of all the `id`s a user has for that NFT collection.

## Quests

1. Take the script that we made in today's lesson to iterate over a user's storage paths and get all their NFT ids. Run that script on Mainnet with your address and see what it returns.

2. Take the script that we made in today's lesson to iterate over a user's storage paths and get all their NFT ids. Change this script to instead iterate public paths. A few questions will arise:
- How do I know if the current iteration is an NFT collection?
- Once I've figured that out, how do I borrow the collection from the account? (Hint: use `NonFungibleToken.CollectionPublic`)
- Once I do borrow it, how do I know the collection is *actually* a NFT collection, and not a random resource that implements `NonFungibleToken.CollectionPublic`?

> Warning: Quest #2 is going to be difficult. It requires a lot of thinking on your end. Use concepts you learned throughout this course to help you complete this.