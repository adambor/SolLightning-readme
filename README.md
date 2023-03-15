# SolBridge

## Explainers
- [Bitcoin lightning <-> Solana](https://github.com/adambor/SolLightning-readme/blob/main/sol-submarine-swaps.md)
- [Bitcoin on-chain <-> Solana](https://github.com/adambor/SolLightning-readme/blob/main/sol-onchain-swaps.md)

## Navigation
- [Bitcoin relay on-chain program](https://github.com/adambor/BTCRelay-Sol)
- [Bitcoin relay off-chain app](https://github.com/adambor/BTCRelay-Sol-Offchain)
- [Swap on-chain program](https://github.com/adambor/SolLightning-program)
- [Swap intermediary implementation](https://github.com/adambor/SolLightning-Intermediary)
- [Swap SDK](https://github.com/adambor/SolLightning-sdk)
- [Proof of concept React web-app utilizing Swap SDK](https://github.com/adambor/SolLightning-PoC)

## Testing the PoC
You can access the demo PoC webapp [here](https://sollightning.z6.web.core.windows.net/).
Be sure to switch your wallet to devnet and have some devnet solana in, to cover transaction fees.
Then you can use a lightning network testnet wallet [here](https://htlc.me/) (you will receive some testnet bitcoin when you create a wallet).
First try receiving (btcln -> solana), so you can get some devnet wbtc, and then you can also send (solana -> btcln).
For testing on-chain you can use a wallet [here](https://sereneblue.github.io/blt-wallet)

## What problems are we solving?
- bitcoin is almost always left out of all the cross-chain bridges (as they are mostly focused on EVM chains), even though it has attained the highest liquidity and it has an immense network effect
- few existing solution includes mainly trusted bridges or solutions like ren, relying on incentivised decentralized quorum of nodes

## Technology
### Bitcoin Relay program
- works by storing bitcoin block headers on-chain, with a program that verifies the blockheader consensus rules (this is easy for PoW blockchains)
- this is completely trustless, permisionless (anyone can submit blockheaders) and relies on PoW to guarantee security
- cost to fake the 6 consecutive block (required confirmations) would be 6.25btc\*6 (37.5 btc=881,250 usd) in the worst case
- [on-chain program repo](https://github.com/adambor/BTCRelay-Sol)
- [off-chain synchronizer app repo](https://github.com/adambor/BTCRelay-Sol-Offchain)

### Submarine swaps (lightning network)
- an atomic swap between bitcoin lightning and Solana on-chain
- exploits the property that lightning network invoice requires the recipient to reveal a pre-image for the payment to execute
- can improve on a security of regular atomic swaps by locking the the swap not till a specific timestamp but till a **Bitcoin Relay** conract reaches the specific blockheight

in depth explanation [here](https://github.com/adambor/SolLightning-readme/blob/main/sol-submarine-swaps.md)

### PTLC and atomic swaps (on-chain)
#### PTLC
- transaction proof is not a pre-image to a hash but instead a proof that the transaction was included in a bitcoin block that was mined

in depth explanation [here](https://github.com/adambor/SolLightning-readme/blob/main/sol-onchain-swaps.md)

### Swap program
#### Lightning swaps
Creates an atomic swap like contract, where:
- transaction proof is the pre-image of the hash of the lightning invoice

#### On-chain swaps (Solana -> Bitcoin)
Creates an atomic swap like contract, where:
- transaction proof is not a pre-image to a hash but instead a proof that the transaction was included in a bitcoin block that was mined

#### On-chain swaps (Bitcoin -> Solana)
Creates an atomic swap contract, where:
- transaction proof is the pre-image of the hash needed to unlock the HTLC on bitcoin

[on-chain program repo](https://github.com/adambor/SolLightning-program)

[off-chain intermediary implementation repo](https://github.com/adambor/SolLightning-Intermediary)

## Solution
- an SDK for wallet developers, allowing any wallet to trustlessly receive and send bitcoin (on-chain and lightning)
- [sdk repo](https://github.com/adambor/SolLightning-sdk)

## Why Solana?
- Solana enables super-quick confirmation of transactions, making the atomic swaps on-chain super quick and cheap
- For bitcoin relay solana offers cheap storage and low fees for storing the data on-chain (0.0125 cents per stored header, compared to 25 cents on eth), decreasing daily running costs from 36 usd (eth) to 0.02 usd

## Impact
- with the advent of Bitcoin lightning network, which is extensible and could serve as a common payment protocol not just limited to Bitcoin, we think providing a bridge between Solana and such a payment protocol will add a lot of value on both ends, acting as a global decentralized chain-agnostic payment protocol

## Use-cases/composability
- solana wallets can receive/send bitcoin lightning and on-chain payments directly

## What's next?
- improve upon the bitcoin relay and intermediary implementations, fix edge-cases
- polish the SDK, to be easy to use for wallet or point of sale devs
- incentive system for people to participate in bitcoin relay
- have a way to automatically choose intermediary based on it's liquidity and reputation in either on-chain or off-chain registry
- achieve higher capital efficiency for intermediaries by investing the funds locked in contract to DeFi, similar to how curve.fi operates on Ethereum
- create an API for on-chain programs to be able to initiate bitcoin on-chain and lightning payments with a CPI (cross-program invocation)

## Business model
- taking a small fee (0.1%) from every transaction and saving it in a treasury
