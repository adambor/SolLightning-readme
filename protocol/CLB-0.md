# Smart contract definition (WIP)

This document contains definitions of capabilities of a smart contract to be used for CrossLightning.

## Parties

Swap client - 
Intermediary - 
Relayer -

## Swap types

HTLC
On-Chain
On-Chain + nonce

## 1. Holding funds of the intermediaries on the smart chain
Smart contract needs to be able to custody/hold funds of the intermediaries in a vault. This is needed to minimize the costs of running an intermediary node, as swap clients are able to pull the funds from this vault providing a valid signature from intermediary, paying for transaction themselves.

### 1.1. Deposit function

Smart contract needs to feature a function allowing an intermediary to deposit any token into the smart contract vault

### 1.2. Withdraw function

Smart contract needs to provide a way for intermediaries to pull their funds out of the smart contracts if they so desire.

### 1.3. Balance getter

Balance of each of the intermediary needs to be publicly available in order for the swap clients to be able to check whether a given intermediary has enough liquidity to honor their swap.

## 2. Reputation of the intermediaries

Smart contract needs to track reputation of the intermediaries, the data is tracked separately for each token and also for each swap type. The data that is tracked:

| Name               | Description                                          |
|--------------------|------------------------------------------------------|
| success_volume     | Total volume of successfully processed swaps         |
| success_count      | Total count of successfully processed swaps          |
| failed_volume      | Total volume of failed swaps (un-cooperative closes) |
| failed_count       | Total count of failed swaps (un-cooperative closes)  |
| coop_closed_volume | Total volume of cooperatively closed swaps           |
| coop_closed_count  | Total count of cooperatively closed swaps            |

### 2.1. Reputation getter

Allows anyone to check full reputation (for every token and for every swap type) of the given intermediary

## 3. Swaps

Smart contract needs to be able to handle all swap types. Handling for them differs. A swap needs to feature following fields:

| Name             | Description                                                                                                                                                                            |
|------------------|----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| kind             | Type of the swap                                                                                                                                                                       |
| confirmations    | Required number of block confirmations for On-Chain swaps, otherwise left 0                                                                                                            |
| nonce            | Nonce to use in a transaction for On-Chain + nonce swaps, otherwise left 0                                                                                                             |
| hash             | A payment hash for HTLC or required transaction output hash for On-Chain                                                                                                               |
| pay_in           | Whether this swap was created by depositing funds to the smart contract (true), or by pulling funds from the intermediary (false)                                                      |
| pay_out          | Whether this swap should pay the funds out to the swap client (true), or keep them in a smart contract and assign to an intermediary (false)                                           |
| offerer          | A party offering the funds in a swap contract                                                                                                                                          |
| claimer          | A party trying to claim the funds in a swap contract                                                                                                                                   |
| amount           | Amount of tokens in the swap contract                                                                                                                                                  |
| token            | A token identifier of the funds in the swap contract                                                                                                                                   |
| expiry           | The expiration of the swap contract (offerer can claim the funds back after this timestamp)                                                                                            |
| claimer_bounty   | A bounty paid out to whoever is able to claim this swap contract on behalf of the claimer (used as an incentive for relayers & watchtowers)                                            |
| security_deposit | A deposit paid out to offerer if he needs to refund the swap, otherwise returned to claimer (only for not pay_in swaps, used to disincentivize people from DDoSing intermediary nodes) |

### 3.1. Swap create function

Allows initialization of the swap contracts.

#### 3.1.1. Direction of the swap

The swap creation function differs for the different directions of the swaps (smart chain <-> bitcoin/lightning)

##### 3.1.1.1. Pay in (smart chain -> bitcoin/lightnig)

Requires a valid replay-protected short-lived signature from a swap intermediary that is replay-protected, this is done to prevent people from spamming the node with swap requests that it might not be able to honor (due to liquidity constrains on bitcoin side) which would result in the decrease of intermediary's reputation. In this case the field **pay\_in=true** & **pay\_out=false** and funds need to be deposited by the swap client.

##### 3.1.1.2. Not pay in (bitcoin/lightning -> smart chain)

Requires a valid replay-protected short-lived signature from a swap intermediary, this is done so people cannot pull funds from the intermediary's smart contract vault and DDoS the intermediary this way. In this case the field **pay\_in=false** & **pay\_out=true** and funds are pulled from the intermediary's vault inside of a smart contract. Swap client is required to put up a security deposit, so in the case he decides to not conclude the swap an intermediary will be compensated for the refund transaction fee and lost time-value of his funds, therefore **security\_deposit>0**. A required security deposit is determined by the intermediary node.
