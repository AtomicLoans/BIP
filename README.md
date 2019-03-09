<pre>
  BIP: ???
  Layer: Applications
  Title: Hashed Time-Locked Atomic Loan Collateral Contract transactions
  Author: Matthew Black &ltmatthewjablack@gmail.com&gt
  Comments-Summary: No comments yet.
  Comments-URI: TBD
  Status: Proposal
  Type: Informational
  Created: 2019-02-26
</pre>

# Abstract #

This BIP describes a script for generalized debt agreement contract based on Hashed Time-Lock Contract (BIP 199) transactions according to the Atomic Loans specification (https://arxiv.org/pdf/1901.05117.pdf)

# Summary #

A Hashed Time-Locked Atomic Loans Collateral Contract consists of two scripts that permit a designated party (the "borrower") to lock funds for a specified amount of time as collateral.

The purpose of each script, is it enables two parties (the "borrower" and the "lender") to create a debt agreement, where the collateral is locked in a P2SH, and can only be spent once the borrower repays the principal + interest in the debt agreement. In the case that the borrower does not repay, the borrower and lender can opt for liquidation of the collateral, which will involved atomically swapping the collateral for the loan currency. In the case that both parties don't opt for liquidation, then each party will be entitled to a percentage of the collateral, decided when the funds are locked in the P2SH.

These funds are locked into two scripts. Refundable and Seizable collateral scripts. The funds sent to these scripts represent the percentage of collateral that each party is entitled to in the case that repayment fails, and the parties don't opt for liquidation.

The Refundable script takes the following form:

```
OP_IF
    [HASHOP] <digest> OP_EQUALVERIFY OP_DUP OP_HASH160 <borrower pubkey hash> OP_EQUALVERIFY OP_CHECKSIG
OP_ELSE
    OP_IF
        <loan expiration num> [TIMEOUTOP] OP_DROP OP_2 <borrower pubkey> <lender pubkey> OP_2 OP_CHECKMULTISIG
    OP_ELSE
        <liquidation expiration num> [TIMEOUTOP] OP_DROP OP_DUP OP_HASH160 <borrower pubkey hash> OP_EQUALVERIFY OP_CHECKSIG
    OP_ENDIF
OP_ENDIF
```

The Seizable script takes the following form

```
OP_IF
    [HASHOP] <digest> OP_EQUALVERIFY OP_DUP OP_HASH160 <borrower pubkey hash> OP_EQUALVERIFY OP_CHECKSIG
OP_ELSE
    OP_IF
        <loan expiration num> [TIMEOUTOP] OP_DROP OP_2 <borrower pubkey> <lender pubkey> OP_2 OP_CHECKMULTISIG
    OP_ELSE
        OP_IF
            <liquidation expiration num> [TIMEOUTOP] OP_DROP [HASHOP] <digest> OP_DUP OP_HASH160 <lender pubkey hash> OP_EQUALVERIFY OP_CHECKSIG
        OP_ELSE
            <seizure expiration num> [TIMEOUTOP] OP_DROP OP_DUP OP_HASH160 <borrower pubkey hash> OP_EQUALVERIFY OP_CHECKSIG
        OP_ENDIF
    OP_ENDIF
OP_ENDIF
```

[HASHOP] is either OP_SHA256 or OP_HASH160.

[TIMEOUTOP] is either OP_CHECKSEQUENCEVERIFY or OP_CHECKLOCKTIMEVERIFY.

## Interaction ##

- Alice (the "borrower") and Bob (the "lender") exchange public keys as well as two secrets A1, A2 for Alice and B1, B2 for Bob, and mutually agree upon a timeout threshold for the loan period, liquidation period, and seizure period. Both parties exchange then construct the script and P2SH address for the Refundable Collateral Contract and Seizable Collateral Contract. Both parties also construct the script for the blockchain in which the loan will be issued.

- Bob sends funds to the loan script on the other blockchain

- Alice sends funds to the Refundable Collateral P2SH address and sends fund to the Seizable Collateral P2SH address.

- Either:
  - Bob accepts locking of collateral by Alice, and reveals B1, allowing Alice to withdraw the loan amount
  - Bob doesn't accept locking of collateral by Alice, and recovers the funds after the timeout threshold while revealing B2, which allows Alice to refund the Refundable and Seizable collateral.

  - If Bob accepts the locking of collateral By alice:

- Either:
  - Alice repays the loan and Bob reveals the preimage to Alice by revealing it in the loan repayment acceptance transaction; OR
  - Alice defaults on the loan and Alice and Bob both opt for collateral liquidation, and a third party Charlie, bids on the collateral, which is transferred in the process of an Atomic Swap in return for the loan currency. This is done by both Alice and Bob signing a multisig and revealing A2 and B2; OR
  - Alice defaults on the loan and Alice or Bob opts out of collateral liquidation, then Alice recovers the Refundable Collateral funds and Bob spends the Seizable Collateral funds.
  - Alice defaults on the loan and Alice or Bob opts out of collateral liquidation and Bob doesn't spend the Seizable Collateral funds, then Alice recovers the Refundable Collateral funds and recovers the Seizable Collateral funds.

Bob is interested in a lower loan timeout to reduce the amount of time that his funds are kept in the contract in the event that Alice defaults.

# Motivation #

In many different protocols, the revealing of secrets is used a settlement mechanism. Hashed Time-Locked Atomic Loan Collateral Contract transactions are a safe way of exchanging secrets to advance the state of a debt agreement, due to the ability to recover a percentage of collateral funds from an uncooperative counterparty, and ensure principal + interest + liquidation fee is paid with a cooperative party. 

# Implementation #

https://github.com/liquality/chainabstractionlayer/blob/7585c205407529b139d4df187b6a4a001d56fcb1/src/providers/bitcoin/BitcoinCollateralProvider.js
