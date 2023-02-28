# SolBridge

## Problems
- bitcoin is almost always left out of all the cross-chain bridges (as they are mostly focused on EVM chains), even though it has attained the highest liquidity and it has an immense network effect
- few existing solution includes mainly trusted bridges or solutions like ren, relying on incentivised decentralized quorum of nodes

## Technology
### Bitcoin Relay program
- works by storing bitcoin block headers on-chain, with a program that verifies the blockheader consensus rules (this is easy for PoW blockchains)
- this is completely trustless, permisionless (anyone can submit blockheaders) and relies on PoW to guarantee security
- cost to fake the 6 consecutive block (required confirmations) would be 6.25btc\*6 (37.5 btc=881,250 usd) in the worst case

### Submarine swaps
- an atomic swap between bitcoin lightning and Solana on-chain
- exploits the property that lightning network invoice requires the recipient to reveal a pre-image for the payment to execute
- improves on a security of regular atomic swaps by locking the the swap not till a specific timestamp but till a **Bitcoin Relay** conract reaches the specific blockheight

### On-chain swap program
Creates an atomic swap like contract, where:
- expiry is denoted in a bitcoin blockheight
- transaction proof is not a pre-image to a hash but instead a proof that the transaction was included in a bitcoin block that was mined

### Lightning swap program
Creates an atomic swap like contract, where:
- expiry is denoted in a bitcoin blockheight
- transaction proof is the pre-image of the hash of the lightning invoice

## Why Solana?
- Solana enables super-quick confirmation of transactions, making the atomic swaps on-chain super quick and cheap
- For bitcoin relay solana offers cheap storage and low fees for storing the data on-chain (0.0125 cents per stored header, compared to 25 cents on eth), decreasing daily running costs from 36 usd (eth) to 0.02 usd

## Solution
- an SDK for wallet developers, allowing any wallet to trustlessly receive and send bitcoin (on-chain and lightning)

## Impact
- with the advent of Bitcoin lightning network, which is extensible and could serve as a common payment protocol not just limited to Bitcoin, we think providing a bridge between Solana and such a payment protocol will add a lot of value on both ends, acting as a global decentralized chain-agnostic payment protocol

## Use-cases/composability
- solana wallets can receive/send bitcoin lightning and on-chain payments directly
- solana programs can send bitcoin lightning and on-chain payments initiated with a cross-program invocation from the program

## What's next?
- incentive system for people to participate in bitcoin relay
- have a way to automatically choose intermediary based on it's liquidity and reputation in either on-chain or off-chain registry
- achieve higher capital efficiency for intermediaries by investing the funds locked in contract to DeFi, similar to how curve.fi operates on Ethereum

## Business model
- taking a small fee from every transaction and saving it in a treasury
or
- sponsored by a company using the project (Hopa B.V.)
