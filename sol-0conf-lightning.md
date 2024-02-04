# Trustless lightning on-boarding using 0-conf channels & bitcoin relay

Describes an instant trustless on-boarding to bitcoin lightning network from Solana using 0-conf lightning network channels.

Uses [bitcoin relay](https://github.com/adambor/BTCRelay-Sol) for proofs that a bitcoin transaction was really sent and confirmed on bitcoin blockchain. This works by storing SPV (simplified payment verification) copy of the bitcoin blockchain on Solana, meaning bitcoin blockheaders (with merkle roots) are verified and stored on Solana. Anyone can then prove that he sent a bitcoin transaction with a merkle proof.

## PTLC (proof-time locked contract)
Contract similar to HTLC (hash-time locked contract), where claimer needs to provide a proof instead of a secret for a hash. In this case the proof is transaction verification through [bitcoin relay](https://github.com/adambor/BTCRelay-Sol).

## 0-conf lightning network channels

0-conf channels can be insantly used by the recipient (even before the channel's funding transaction gets confirmed on-chain), this is commonly used by LSPs (lightning service provider) to give recipient on-demand inbound liquidity when he tries to receive a payment larger than his existing inbound liquidity. The main drawback of 0-conf channels is that they are not really trustless and LSP could rug the recipient by double-spending the funding transaction.

### Example

1. User A wishes to pay user B, but B has no existing channel/insufficient inbound liquidity
2. There exists an LSP (lightning service provider) C, who is willing to provide a 0-conf channel to user B
3. User A constructs a lightning network payment route going through C (A->C->B)
4. When C receives the incoming HTLC from A, it pings user B and they negotiate opening of a new lightning channel (C-B)
5. C sends the funding transaction on-chain, but doesn't wait for confirmation - creating a 0-conf channel
6. C adds an HTLC to his channel with user B
7. B reveals the pre-image and settles the HTLC with C, who in turn settles the HTLC with A
8. B now has a new channel with C with balance as received from A

It is important to note that LSP C can at anytime before funding transaction confirms double-spend the transaction and B will loose funds in that case, therefore LSP C needs to be trusted by B till the funding transaction confirms. Nevertheless they are a great tool for quickly on-boarding users on the lightning network.

## Cross-chain 0-conf lightning network channels

If we on-board a user from Solana to lightning using 0-conf channel, we can utilize PTLCs (proof-time locked contracts) with BTC Relay to make the whole process atomic - the pay out to LSP on the Solana side is tied to the lightning network funding transaction confirming.

### Parties 
- **LSP** - handling the swap - receives solana or spl token and creates a 0-conf channel on bitcoin lightning network
- **Client** - wants to be on-boarded into lightning and pays solana or spl token for lightning network sats

### Process

1. **Client** queries the **LSP** off-chain, sending the requested channel parameters (amount, additional inbound liquidity)
2. **LSP** prepares lightning funding transaction (not signed yet!) & commitment transaction (with desired amount of liquidity on the **client**'s side of the channel, and requested inbound liquidity on the **LSP** side), and signed message _Mi (initialize)_ allowing the **client** to create a PTLC on Solana
3. **Client** reviews the returned fees and sends a transaction to construct a PTLC on Solana:
	- paying the funds to **LSP** if he can prove that he sent a lightning network funding transaction (as identified by its tx id), and it was confirmed on bitcoin blockchain 
	- refunding the **client**, but only after _locktime T_
	- refunding the **client**, but only with a specific message _Mr (refund)_ signed by **LSP** (for co-operative close, when swap fails for whatever reason)

##### Success

4. **LSP** observes the creation of PTLC on Solana and proceeds to send a lightning funding transaction.
5. **Client** can instantly use his 0-conf lightning network channel to send lightning network transactions.
5. Lightning funding transaction confirms on bitcoin chain **LSP** proves this via bitcoin relay and claims the funds from the PTLC on Solana

##### LSP double-spends

4. **LSP** observes the creation of PTLC on Solana and proceeds to send the lightning funding transaction.
5. **Client** can instantly use his 0-conf lightning network channel to send lightning network transactions.
6. **LSP** double-spends the lightning funding transaction, invalidates the 0-conf channel, but also forfeits himself the funds from the PTLC on Solana.
7. **Client** waits till the expiry of  _locktime T_ and then refunds his funds back from the PTLC

##### LSP failure
4. **LSP** observes the creation of PTLC, however it decides not to continue with the swap, maybe **LSP** ran out of funds in the meantime, or **LSP** thinks it's not possible to safely send the transaction with pre-agreed fee and have it confirm in under _locktime T_.
5. Upon request by **client**, **LSP** creates a specific signed message _Mr (refund)_, allowing the **client** to refund his funds from the PTLC (cooperative close)

##### LSP went offline
4. **Client** waits till the expiry of _locktime T_ and then refunds his funds back from the PTLC

### Example

1. **Client** requests a 100000 sats bitcoin lightning network channel with additional 50000 sats in inbound liquidity
2. **LSP** prepares a funding transaction, funding the channel with 150000 sats in total, and commitment transaction (this also needs to be signed by the **client**) with 100000 sats assigned to **client**'s output & 50000 sats assigned to **LSP**'s output of the channel. **LSP** also calculates the amount required to be paid by the **client** on Solana (107850 sats = ~50 USDC), this is taking into account:
    - network fee for the funding transaction (50sats/vB * 141vB = 7050 sats)
    - swap fee (100000 sats * 0.3% = 300 sats)
    - sats amount (100000 sats)
    - inbound liquidity lease fee - lease for a year for 1% (50000 * 1% = 500 sats)
3. **Client** reviews the fees and the amount he needs to pay on Solana, then sends a Solana transaction locking up 50 USDC in a PTLC
4. **LSP** observes the creation of PTLC on Solana and sends the lightning funding transaction.
5. **Client** can instantly use his balance of 100000 sats to send lightning network payments
6. **LSP** claims 50 USDC from the PTLC on Solana as soon as the lightning funding transaction confirms on bitcoin blockchain 

