# Submarine swaps between bitcoin lightning network and Solana

Submarine swaps are based on a fact that in order to settle a bitcoin lightning network invoice, the receiving party must reveal a _secret S_ with _hash H(S)_ equal to _payment hash P_ specified in the invoice. This fact can be used to create a similar HTLCs (hash time locked contracts) with same _hash P_ on other chains to depend on sending/receiving of lightning network payment.

## Solana -> Bitcoin lightning

### Requirements
- lightning invoice's payment _hash P_ needs to be known upfront
- lightning invoice has to have a fixed amount
- payee needs to be online at the time of payment

### Parties
- **payer** - the one paying in solana or spl token and using intermediary to do the swap
- **intermediary** - handling the swap receives solana or spl token and sends lightning network payment
- **payee** - recipient of the lightning network payment

### Process
1. **Payee** creates a regular bitcoin lightning invoice with fixed amount and sends it to the **payer** (this invoice contains the payment hash P)
2. **Payer** queries the **intermediary** off-chain, sending the lightning invoice and desired locktime T, **intermediary** tries to probe for the route of the payment and returns its confidence score (how likely **intermediary** thinks that the payment will succeed) along with its fee and details the **payer** needs to construct a HTLC on-chain
3. **Payer** reviews the returned confidence score + fee and sends a transaction to construct an HTLC on Solana:
	- paying the funds to **intermediary** if he can supply a valid _secret S_, such that _hash of secret H(S)_ is equal to _payment hash P_, but only until a specific time in the future - _locktime T_
	- refunding the **payer**, but only after _locktime T_
	- refunding the **payer**, but only with a specific message _Mr (refund)_ signed by **intermediary** (for co-operative close, when payment fails)
	
    **NOTE:** _locktime T_ is determined by the **payer** and is a trade-off between likelihood of payment being successful and locking the funds for shorter periods in case of **intermediary's** non-cooperativeness.
    - Larger locktime means more payment paths can be considered (more hops on the lightning network), increases the likelihood of payment being successfully routed, but will also lead to longer lock periods when **intermediary** is non-cooperative.
    - Smaller locktime means less payment paths can be considered (less hops on the lightning network) and increases the likelihood of payment routing failures, but will also lead to shorter lock periods when **intermediary** is non-cooperative

4. **Intermediary** observes the creation of HTLC on Solana and proceeds to attempt a payment of the lightning invoice.

##### Successful payment
5. **Payee** reveals a _secret S_ to intermediary in order to accept the payment of the lighting invoice.
6. **Intermediary** uses the knowledge of _secret S_ to obtain the funds from the HTLC on Solana and swap is finished.

##### Failed payment
5. The payment was unsuccessful, so **payee** did not reveal a _secret S_ to the **intermediary**.
6. Upon request by **payer**, **intermediary** creates a specific signed message _Mr (refund)_, allowing the **payer** to refund his funds from the HTLC

##### Intermediary went offline
5. **Payer** waits till the expiry of _locktime T_ and then refunds his funds back from the HTLC

## Bitcoin lightning -> Solana

### Requirements
- lightning invoice has to have a fixed amount

### Parties
- **payee** - recipient of the solana/spl token, using intermediary to do the swap
- **intermediary** - handling the swap, sends solana or spl token and receives lightning network payment
- **payer** - the one paying on bitcoin lightning network

### Process
1. **Payee** creates a _secret S_ and _payment hash P_ that is produced by _hash of secret H(S)_
2. **Payee** queries the **intermediary** off-chain, with _payment hash P_ and an amount he wishes the receive, **intermediary** creates a bitcoin lightning invoice using _payment hash P_, with the amount specified and returns it to **payee**
3. **Payee** sends this lightning invoice to the **payer**
4. **Intermediary** receives an incoming lightning network payment from **payer**, but cannot settle it because **intermediary** doesn't know _secret S_ yet.
5. **Payee** queries the **intermediary** off-chain to obtain a specific message _Mi (initialize)_ signed by **intermediary** allowing payee to create an HTLC on Solana with funds pulled from **intermediary's** vault, an HTLC is constructed:
	- paying the funds to **payee** if he can supply a valid _secret S_, such that _hash of secret H(S)_ is equal to _payment hash P_, but only until a specific time in the future - _locktime T_
	- refunding the **payer**, but only after _locktime T_

    **NOTE:** _locktime T_ is determined by **intermediary** based on lightning invoice's _min\_cltv\_delta_ - the minimal timeout delta for last lightning network HTLC in chain (last hop of the lightning network payment) as **intermediary** needs to have a knowledge of _secret S_ before then to successfully receive a payment

##### Successful payment
6. Upon confirmation of HTLC creation's transaction on Solana, **payee** submits a second transaction revealing the _secret S_ and claiming the funds from HTLC
7. **Intermediary** observes this transaction on Solana and uses the revealed _secret S_ to settle the lightning network payment.

##### Payee went offline
6. **Intermediary** waits till the expiry of _locktime T_ and then refunds his funds back from the HTLC

## Locktime
A locktime for atomic swaps needs to account for several things:
- as lightning invoices have expiries determined in terms of bitcoin blockheight, it needs to account for cases when bitcoin blocks take a bit longer to mine due to bad luck
- Solana cluster might be down for some time, resulting in a skew of on-chain time
- Intermediary or other party might be down for some time

All this results in a locktime being much larger than actually needed (for most of the times), in turn locking the funds for longer that might be neccessary. To solve this, **bitcoin relay** can be utilized, and locktime expressed in terms of bitcoin blockheight, this way the HTLC on Solana and lightning invoice would use the same time-chain, allowing for much tighter tolerances and shorter locktimes.
