# On-chain swaps between bitcoin and Solana utilizing bitcoin relay

## PTLC (proof-time locked contract)
Contract similar to HTLC (hash-time locked contract), where claimer needs to provide a proof instead of a secret for a hash. In this case the proof is transaction verification through bitcoin relay.



## Solana -> Bitcoin on-chain

### Parties
- **payer** - the one paying in solana or spl token and using intermediary to do the swap
- **intermediary** - handling the swap receives solana or spl token and sends on-chain bitcoin
- **payee** - recipient of the bitcoin on-chain payment

### Process
1. **Payer** queries the **intermediary** off-chain, sending the payee's bitcoin address and an amount he wishes to send, **intermediary** returns the network fee along with his swap fee needed for the swap
2. **Payer** reviews the returned fees and sends a transaction to construct a PTLC on Solana (this also increments a _nonce N_ in the contract state):
	- paying the funds to **intermediary** if he can prove that he sent a pre-agreed amount to payee's address in a bitcoin transaction (tagged with _nonce N_ to prevent replay attacks) that has >=6 confirmations
	- refunding the **payer**, but only after _locktime T_
	- refunding the **payer**, but only with a specific message _Mr (refund)_ signed by **intermediary** (for co-operative close, when payment fails for any reason)

	**NOTE:** If **payer** tries to send same amount to same address twice, even though bitcoin addresses should not be reused this can't be avoided as **payee's** wallet implementation is out of our influence, in this case **intermediary** could use the same transaction proof to claim both of the PTLCs and paying bitcoin transaction just once. To prevent this, it is required that each transaction is tagged with the 7-byte _nonce N_ with least significant 3 bytes prefixed with 0xFF being written as _nSequence_ field for ALL inputs and rest - 4 most significant bytes being treated as integer, adding 500,000,000 to that integer and writing it as _locktime_ field for the transaction.

3. **Intermediary** observes the creation of PTLC on Solana and proceeds to send a bitcoin transaction.

##### Successful payment
4. Transaction confirms on bitcoin chain (has >=6 confirmations) **intermediary** proves this via bitcoin relay and claims the funds from the PTLC

##### Failed payment
4. The payment was unsuccessful, maybe **intermediary** ran out of funds in the meantime, or **intermediary** thinks it's not possible to safely send the transaction with pre-agreed fee and have it confirm in under _locktime T_.
5. Upon request by **payer**, **intermediary** creates a specific signed message _Mr (refund)_, allowing the **payer** to refund his funds from the PTLC (cooperative close)

##### Intermediary went offline
4. **Payer** waits till the expiry of _locktime T_ and then refunds his funds back from the PTLC



## Bitcoin on-chain -> Solana

### Requirements
- payee needs to claim the transaction manually, once it confirms on bitcoin chain

### Parties
- **payee** - recipient of the solana/spl token, using intermediary to do the swap
- **intermediary** - handling the swap, sends solana or spl token and receives bitcoin on-chain payment
- **payer** - the one paying bitcoin on-chain

### Process
1. **Payee** generates a _secret S_, a _payment hash P_ that is produced by _hash of secret H(S)_ and a _secp256k1 keypair for signing bitcoin transactions Kp_
2. **Payee** queries the **intermediary** off-chain, with publickey of his _keypair Kp_, _payment hash P_ and _locktime T1_ he wishes to use, **intermediary** returns his swap fee, min/max receivable along with **intermediary's** _public key Ki_ and generated _bitcoin address A_ which is an HTLC such that:
	- paying the funds to **intermediary** if he can supply a valid _secret S_, such that _hash of secret H(S)_ is equal to _payment hash P_
	- refunding the **payee**, but only after _locktime T1_
	- refunding the **payee**, but this can only be intiated with signatures from both **payee** and **intermediary** (cooperative close)

3. **Payer** sends an on-chain payment to _bitcoin address A_
4. **Payee** observes the on-chain payment by **payer** and waits till it has enough confirmations.
5. **Payee** queries the **intermediary** off-chain to obtain a specific message _Mi (initialize)_ signed by **intermediary** allowing **payee** to create an HTLC on Solana with funds pulled from **intermediary's** vault, an HTLC is constructed:
	- paying the funds to **payee** if he can supply a valid _secret S_, such that _hash of secret H(S)_ is equal to _payment hash P_, but only until a specific time in the future - _locktime T2_
	- refunding the **intermediary**, but only after _locktime T2_

    **NOTE:** _locktime T2_ is determined by **intermediary** strictly lower than _locktime T1_, since **intermediary** needs to have a knowledge of _secret S_ and his sweep transaction on bitcoin needs to confirm before _locktime T1_.

##### Successful payment
6. Upon confirmation of HTLC creation's transaction on Solana, **payee** submits a second transaction revealing the _secret S_ and claiming the funds from HTLC
7. **Intermediary** observes this transaction on Solana and uses the revealed _secret S_ to sweep his funds from the HTLC on bitcoin.

##### Payee went offline
6. **Intermediary** waits till the expiry of _locktime T2_ and then refunds his funds back from the HTLC

##### Intermediary went offline
6. **Payee** waits till the expiry of _locktime T1_ and can then either retry the swap with different intermediary, or send the bitcoin back to sender (all the while paying a bitcoin network fee)



## Bitcoin on-chain -> Solana (discontinued)

### Requirements
- an amount of bitcoin to receive must be known upfront
- payee needs to be online till bitcoin transaction confirms otherwise he may lose funds

### Parties
- **payee** - recipient of the solana/spl token, using intermediary to do the swap
- **intermediary** - handling the swap, sends solana or spl token and receives bitcoin on-chain payment
- **payer** - the one paying bitcoin on-chain

### Process
1. **Payee** queries the **intermediary** off-chain, with an amount he wishes the receive and _locktime T_ he wishes to use, **intermediary** returns his swap fee needed for a swap along with a specific message _Mi (initialize)_ signed by **intermediary** allowing payee to create a PTLC on Solana with funds pulled from **intermediary's** vault. **Intermediary** can also charge a non-refundable fee based on _locktime T_ to disincetivize spamming.
2. **Payee** reviews the returned fee and sends a transaction creating PTLC on Solana using message _Mi (initialize)_ signed by **intermediary** to pull funds from his vault:
	- paying the funds to **payee** if he can prove that a pre-agreed amount was sent to **intermediary's** address in a bitcoin transaction that has >=6 confirmations
	- refunding the **intermediary**, but only after _locktime T_
	- refunding the **intermediary**, but this can only be intiated by **payee**, (for co-operative close, when payee wishes to cancel the payment), this way **payee** can get part of his non-refundable fee back

	**NOTE:** Here the replay protection needs to be handled by the **intermediary** who needs to make sure to generate a new address for every swap (if he doesn't, he is the one losing money), as we cannot possibly influence fields such as _nSequence_ and _locktime_ in bitcoin transaction sent by payer because his wallet implementation is outside our influence.

##### Successful payment
3. Transaction confirms on bitcoin chain (has >=6 confirmations) **payee** proves this via bitcoin relay and claims the funds from the PTLC

##### Failed payment
3. No payment arrived in **intermediary's** wallet address until the _locktime T_, therefore **intermediary** can refund his funds back from the PTLC, keeping the non-refundable fee

##### Payment cancelled
3. **Payee** is sure that no bitcoin payment will come and wishes to cancel the payment (there is no going back from there), he closes the PTLC, refunding the funds back to **intermediary** while keeping part of the non-refundable fee
