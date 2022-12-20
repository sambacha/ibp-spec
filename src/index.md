---
title: "Transaction Inclusion Protocol"
subtitle: "InclusionBond Protocol Draft"
author: "@sambacha"
author-url: 'https://github.com/sambacha'
date: "Tue Dec 20 23:47:20 UTC 20220"
return-url: '..'
return-text: '← Return home'
---

:::{.note .blue}
------------------------------------------------------------------------
ℹ️ Outline

[→ More information](https://github.com/jez/pandoc-markdown-css-theme/#license)
------------------------------------------------------------------------

:::

# Abstract



:::{.note .yellow}
------------------------------------------------------------------------
👋 Men at Work

[→ More information](#)
------------------------------------------------------------------------

:::

## InclusionBond Design

InclusionBond takes a different approach -- it incentives the correct execution rather than enforcing it. We set a hedging contract between a single block builder and the transaction issuer. This enables InclusionBond to have only a relatively-low overhead -- there is no need for elaborate proofs of misbehavior.[^dragons]

[^dragons]:
  {-} The boundaries of this 'rational behavior implicitly assume no agreements are made outside this context, which is not, in a production sense, a safe bet. 

Our implementation follows the smart contract wallet (SCW)  design pattern. This design enables customizing the retrieval of the contract tokens, which, for InclusionBond, is done only through the Settle, Exhaust, and Recoup functions. Additionally, this design enables decoupling the transaction issuer (i.e., the party that pays the transaction fees) from the transaction signer (the party that creates the transaction). This, in turn, enables having one party, Seller, use her gas allocation (or pay the transaction fees) to confirm a transaction by the other party Buyer, using so-called meta transactions

Our implementation is compatible with EIP1559 since `txpayload` is a meta transaction, and the transaction by Seller that invokes the Settle is the one that pays the required base fee.


```{.numberLines}
Buyer
Seller
Underwriter
```

## Phases of Operation 

InclusionBond operates in three phases. 

At the setup phase, the gas-purchasing user (the Buyer) sets
the contract parameters, namely:

- the target confirmation interval, 
- the required gas,
- payment details. 

Buyer also locks the payment upfront, and then the gas-selling block builder (the Seller) can accept the contract by depositing a collateral of her own. At this point Buyer does not commit to a future transaction.

At the target execution phase, Buyer publishes a (zero fee) transaction of her choice, which Seller can then confirm through a designated Settle function. InclusionBond verifies the provided transaction was indeed created by Buyer, and if it executes successfully, it sends the funds to Seller.

But what happens if Buyer publish a transaction that exceeds the agreed-upon gas amount, or a transaction that fails, or maybe does not even publish a transaction at all? For that there exists an alternative function, exhaust, that enables Seller to extract the funds without any corporation from Buyer. However, to prevent Seller abusing this function, its invocation spends gas equal to the agreed-upon amount through repeated null operations. This construction results with the Settle function being preferred in case Buyer abides by the contract, while protecting Seller if not.

To analyze InclusionBond we first consider what are the possible interactions Buyer and Seller can have with it – who can do what, and when. These, in turn, give rise to a game, which we analyze using the `subgame` perfect equilibrium solution concept. Our analysis confirms that fulfilling the contract as intended is the `subgame` perfect equilibrium for a wide range of practical parameters.

Consider two system participants, Buyer and Seller, with the following interests: Buyer requires $g_{\text {alloc }}$ gas allocated to a transaction of her choice in future blocks; Seller has a gas allocation of $g_{\text {alloc }}$ in such a suitable block, which she can sell for tokens.

We denote the transaction that Buyer wants to be included by $t x_{\text {payload }}$, and the relevant block interval for its inclusion

by $\left[b_{\text {start }}, b_{\text {end }}\right]$. Note that the content of $t x_{\text {payload }}$ is not necessarily known up to $b_{\text {start }}$. We also denote the block interval in which Buyer wishes to assure the future allocation by $\left[b_{\text {init }}, b_{a c c}\right]$ such that init $\leq$ acc $<$ start $\leq$ end.

The contract instance, the required gas for the future transaction, and a required collateral to be deposited by Seller. She also deposits the token payment for the future transaction confirmation. Following its initiation, the contract starts an acceptance block countdown, during which a Seller can accept it using a transaction. Additionally, accepting the contract requires Seller to deposit tokens as a collateral matching the collateral parameter. The collateral is returned conditioned on Seller further interacting with the contract. Either if Seller accepted the contract, or if the acceptance countdown is completed, the contract accepts no further interactions until the exec phase.

Towards or even during the exec phase, Buyer can publish txpayload. This allows Seller to Settle it, executing txpayload, and getting the payment and collateral tokens from the contract. This is the main functionality of InclusionBond – enabling Seller to execute a transaction provided by Buyer. Alternatively, Seller can exhaust the contract, consuming the hedged gas on null operations, and then receiving its tokens. The motivation for this functionality is to enable Seller to claim the tokens, regardless if Buyer provides a transaction or not; this protects Seller from a faulty or malicious Buyer. However, the naive solution of letting Seller report Buyer as faulty is not sufficient: It allows a Seller to falsely accuse a correct Buyer, getting the contract tokens without providing the confirmation service. By making Seller waste equivalent gas, we remove her incentive to 
do so. If Seller has not accepted the contract, then Buyer can recoup the contract tokens using a transaction.

> Note: I hate the name Inclusion Bond, as these are no 'bonds' but fuck it.

### Settle 

Settle. The Settle function  implements the main functionally of InclusionBond: Seller executing a transaction $t x_{\text {provided }}$ provided by Buyer, and receiving the agreed-upon payment for doing so.

This function takes as an input a transaction $t x_{\text {provided }}$, and first verifies $t x_{\text {provided }}$ was issued by Buyer (line 20). Then, it verifies the invocation is within $\left[b_{\text {start }}, b_{\text {end }}\right]$ (line 21), that Seller had previously accepted (line 22), and that the invocation is by Seller (line 23). The contract then executes the operations of $t x_{\text {provided }}$ as a subroutine (line 24), marks its status completed (line 32), and sends payment $+\varepsilon+$ col tokens to Seller (line 33).
Considering all operations except the execution of $t x_{\text {provided }}$, the Settle function performs similar operations to those of Recoup. We therefore consider its gas consumption, aside from execution of $t x_{\text {provided }}$, is also $g_{\text {done }}$.

### Exhaust

The Exhaust function allows Seller to get payment+col tokens for expending $g_{\text {alloc }}$ gas during the required block interval. Its goal is to protect Seller from a spiteful Buyer, specifically from the case where Buyer does not publish a $t x_{\text {provided }}$ transaction, or publishes ones that consume more than $g_{\text {alloc }}$ gas. When Exhaust is invoked, the contract first verifies the invocation is within $\left[b_{\text {start }}, b_{\text {end }}\right]$ (line 28), that Seller had previously accepted (line 29), and that the invocation is by Seller (line 30).
The contract then performs null operations consuming $g_{\text {alloc }}$ gas (line 31), marks its status completed (line 32), and sends Seller payment $+$ col tokens (line 33).


### Specification

Parameter : acc, start, end, block number operation ranges
Parameter $\quad: g_{\text {alloc }}$, required gas
Parameter $\quad: c o l$, the required collateral by Seller
Parameter : payment, payment for execution
Parameter $\quad: \varepsilon$, additional payment for successful execution.
Global Variable : current, current block number
Variable $\quad$ : status $\leftarrow \perp$, contract status variable
Variable $\quad: P K_{\text {Seller }} \leftarrow \perp$, public identifier of Seller
Variable $\quad: P K_{\text {Buyer }} \leftarrow \perp$, public identifier of Buyer


Function Exhaust (txIssuer, sentTokens):
Assert: start $\leq$ current $\leq$ end
Assert: status $=$ accepted
Assert: $P K_{\text {Seller }}=$ txIssuer
Perform null operations summing to $g_{\text {alloc }}$ gas status $\leftarrow$ completed
Send payment $+$ col to $P K_{\text {Seller }}$


Function Settle (txIssuer, sentTokens; tx provided):
Assert: $t x_{\text {provided }}$ was issued by $PK _{\text {Buyer }}$
Assert: start $\leq$ current $\leq$ end
Assert: status $=$ accepted
Assert: $P K_{\text {Seller }}=$ txIssuer
Execute the operations of $t x_{\text {provided }}$
status $\leftarrow$ completed
Send payment $+\varepsilon+$ col to $P K_{\text {Seller }}$


Function Accept (txIssuer, sentTokens):
Assert: current $\leq$ acc
Assert: status $=$ initiated
Assert: sentTokens $\geq$ col
$P_{\text {Seller }} \leftarrow$ txIssuer
status $\leftarrow$ accepted


Function Initiate (txIssuer, sentTokens; acc, start, end, $g_{\text {alloc }}$, col, $\varepsilon$ ):
Assert: current $\leq$ acc $<$ start $\leq$ end
Assert: $g_{\text {alloc }}>0$, col $\geq 0, \varepsilon \geq 0$, sentTokens $\geq \varepsilon$
Set acc, start, end, $g_{\text {alloc }}$, col from inputs, payment $\leftarrow$ sentTokens $-\varepsilon$ $P K_{\text {Buyer }} \leftarrow$ txIssuer
status $\leftarrow$ initiated


Game state $N o L H$ (in $\varphi_{\text {exec }}$ ) takes place after Nature draws $\pi_{\text {exec }} \sim F$. In this state, Buyer can pay the gas-price $\pi_{\text {exec }}$ o have $t x_{\text {payload }}$ confirmed (action $a_{p u b T x}$ ), incurring the fee cost $g_{\text {alloc }} \cdot \pi_{\text {exec }}$, but have $t x_{\text {payload }}$ confirmed. Alternatively, she



$W_{\text {Seller }}\left(\pi_{\text {exec }}\right.$, PublishTx, $\left.\bar{s}_{\text {spe }}\right)=$
$\begin{cases}W_{\text {Seller }}\left(\pi_{\text {exec }}, \text { FulfillTx, } \bar{s}_{\text {spe }}\right), & \pi_{\text {exec }}<\frac{\text { payment }+\text { col }+\varepsilon}{g_{\text {alloc }}+g_{\text {done }}} \\ W_{\text {Seller }}\left(\pi_{\text {exec }}, \text { FulfillNoTx, } \bar{s}_{\text {spe }}\right), & \pi_{\text {exec }}>\frac{\text { payment }+\text { col }+\varepsilon}{g_{\text {alloc }}+g_{\text {done }}}\end{cases}$
and
$W_{\text {Buyer }}\left(\pi_{\text {exec }}\right.$, PublishTx, $\left.\bar{s}_{\text {spe }}\right)=$
$\left\{\begin{array}{ll}W_{\text {Buyer }}\left(\pi_{\text {exec }}, \text { FulfillTx, } \bar{s}_{\text {spe }}\right), & \pi_{\text {exec }}<\frac{\text { payment }+\text { col }+\varepsilon}{g_{\text {alloc }}+g_{\text {done }}} \\ W_{\text {Buyer }}\left(\pi_{\text {exec }}, \text { FulfillNoTx, } \bar{s}_{\text {spe }}\right), & \pi_{\text {exec }}>\frac{\text { payment }+\text { col }+\varepsilon}{g_{\text {alloc }}+g_{\text {done }}}\end{array}\right.$.


The maximum allowed gap between the epoch of a routing peer and the incoming message. Should be set based on measures the maximum number of epochs that can elapse since a message gets routed from its origin to all the other peers in the network. Can be calculated as:

ms delay + clock / epoch len. 


---

:::{.note .blue}
------------------------------------------------------------------------
ℹ️ Erratta

[→ More information](#)
------------------------------------------------------------------------

:::



# Features
<p class="signoff">
  <a href="/">↑ Back to the top</a>
</p>

