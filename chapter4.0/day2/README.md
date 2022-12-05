# Chapter 4 Day 2 - Time in Cadence

Welcome back to another lesson, it's your favorite (and hottest) instructor back again to teach you how to measure time in Cadence. Woohoo!

## Video

You can watch a video on this lesson here: https://www.youtube.com/watch?v=oBwFnsBSaK8

## How to Represent Time

The answer is short: a unix timestamp. A unix timestamp is a number representing the amount of seconds since January 1st, 1970. Check out <a href="https://www.unixtimestamp.com/">this website</a> to learn more. 

More specifically to Cadence, the type we will use to represent the unix timestamp is a `UFix64`.

## How to Get the Current Time

In Cadence, you can use the following function to get the current unix timestamp: `getCurrentBlock().timestamp`. 

`getCurrentBlock` is a built-in function that you can use to get information about the block the current transaction is being executed in. Each block has an associated timestamp, so you can easily access the current time of the block with the `timestamp` variable.

Note that `timestamp` will not necessarily be the exact current time, rather it will be the time of the block. The block can certainly be out of sync with real-time, but it is close enough so that we can use it.

## Calculating Time

Let's say we want to make a resource that allows users to deposit $FLOW token, and after one hour, they are allowed to withdraw that $FLOW again. I have no idea why you would make a resource like this, so don't ask me why. Just follow along LOL.

```cadence
import FlowToken from 0x02
pub contract Test {

  pub resource Lock {
    pub let vault: @FlowToken.Vault
    // The unix timestamp of the last time the user deposited
    pub let lastDeposited: UFix64

    pub fun depsosit(vault: @FlowToken.Vault) {
      self.vault.deposit(from: <- vault)
      self.lastDeposited = getCurrentBlock().timestamp
    }

    pub fun withdraw(): @FlowToken.Vault {
      pre {
        // 60 seconds in a minute, 60 minutes in an hour.
        getCurrentBlock().timestamp >= self.lastDeposited + 60*60: 
          "It has not been an hour since you last deposited."
      }
      return <- vault.withdraw(amount: self.vault.balance) as! @FlowToken.Vault
    }

  }
}
```

Not too bad, right? When we deposit, we make `lastDeposited` equal to the current unix timestamp by using `getCurrentBlock().timestamp`. Then, when we call the withdraw function, we check if the current unix timestamp (again, `getCurrentBlock().timestamp`) is greater or equal to the time we last deposited, plus (60 x 60). The reason we multiply 60 by 60 is because unix timestamps are all in seconds. So we have to figure out how many seconds there are in an hour. If you're not a complete nincompoop, you'll realize there are 60 seconds in a minute, and 60 minutes in an hour. So there are (60 x 60 = 3600) seconds in an hour.

## Quests

1. Using the same exact example as we did in the lesson, modify the code to allow the user to withdraw if it has been (submit a separate answer for each):
- 1 day
- 1 week
- 1 month
- 1 year
- 15 years, 8 months, 3 weeks, 1 day, 12 hours, 14 minutes, and 12 seconds (hahahahahhhahahahaha I'm so sorry)

2. How do you get the current unix timestamp in Cadence?

3. Is it possible to make a smart contract (on Flow) automatically execute code after a certain amount of time? Why or why not (using basic principles of smart contracts)?