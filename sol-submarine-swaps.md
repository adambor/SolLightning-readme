# Submarine swaps between bitcoin lightning network and Solana

Submarine swaps are based on a fact that in order to settle a bitcoin lightning network invoice, the receiving party must reveal a secret pre-image S with hash H(S) equal to payment hash P specified in the invoice. This fact can be used to create a similar HTLCs (hash time locked contracts) with same hash P on other chains to depend on sending/receiving of lightning network payment.

## Locktime
A locktime for atomic swaps needs to account for several things:
- as lightning invoices have expiries determined in terms of bitcoin blockheight, it needs to account for cases when bitcoin blocks take a bit longer to mine due to bad luck
- Solana cluster might be down for some time, resulting in a skew of on-chain time
- Intermediary or other party might be down for some time

All this results in a locktime being much larger than actually needed (for most of the times), in turn locking the funds for longer that might be neccessary. To solve this, **bitcoin relay** can be utilized, and locktime expressed in terms of bitcoin blockheight, this way the HTLC on Solana and lightning invoice would use the same time-chain, allowing for much tighter tolerances and shorter locktimes.

## Solana -> Bitcoin lightning

### Requirements
- lightning invoice's payment hash P needs to be known upfront
- lightning invoice has to have a fixed amount
- payee needs to be online at the time of payment

### Parties
- **payer** - the one paying in solana or spl token and using intermediary to do the swap
- **intermediary** - handling the swap receives solana or spl token and sends lightning network payment
- **payee** - recipient of the lightning network payment

### Process
1. **Payee** creates a regular bitcoin lightning invoice with fixed amount and sends it to the **payer** (this invoice contains the payment hash P)
2. **Payer** queries the **intermediary** off-chain, sending the lightning invoice, **intermediary** tries to probe for the route of the payment and returns its confidence score (how likely **intermediary** thinks that the payment will succeed) along with its fee and details the **payer** needs to construct a HTLC on-chain
3. **Payer** reviews the returned confidence score + fee and sends a transaction to construct an HTLC on Solana:
	- paying the funds to **intermediary** if he can supply a valid secret S, such that hash of secret H(S) is equal to payment hash P, but only until a specific time in the future - locktime T
	- refunding the **payer**, but only after locktime T
	- refunding the **payer**, but only with a specific message Mr (refund) signed by **intermediary** (for co-operative close, when payment fails)
4. **Intermediary** observes the creation of HTLC on Solana and proceeds to attempt a payment of the lightning invoice.

##### Successful payment
5. **Payee** reveals a secret S to intermediary in order to accept the payment of the lighting invoice.
6. **Intermediary** uses the knowledge of secret S to obtain the funds from the HTLC on Solana and swap is finished.

##### Failed payment
5. The payment was unsuccessful, so **payee** did not reveal a secret S to the **intermediary**.
6. Upon request by **payer**, **intermediary** creates a specific signed message Mr (refund), allowing the **payer** to refund his funds from the HTLC

##### Intermediary went offline
5. **Payer** waits till the expiry of locktime T and then refunds his funds back from the HTLC

## Bitcoin lightning -> Solana

### Requirements
- lightning invoice has to have a fixed amount

### Parties
- **payee** - recipient of the solana/spl token, using intermediary to do the swap
- **intermediary** - handling the swap, sends solana or spl token and receives lightning network payment
- **payer** - the one paying on bitcoin lightning network

### Process
1. **Payee** creates a secret S and payment hash P that is produced by hash of secret H(S)
2. **Payee** queries the **intermediary** off-chain, with payment hash P and an amount he wishes the receive, **intermediary** creates a bitcoin lightning invoice using payment hash P, with the amount specified and returns it to **payee**
3. **Payee** sends this lightning invoice to the **payer**
4. **Intermediary** receives an incoming lightning network payment from **payer**, but cannot settle it because **intermediary** doesn't know secret S yet.
5. **Payee** queries the **intermediary** off-chain to obtain a specific message Mi (initialize) signed by **intermediary** allowing payee to create an HTLC on Solana with funds pulled from **intermediary's** vault, an HTLC is constructed:
	- paying the funds to **payee** if he can supply a valid secret S, such that hash of secret H(S) is equal to payment hash P, but only until a specific time in the future - locktime T
	- refunding the **payer**, but only after locktime T

##### Successful payment
6. Upon confirmation of HTLC creation's transaction on Solana, **payee** submits a second transaction revealing the secret S and claiming the funds from HTLC
7. **Intermediary** observes this transaction on Solana and uses the revealed secret S to settle the lightning network payment.

##### Payee went offline
6. **Intermediary** waits till the expiry of locktime T and then refunds his funds back from the HTLC
