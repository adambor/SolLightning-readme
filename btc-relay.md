# Bitcoin relay

This project is heavily inspired by:
- https://github.com/crossclaim/btcrelay-sol
- https://github.com/ethereum/btcrelay
Bitcoin relay is an on-chain program used to verify and store bitcoin blockheaders. This program is completely permissionless and trustless, anyone can write blockheaders as their validity is verified on-chain.

## Verification
Program checks blockheader consensus rules:
- correct PoW difficulty target
- possible difficulty adjustments
- previous block hash
- blockhash is lower than target (block's PoW)
- timestamp is > median of last 11 blocks
- timestamp is < current time + 4 hours

## Storage
To save on storage costs, the blockheader data is emitted as an Event from the program, and only sha256 fingerprint of that blockheader data is stored on-chain.
Another storage costs saving mechanism used is pruning - only last X block headers are kept stored on-chain in a ring buffer. Where X is the pruning factor.

## Transaction verification
As merkle roots of the bitcoin blocks from blockheaders are known, they can be used to verify that any transaction was included in a block by its transaction id and merkle proof. However due to pruning, this also means that transaction verification can only be done for transactions confirmed in the last X blocks. Where X is the pruning factor.

## Forks
Should a fork on the bitcoin main chain occur, the program provides a way for anyone to submit fork blockheaders, and they automatically become the main chain when their chain work is greater than that of a current main chain in the bitcoin relay program.
This can be done in 2 ways, because of solana's \~1.2kB transaction size limitation:
- smaller forks of <6 blocks can be submitted in a single transaction
- larger forks of >=6 blocks must be submitted in multiple transactions by opening a new account storing the data

## Possible attack vectors
### Fake block headers
A party might start submitting valid bitcoin blockheaders to the bitcoin relay and not on the bitcoin main chain. However as those blockheaders must be valid a non-trivial amount of resources must be expedited on PoW. Cost of such an attack depends on whether there is at least 1 honest party submitting blockheaders to the relay:
- if there are no honest parties, the cost of faking 1 blockheader can be expressed as value of lost bitcoin block reward incurred due to not submitting the blockheader to the main chain, which is currently 6.25 BTC ~ 140k usd
- if there is at least 1 honest party, the adversary party needs to capture at least 51% of the bitcoin hashing power to overrun the honest chain submitted by honest party
