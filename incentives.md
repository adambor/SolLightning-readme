# Proper incentive system for SolLightning

__NOTE:__ This is just a first draft of this document, subject to change and unfinished
## Solana -> BTC & Solana -> BTC-LN
Intermediaries are incentivized to proceed with the payment, because they do get a fee out of it, not acting is not in their economic interest.

### Reputation
Furthemore reputation of intermediaries is tracked on-chain, where anyone can check amount of success, coop closed and failed swaps.


## BTC-LN -> Solana & BTC -> Solana
For swaps from bitcoin lightning and on-chain, the claimer (client) can act uncooperatively, locking up the funds of intermediary for nothing, he still needs to pay the fee for initiating the swap, but that might not be enough to discourage him from DDoS-ing intermediaries and acting un-cooperatively. Furthemore, intermediary then needs to pay for refund transaction.

### Security deposit
We propose the idea of security deposit, that needs to be deposited by claimer (client), and is either returned back to him (upon successful swap) or passed to intermediaries (when client acts uncooperatively). This security deposit is denominated in Solana.

The security deposit is composed of 2-4 parts

##### Base fee - B
Used to cover transaction fees for refund transaction.
E1 - expected fee of the refund TX
B - base fee
B = E1

Example:
B = E1 = 0.000005 Sol = $0.00012

##### Proportional fee - P
Denoted as % of amount, used to cover the opportunity cost of locked funds.
Y - desired yield p.a.
T - locked time in seconds
P - proportional fee
P = ( (1+Y) ^ ( T/(365\*24\*60\*60) ) )-1

Example (LN):
Y = 10%
T = 3600
P = ((1+0.1)^(1/(365\*24)))-1 = 0.001%

Example (On-chain):
Y = 10%
T = 6\*3600
P = ((1+0.1)^(6/(365\*24)))-1 = 0.006%

##### Hedging fee (optional) - X
For swaps between different currencies
TODO - price hedging through options

##### Watchtower fee (BTC -> Solana only) - W
For swaps from on-chain BTC, we utilize watchtowers to claim the swaps on behalf of claimer (client), otherwise claimer might loose funds if he goes offline for a long time, therefore watchtowers need to be well-incentivized for handling these swaps. Absolute minimum is the sum of fees they have to pay to 1. synchronize BTC relay to required blockheight and 2. claim the swap.

H0 - current height of BTC relay
H1 - current height of Bitcoin blockchain
s - safety factor, since average bitcoin blocktime in that short period of time can be <600s
T - locked time in seconds
H2 - required height of BTC relay at the time of claim
H2 = H1 + (T/600)\*s

b - amount of blockheaders needed to be synchronized
b = H2 - H0

S(b) - expected fee for synchronzing _b_ blockheaders
E2 - expected fee for claiming the swap
N - net profit for the watchtower
W - gross fee paid by the claimer
W = S(b) + E2 + N

Example:
H0 = 100,000
H1 = 100,005
s = 2
T = 6\*3600
H2 = 100,005 + (6\*3600/600) \* 2 = 100,077

b = 100,077 - 100,000 = 77

S(b) = 77 * 0.000005 Sol = 0.000385 Sol
E2 = 0.000005 Sol
N = 0.001 Sol
W = 0.000385 + 0.000005 + 0.001 = 0.00139 Sol = $0.03

### Security deposit calculation
D - swap deposit
A - swap amount
D = max(W, B + (X + P) \* A)

Example (LN):
A = 100,000 Sats = $27.29
B = 0.000005 Sol
W = 0
X = 0
P = 0.001%
D = 0.000005 Sol + 0.00001 \* 100,000 Sats = 0.000005 Sol + 1 Sat = 0.000005 Sol + 0.000012 Sol = 0.000017 Sol = $0.000374

Example (On-chain):
A = 1,000,000 Sats = $272.90
B = 0.000005 Sol
W = 0.00139 Sol
X = 0
P = 0.006%
D = max(0.00139 Sol, 0.000005 Sol + 0.00006 \* 1,000,000 Sats) = max(0.00139 Sol, 0.000005 Sol + 60 Sats) = max(0.00139 Sol, 0.000725 Sol) = 0.00139 Sol = $0.031
