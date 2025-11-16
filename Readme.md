# Anchor Flash Loan

This is a Solana Anchor program, a flash loan that lets users borrow tokens and repay them within the same transaction. I learned this from Blueshift, and I'm documenting what I learned along the way. [Blueshift challenge link](https://learn.blueshift.gg/en/challenges/anchor-flash-loan)

## About

A flash loan program that allows users to:

- Borrow tokens instantly at the start of a transaction
- Use the borrowed tokens for trading, arbitrage, or other operations
- Repay the loan plus a fee before the transaction ends

The key thing I learned is that flash loans rely on something called **instruction introspection**. The ability for a program to look at other instructions in the same transaction. This was completely new to me and blew my mind!

## What I Learned About Instruction Introspection

At first, I thought programs could only see the current instruction they're executing. But instruction introspection is like having X-ray vision for transactions your program can examine and analyze other instructions within the same transaction bundle, even instructions that haven't executed yet!

In the program, we use instruction introspection in two ways:

1. **In the `borrow` function**: I look ahead to verify that a `repay` instruction exists later in the transaction. This ensures the borrower will repay the loan.

2. **In the `repay` function**: I look back to verify the `borrow` instruction exists and extract the borrowed amount from its data.

This is made possible by accessing the **Instructions Sysvar** account, which contains all instruction data for the current transaction. I learned that sysvars are special accounts that Solana provides to programs, and the Instructions Sysvar is like a window into the entire transaction.

The coolest part is that this enables flash loans to be risk free for the lender. Since Solana transactions are atomic (all or nothing), if the repayment fails, the entire transaction rolls back including the borrow! So either the loan gets repaid, or it never happened.

## Understanding the Account Structure

I learned that in Anchor, you define accounts using a struct with constraints. The `Loan` struct has several accounts:

- **`borrower`** - The user requesting the flash loan (marked as `mut` because their token balance changes)
- **`protocol`** - A PDA that owns the protocol's liquidity pool (also `mut` because it holds the tokens)
- **`mint`** - The token being borrowed
- **`borrower_ata`** - The borrower's Associated Token Account (created if needed with `init_if_needed`)
- **`protocol_ata`** - The protocol's Associated Token Account (source of funds, marked `mut`)
- **`instructions`** - The Instructions Sysvar account for introspection
- Standard programs: `token_program`, `associated_token_program`, `system_program`

## Borrow Function

Here's what I learned while implementing `borrow`:

**Check the borrow amount** - I use `require!` to make sure the amount is greater than zero. This prevents invalid borrow requests.

**Access the Instructions Sysvar** - This was tricky at first. I learned that I need to:
1. Get the account info from `ctx.accounts.instructions`
2. Borrow the data mutably to read it
3. Extract the instruction count from the first 2 bytes

**Look ahead for repay instruction** - I learned to use `load_instruction_at_checked` to load an instruction at a specific index. Since `repay` should be the last instruction, I check `len - 1`. Then I verify:
- The program ID matches our program
- The instruction discriminator matches the `Repay` instruction

If the repay instruction doesn't exist, I return an error. This ensures the borrower can't just borrow without repaying!

**Transfer tokens with PDA signing** - This was my first time using a PDA as a signer. I learned that to sign on behalf of a PDA, I need to provide the seeds that created it, plus the bump:

```rust
let seeds = &[
    b"protocol".as_ref(),
    &[ctx.bumps.protocol]
];
let signer_seeds = &[&seeds[..]];

transfer(
    CpiContext::new_with_signer(
        ctx.accounts.token_program.to_account_info(), 
        Transfer { ... }, 
        signer_seeds
    ), 
    borrow_amount
)?;
```

The `CpiContext::new_with_signer` is different from `new` because the protocol PDA needs to sign the transfer. The seeds prove that we are allowed to move funds from this specific PDA.

## Repay Function

I had to learn several new concepts for the repay function:

**Verify the repay instruction** - First, I check that the current instruction is actually a repay instruction by looking at the last instruction in the transaction and verifying its discriminator. I also check that the accounts match (borrower_ata and protocol_ata).

**Extract the borrowed amount** - This was the trickiest part! I learned that Anchor instruction data has a specific format:
- First 8 bytes: instruction discriminator
- Next bytes: instruction parameters


**Transfer back with borrower signing** - Unlike the borrow, the repay transfer is signed by the borrower (not a PDA), so I use `CpiContext::new` instead of `new_with_signer`. The borrower transfers the borrowed amount plus the fee back to the protocol.

## Error Handling

I learned about Anchor's error system. Created a custom error enum with various errors:

- **InvalidIx** - When an instruction doesn't match what we expect
- **InvalidAmount** - When the borrow amount is zero or invalid
- **MissingRepayIx** - When the borrow instruction doesn't find a repay instruction
- **MissingBorrowIx** - When the repay instruction doesn't find a borrow instruction
- **InvalidBorrowerAta** / **InvalidProtocolAta** - When account addresses don't match
- **Overflow** - When math operations overflow
- And more...

The `#[error_code]` attribute tells Anchor to generate error codes automatically, and `#[msg()]` sets the error message. This makes debugging much easier!

## Running the Program

To build:

```bash
anchor build
```
