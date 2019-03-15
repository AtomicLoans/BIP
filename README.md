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
    [HASHOP] <secret b2> OP_EQUALVERIFY OP_DUP OP_HASH160 <borrower pubkey hash> OP_EQUALVERIFY OP_CHECKSIG
OP_ELSE
    OP_IF
        <loan expiration num> [TIMEOUTOP] OP_DROP [HASHOP] <secret a2> OP_EQUALVERIFY [HASHOP] <secret b3> OP_EQUALVERIFY OP_2 <borrower pubkey> <lender pubkey> OP_2 OP_CHECKMULTISIG
    OP_ELSE
        <liquidation expiration num> [TIMEOUTOP] OP_DROP OP_DUP OP_HASH160 <borrower pubkey hash> OP_EQUALVERIFY OP_CHECKSIG
    OP_ENDIF
OP_ENDIF
```

The Seizable script takes the following form

```
OP_IF
    [HASHOP] <secret b2> OP_EQUALVERIFY OP_DUP OP_HASH160 <borrower pubkey hash> OP_EQUALVERIFY OP_CHECKSIG
OP_ELSE
    OP_IF
        <loan expiration num> [TIMEOUTOP] OP_DROP [HASHOP] <secret a2> OP_EQUALVERIFY [HASHOP] <secret b3> OP_EQUALVERIFY OP_2 <borrower pubkey> <lender pubkey> OP_2 OP_CHECKMULTISIG
    OP_ELSE
        OP_IF
            <bidding expiration num> [TIMEOUTOP] OP_DROP [HASHOP] <secret a1> OP_EQUALVERIFY OP_DUP OP_HASH160 <lender pubkey hash> OP_EQUALVERIFY OP_CHECKSIG
        OP_ELSE
            <seizure expiration num> [TIMEOUTOP] OP_DROP OP_DUP OP_HASH160 <borrower pubkey hash> OP_EQUALVERIFY OP_CHECKSIG
        OP_ENDIF
    OP_ENDIF
OP_ENDIF
```

[HASHOP] is either OP_SHA256 or OP_HASH160.

[TIMEOUTOP] is either OP_CHECKSEQUENCEVERIFY or OP_CHECKLOCKTIMEVERIFY.

## Interaction ##

- Alice (the "borrower") and Bob (the "lender") exchange public keys as well as two secrets A1, A2 for Alice and B1, B2, B3 for Bob, and mutually agree upon a timeout threshold for the loan period, liquidation period, and seizure period. Both parties exchange then construct the script and P2SH address for the Refundable Collateral Contract and Seizable Collateral Contract. Both parties also construct the script for the blockchain in which the loan will be issued.

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

# Compatibility #

BIP ??? is compatible with [ERC ???](https://github.com/atomicloans/eip) for [atomic loans](https://arxiv.org/pdf/1901.05117.pdf) with Ethereum and other HTLC and smart contract compatible chains.

# Motivation #

In many different protocols, the revealing of secrets is used a settlement mechanism. Hashed Time-Locked Atomic Loan Collateral Contract transactions are a safe way of exchanging secrets to advance the state of a debt agreement, due to the ability to recover a percentage of collateral funds from an uncooperative counterparty, and ensure principal + interest + liquidation fee is paid with a cooperative party. 

# Definitions #

`borrower`: entity that receives loan amount from `lender` once the `borrower` locks their collateral

`lender`: entity that contributes funds to the `borrower` for the debt agreement

`secret`: random number chosen by the `borrower` or `lender`, revealed to allow the parties to change the state of the debt agreement

`secretHash`: hash of the `secret`, used in the construction of Hashed Time-Locked Atomic Loan Contract

`expiration`: timestamp the determines which part of the debt agreement the two parties are in

`SecretA1`: `borrower` secret used for proving that the `borrower` has withdrawn the loan

`SecretA2`: `borrower` secret used for allowing `bidder` to spend the collateral funds

`SecretB1`: `lender` secret used for accepting `borrower` collateral locking, enabling them to withdraw the loan amount

`SecretB2`: `lender` secret used for refunding themselves in the event they aren't satisfied with locking of the collateral or for accepting repayment

`SecretB3`: `lender` secret used for accepting bid process

`SecretC`: `bidder` secret used for accepting the signatures of the `borrower` and `lender` to liquidate the collateral

`loan expiration num`: timestamp that determines when the `borrower` must repay the loan by

`bidding expiration num`: timestamp that determines the amount of time allocated to bidding before seizure period occurs

`seizure expiration num`: timestamp that determines the amount of time allocated to seizing the collateral before borrower can refund

**Withdraw Period:**
During this time, the `lender` deploys the Hashed Time-Locked Atomic Loan Contract. Following this, the borrower locks their collateral on the collateral blockchain in a Hashed Time-Locked Atomic Loan Collateral Contract https://github.com/mattBlackDesign/BIP. The `lender` then either reveals `secretB1` to signify that they are satisfied with the collateral, and the borrower can withdraw the loan using `secretA1`. Otherwise, the lender refunds their loan amount, by revealing `secretB2`, allowing the borrower to refund their Collateral. 

**Loan Period:**
Once the `borrower` has withdrawn the loan amount, the loan begins. Once the loan term is finished, the borrower is expected to repay the loan. If they do, the `lender` can then accept the repayment by revealing `secretB2` enabling the borrower to refund their Collateral. In the case that the `borrower` defaults on the `lender` does not accept the loan repayment, the parties can opt for liquidation of the collateral in the Bidding Period.

**Bidding Period:**
In the case of a default or the `lender` not accepting the `borrower` repayment, the `lender` and `borrower` can opt for liquidation of the collateral through the process of third parties bidding on the collateral. This is done by either the `lender` or `borrower` initiating bidding, which will allow third parties to bid on the collateral. Once the bidding timeout occurs, the `lender` and `borrower` must each provide a signature to allow the collateral to be spent by the `bidder`.

**Seizure Period:**
In the case that bidding is unsuccessful, the `lender` can seize a percentage of the collateral. The amount is dependent on the amount of collateral locked in the seizable script and refundable script as described in the BIP (https://github.com/mattBlackDesign/BIP). During this period the `borrower` can also spend the funds locked in the refundable script.

**Refund Period:**
In the case that the `lender` does not seize the collateral locked in the seizable script, then the borrower can refund the funds locked in the refundable script.

# Implementation #

https://github.com/liquality/chainabstractionlayer/blob/7585c205407529b139d4df187b6a4a001d56fcb1/src/providers/bitcoin/BitcoinCollateralProvider.js
