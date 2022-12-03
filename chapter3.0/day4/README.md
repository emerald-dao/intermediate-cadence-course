# Chapter 3 Day 4 - Prioritizing nil

This lesson will be super short, but it's an important concept to understand when thinking about the relationship between Cadence (what we're good at), and your frontend client that is *using* your smart contract code.

## Getter Functions

As we know, a "getter" function is something that reads data. This occurs inside of a script when we want to read some data from our contract.

Let's say we have a contract like this:

```cadence
pub contract Profiles {
  pub let profiles: @{UInt64: Profile}

  pub resource Profile {
    pub let name: String
    pub let luckyNumber: UInt64

    init(_ name: String, _ luckyNumber: UInt64) {
      self.name = name
      self.luckyNumber = luckyNumber
    }
  }

  pub fun createProfile(name: String, luckyNumber: UInt64) {
    let profile: @Profile <- create Profile(name, luckyNumber)
    self.profiles[profile.uuid] <- profile
  }
}
```

Pretty simple, right? Now, let's say we want to write a function to fetch a `Profile` reference from the `profiles` dictionary. There are two ways we could write this function:

```cadence
pub fun getProfile(id: UInt64): &Profile? {
  return &self.profiles[id] as &Profile?
}
```

or...

```cadence
pub fun getProfile(id: UInt64): &Profile {
  return (&self.profiles[id] as &Profile?)!
}
```

Although both of these are correct, the first one is actually much better. This is what I call "prioritizing `nil` over a `panic`." The reason being that we want to handle the case where the profile does not exist rather than aborting the whole script. 

## Script Data

Another reason for writing the function this way is because we don't want our script to return errors. Imagine we had written our getter function so that it does *not* return an optional, so our script looks like this:

```cadence
import Profiles from 0x01

pub fun main(id: UInt64): &Profile {
  return Profiles.getProfile(id: id)
}
```

In this case, if the profile does not exist, the client calling this script will actually receive an error, which is sort of strange, right? The script itself is performing perfectly fine, but we were not able to find the profile. So we should have a much better way of communicating to our client that the profile simply doesn't exist by returning `nil`:

```cadence
import Profiles from 0x01

pub fun main(id: UInt64): &Profile? {
  // assuming `getProfile` returns a `nil`
  return Profiles.getProfile(id: id)
}
```

## Conclusion

When writing a getter function, by default you should always return an optional rather than aborting the script.

## Quests

1. Take a look at the `getSetData` function in the TopShot smart contract found <a href="https://github.com/dapperlabs/nba-smart-contracts/blob/master/contracts/TopShot.cdc#L1345">here</a>. Find a different line in that contract that calls the function, and explain why it is beneficial that the function returns an optional rather than `panic`ing.

2. Find a contract on Mainnet that has a getter function that returns an optional type. Explain the benefit of having it written that way. Make sure to paste the link and line #.