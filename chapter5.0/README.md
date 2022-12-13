# Chapter 5 - Assessment

Why hello there! You have officially made it to the end of this Intermediate Cadence Course. I congratulate you and hope you enjoyed taking this course.

But before you run away and cheat on me with a different course, I have a final assessment for you. Yes, it will be hard. Or maybe not? If it is, then get rekt. If it's not, then I guess I'm just a great instructor.

This assessment will walk you through a progressive example that will incorporate a lot of different concepts I taught you throughout this course. 

> NOTE: This assessment is a lot of DOING, and very little instruction. This means it will be a bit open-ended, and there may be different ways of doing things. The point of this is to make you think, have you STRUGGLE, and learn on your own. Use each Part as a milestone/checkpoint.

> NOTE #2: Throughout this assessment, I may not tell you exactly where to find things. For example, the core contracts or even 3rd party contracts. The point of this is to make you better at finding information you need on the internet, which is surprisingly a really important skill. Or, ask a friend!

Let's go!

## Part 1

a) Write your own Fungible Token contract that implements the `FungibleToken` standard on Flow. Name the contract whatever you want. *NOTE: It is okay to find an example implementation online. But that implementation may overcomplicate the solution. So you may want to be careful.*

b) Inside the contract, define a resource that handles minting. Make it so that only the contract owner can mint tokens.

c) You will find that inside your `deposit` function inside the `Vault` resource you have to set the incoming vault's balance to `0.0` before destroying it. Explain why.

## Part 2

a) Write the following transactions:
- MINT: Mint tokens to a recipient.
- SETUP: Properly sets up a vault inside a user's account storage.
- TRANSFER: Allows a user to transfer tokens to a different address.

b) Write the following scripts:
- READ BALANCE: Reads the balance of a user's vault.
- SETUP CORRECTLY?: Returns `true` if the user's vault is set up correctly, and `false` if it's not.
- TOTAL SUPPLY: Returns the total supply of tokens in existance.

## Part 3

Modify your transactions/scripts from Part 2 as such:

- SETUP: Ensure that this transaction first identifies if the user's vault is already set up poorly, and subsequently fix it if so. If it's not already set up, then simply set it up.
- READ BALANCE:
  - Ensure that your script work with vaults that are not set up correctly, and subsequently (temporarily) fix them so that it will always return a balance without fail.
  - Make sure that the balance being returned is from a vault that is guaranteed to be your token's type, and not some random vault that implements the `FungibleToken` interface.
  - Using comments in the script, explain 2 ways in which you can guarantee that the above requirement is true. One way must be using resource identifiers.

## Part 4

a) Modify your token contract so that the Admin is allowed to withdraw tokens from a user's Vault at any time, and then in the same function, deposit them an equivalent amount of $FLOW tokens. *HINT: This will require more than simply adding a function.*

b) Write a transaction (that is only signed by the Admin) that executes section a).

## Part 5

a) Write a script that returns the balance of the user's $FLOW vault, and your custom vault. Make it organized so the client can easily read it.

b) Write a script that neatly returns (at a minimum) the resource identifier and balance of all the (official) Fungible Token vaults that the user has in their account storage. You can be creative in whatever other information you want to return.

## Part 6

a) Write a second contract, `Swap`, that allows users deposit $FLOW and receive your custom token in return. The amount of tokens the user should get in return is `2*(THE TIME SINCE THEY LAST SWAPPED)`. 

b) In the swapping function, make sure to prove who the person is that is attempting a swap. In other words, make sure someone couldn't swap for you. You must implement this function two ways:
  - Using a custom resource you define to represent identity, and passing in a `@FlowToken.Vault` as an argument for the payment.
  - Using a reference to the user's Flow Token vault that proves only they could pass such a reference in, and subsequently getting the address of the owner from that reference.

## Part 7

a) Ensure all transactions are architected properly, as seen in Chapter 4 Day 3.

## Conclusion

That's it, folks! Do a little happy dance and pat your self on the back. 

I mean it. Don't just read it, do it! I'm waiting...