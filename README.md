# atomiq.exchange (formerly SolLightning)
[![Twitter](https://img.shields.io/twitter/url/https/twitter.com/atomiqlabs.svg?style=social&label=Follow%20%40atomiqlabs)](https://twitter.com/atomiqlabs)

A fully trustless DEX (decentralized exchange) protocol between Solana (any SPL token) <-> Bitcoin (on-chain and lightning). Utilizing submarine swaps (for lightning network swaps) and on-chain SPV verification through [bitcoin relay](https://github.com/adambor/BTCRelay-Sol) (for on-chain).

**NOTE:** We are not issuing a new wrapped bitcoin token on Solana, as that would require overcollateralization (and be exposed to exchange rate and oracle risks), which we deem insecure, therefore SolLightning is only acting as a cross-chain DEX (Any SPL token <-> Bitcoin).

Live web-app link: https://app.atomiq.exchange/

## Explainers
- [Bitcoin lightning <-> Solana](https://github.com/adambor/SolLightning-readme/blob/main/sol-submarine-swaps.md)
- [Bitcoin on-chain <-> Solana](https://github.com/adambor/SolLightning-readme/blob/main/sol-onchain-swaps.md)

## Navigation
#### Bitcoin relay
- [Bitcoin relay on-chain program](https://github.com/adambor/BTCRelay-Sol)
- [Bitcoin relay + Watchtower implementation](https://github.com/adambor/BtcRelay-Sol-TS)

#### Swaps
- [Swap on-chain program](https://github.com/adambor/SolLightning-program)
- [Swap intermediary implementation](https://github.com/adambor/SolLightning-Intermediary-TS)
- [Swap SDK](https://github.com/adambor/SolLightning-sdk)

#### Web-app
- [Web-app (hosted on atomiq.exchange)](https://github.com/adambor/SolLightning-dApp-v2)

## Audits

### Ackee Blockchain

Ackee Blockchain a.s. finished audit of our solana programs ([BTC Relay](https://github.com/adambor/BTCRelay-Sol) & [Swap Program](https://github.com/adambor/SolLightning-program)) on 20th of December 2023, with a final re-audit finished on 12th of January 2024. Report can be found [here](https://github.com/adambor/SolLightning-readme/blob/main/audits/ackee-blockchain-sollightning-report.pdf).

## Check it out on devnet
You can access the devnet webapp version: [devnet.atomiq.exchange](https://devnet.atomiq.exchange/).
1. Be sure to switch your wallet to devnet and have some devnet solana in, to cover transaction fees.
2. Use a lightning network testnet wallet [here](https://htlc.me/) (you will receive some testnet bitcoin when you create a wallet).
3. Try receiving (btcln -> solana) - only SOL is supported on devnet, and then you can also send (solana -> btcln).
4. For testing on-chain, you will need to download a bitcoin testnet wallet (unfortunatelly there is no online testnet web wallet).
5. For sending (solana -> btc) you can just try sending to some random testnet address (e.g. mijXVEL3Ko6fuE1p8M42R95ndVhgQGexEo) and then check it on block explorer [here](https://mempool.space/testnet/address/mijXVEL3Ko6fuE1p8M42R95ndVhgQGexEo)

## Motivation
We are allowing Solana wallets to use existing bitcoin payment infrastructure - being able to seamlessly receive and pay with native bitcoin. Using lightning network, which with SolLightning can become a common payment protocol not just limited to Bitcoin and turn into global decentralized chain-agnostic payment protocol - you are able to settle (send/receive) lightning invoices in any SPL token (USDC, USDT, WBTC).

## Technology
### Bitcoin relay program
Works by storing an SPV (simplified payment verification) copy of bitcoin blockchain on-chain, with an on-chain program that verifies the blockheader consensus rules. Anyone can then prove that he really sent a bitcoin transaction (and it got confirmed - included in a block) with just a merkle proof.
More info [here](https://github.com/adambor/BTCRelay-Sol).

### Submarine swaps (lightning network swaps)
Similar to atomic swaps, but exploits the property that lightning network invoice requires the recipient to reveal a pre-image for the payment to confirm. Can improve on security and speed of regular atomic swaps by locking the the swap not till a specific timestamp but till a **Bitcoin Relay** program reaches the specific blockheight, this way both sides use the same time-chain.
More info [here](https://github.com/adambor/SolLightning-readme/blob/main/sol-submarine-swaps.md)

### Proof-time locked contracts (on-chain swaps)
Similar to atomic swaps, but the claimer needs to prove (with merkle proof and **Bitcoin Relay** program) that he sent the desired bitcoin transaction and it confirmed on bitcoin blockchain. Can improve on security and speed of regular atomic swaps by locking the the swap not till a specific timestamp but till a **Bitcoin Relay** program reaches the specific blockheight, this way both sides use the same time-chain.
More info [here](https://github.com/adambor/SolLightning-readme/blob/main/sol-onchain-swaps.md)

## Protocol workings
Parties:
- __Intermediary node__
    - handles the cross-chain swaps
    - needs liquidity on both chains
    - determines prices
    - earns swap fees
    - has reputation (stored on-chain on Solana) - successful, failed, cooperative closed swaps
- __Relayer/Watchtower__
    - submits bitcoin blockheaders to bitcoin relay program and keep it in sync
    - claims the swaps on behalf of clients
    - earns a small fee for every claimed swap
- __Client__
    - intiates the swap

The protocol is backed by __intermediary nodes__ and __relayers/watchtowers__, which can be run by anyone. There is a registry (for now a github repo, but will move on-chain), of every __intermediary node__ in the network.

Swap initialization:
1. __Client__ fetches __intermediary nodes__ from registry, fetches their fees and reputation
2. Based on these metrics the __client__ chooses an __intermediary node__ he wants to use
3. __Client__ sends a request to the desired __intermediary node__ to initiate the swap, if the node is unresponsive, or returns invalid data he blacklists it internally and proceeds to send the same request to next best __intermediary node__.

In case of Bitcoin on-chain to Solana swaps, the __client__ has to wait for certain number of confirmations on his bitcoin transaction till he is able to claim his funds on Solana (which with bitcoin's 10 minutes blocktime can take a long time). If a __client__ were to go offline during that time and not return before expiration of the PTLC (proof-time locked contract) he would loose his funds, therefore __relayers__ double-down also as __watchtowers__.
1. All __relayers/watchtowers__ observe creation of Bitcoin -> Solana swaps on-chain.
2. They do check if the bitcoin transaction corresponding to any of the currently active swaps did get enough confirmations in the blockheader that they are going to submit to bitcoin relay program.
3. If there is a bitcoin transaction that claims the swap they will claim it on behalf of __the user__, while earning a small fee (currently rent exempt amount of lamports from the swap PDA ~ 0.0027 SOL)

## What's next?
- &#9744; improve upon the bitcoin relay and intermediary implementations, fix edge-cases
- &#9744; polish the SDK, to be easy to use for wallet or point of sale devs
- &#9745; incentive system for people to participate in bitcoin relay
- &#9745; have a way to automatically choose intermediary based on it's liquidity and reputation in either on-chain or off-chain registry
- &#9744; achieve higher capital efficiency for intermediaries by investing the funds locked in contract to DeFi, similar to how curve.fi operates on Ethereum
- &#9744; create an API for on-chain programs to be able to initiate bitcoin on-chain and lightning payments with a CPI (cross-program invocation)

## Contact
Via mail: adamborcany(at)gmail(dot)com
