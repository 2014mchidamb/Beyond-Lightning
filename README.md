# Beyond-Lightning

**Project Team:** Muthu Chidambaram

**Project Goal:** Understanding the Lightning Network proposal as well as hypothesizing improvements or alternatives to parts (or all) of the proposal.

## Overview of the Lightning Network

**Main Idea:** Transactions that take place on the blockchain are inefficient (have to wait for confirmations). Thus, the idea is to conduct transactions using off-the-blockchain micropayment channels and to only broadcast transactions once a transaction consensus has been reached between the parties involved.

### Micropayment Channels

Consider two parties, Alice and Bob. In order to transact with one another off of the blockchain, Alice and Bob create a 2-of-2 multisig address in which they plan on pooling funds. Furthermore, Alice and Bob also create refund transactions with a specified nLock time from the address (this just means the refund cannot be accessed until after that amount of time), so that they can redeem their funds in the case that no consensus is reached in the set timespan.

#### One-Directional Channel

Without loss of generality, assume that Alice is paying Bob. In a one-directional payment channel, Alice sends some amount of bitcoin to the multisig address after she has generated a refund transaction to herself from the multisig address. Now, Alice can generate and sign transactions from the multisig address that partition the amount of bitcoin in the address between herself and Bob. Alice can generate as many transactions as she wants during the set time period, with each transaction replacing the previous transaction's balance. When Bob is satisfied with the balance, he can sign the transaction and then broadcast it to the blockchain (since it now has both Alice and Bob's signatures). 

To understand this, consider the following example: Alice sends 1 BTC to the multisig address and then signs a transaction sending 0.9 BTC back to herself and 0.1 BTC to Bob. Bob does not sign this transaction. Alice then creates a new transaction sending 0.8 BTC back to herself and 0.2 BTC to Bob, which effectively replaces the old transaction. Bob then chooses to sign this transaction, and then broadcasts it to the blockchain. If Bob does not sign any of the transactions generated by Alice during the agreed upon time span (nLock time for the refund), Alice can redeem her refund transaction.

#### Bidirectional Channel

The aforementioned scheme only allows Alice to send money to Bob. In order for Bob to be able to send money to Alice as well, we introduce an nLock time to the transactions generated from the multisig address. That is, we start an initial transaction with an nLock time that is decremented every time the direction of the transaction is changed.

This can be illustrated by the following example: Alice creates and signs a transaction from the shared multisig address that sends 0.8 BTC back to herself and 0.2 BTC to Bob, with some specified nLock time X. Bob wishes to send 0.1 BTC to Alice, so he creates and signs a new transaction from the multisig address that sends 0.9 BTC to Alice and 0.1 BTC to himself with an nLock time of X-1. Due to the decremented nLock time, Alice can sign this transaction before Bob goes back on his word and signs the previous transaction that sent 0.8 BTC to Alice and 0.2 BTC to Bob.

#### Bidirectional Channels with Revocable Transactions

Suppose we wished to create bidirectional channels in a way that would penalize parties attempting to broadcast old transactions - how would we do this? The LN solution is to use a symmetric pair of commitment transactions stemming from a mutual funding transaction. This basically means that both Alice and Bob contribute to a funding transaction and then obtain refund/commitment transactions signed by the other party. Updating the commitment transactions thus requires both Alice and Bob to sign new commitment transactions containing the updated balance for the other party.

Since Alice and Bob both have their own commitment transactions derived from the funding transaction, they both have the ability to broadcast their respective commitment transactions at any time and end the channel. However, each commitment transaction is encumbered with a time lock so that the owner only receivers his or her respective output after a certain delay. In other words, if Alice chooses to broadcast her commitment transaction, Bob can spend his output immediately, but Alice must wait some number of days (or rather, some number of blocks) before spending hers. Furthermore, all commitment transactions are signed using "throwaway" private keys, so that the keys can be released once the commitment transactions are updated. The combination of the released private keys and the time-locked outputs allows for one party to obtain control of all of the funds should the other party attempt to broadcast an old commitment transaction.

### Hashed Timelocked Contracts (HTLC's)

Bidirectional payment channels only allow transactions between two parties. Ideally, we would like to build a network of these payment channels to allow any number of parties to interact with one another without having to open up new payment channels. The LN approach to building this network is to employ hashed timelocked contracts (HTLC's).

What are HTLC's? I believe they are best illustrated through example. Suppose Alice wants to pay Bob 1 BTC, but does not want to open up a new payment channel with Bob. Alice and Bob both already have payment channels open with Chris - thus, Alice would like to somehow send the 1 BTC payment to Bob using Chris as an intermediary. 

To accomplish this, Bob sends Alice a hash. Alice then promises to pay Chris 1 BTC if he gives her the pre-image of the hash within a set time period of N days. If Chris does not, Alice can redeem a refund transaction so that she does not lose her 1 BTC. Similarly, Chris promises to pay Bob 1 BTC if Bob gives Chris the pre-image of the hash within a time period of N-1 days. The time period for Chris is N-1 days as opposed to N days to prevent the scenario in which Alice redeems her 1 BTC before Chris, leaving Chris with a loss of 1 BTC due to Bob's inactivity. Chris also generates a refund transaction in case Bob becomes unresponsive.

As can be seen pretty easily, this model can be generalized by continuing to use decrementing timelocks between parties. In essence, HTLC's are payments contingent on the revelation of a specified pre-image within a set period of time. By embedding HTLC's within payment channels, transactions can be conducted through a network without having to repeatedly hit the blockchain.

## Problems Facing the Lightning Network

In theory, the Lightning Network seems like a strong choice for the future direction of bitcoin's infrastructure as it continues to grow more popular. This, of course, leads to the following question: why hasn't anyone built it yet?

### Implementation Details

Unsurprisingly, the answer to the previous question is that people are building it - carefully. The Lightning Network is a system that depends on a large number of components to work together smoothly in order to operate as envisioned. Thus, early mistakes in the engineering process could have costly ramifications in the future, especially if and when the network attempts to take on a much higher transaction load.

Rusty Russell, a core tech engineer at Blockstream and developer actively working on implementing the Lightning Network, discussed his main concerns with developing the network in a Reddit thread on potential issues with the network. The following is a brief summary of his key points:

+ Protocol decisions need to be made cautiously, as undoing them in the future may prove difficult once the network is in use.
+ There is the potential of losing privacy and moving towards centralization if the network is not engineered in a way that is as trustless as possible, since many LN users will likely be sending their own transactions and connecting with arbitrary nodes.
+ Micropayments may prove to be more sensitive to traffic analysis than large transactions, which would negatively impact anonymity.
+ Payments may move across millions of nodes prior to reaching their respective destinations, which is a significant routing problem since this needs to be done while revealing as little information as possible.

### Transaction Malleability

Another term that appears to be ubiquitous in discussions of issues with the Lightning Network is transaction malleability. In simple terms, transaction malleability refers to the idea that minor modifications to a bitcoin transaction's data can lead to an identical transaction with a different hash. 

Why is this a big deal? Transaction malleability becomes a serious problem when the transaction hash is used as a unique identifier. For example, consider payment channels in the Lightning Network. Payment channels rely on the integrity of multiple unconfirmed transactions, with transactions building on top of one another. Thus, if some transaction were to become compromised in the chain of transactions, the transactions using that compromised transaction would also become invalidated. 

## References
[BIP 112](https://github.com/bitcoin/bips/blob/master/bip-0112.mediawiki)

[Lightning Network Issues Discussion](https://www.reddit.com/r/Bitcoin/comments/3tucne/eli19_what_are_the_issues_with_the_lightning/)

[Deployable Lightning](https://github.com/ElementsProject/lightning/blob/master/doc/deployable-lightning.pdf)

[The Bitcoin Lightning Network](https://docs.google.com/viewer?url=https%3A%2F%2Flightning.network%2Flightning-network-paper.pdf)

[Segregated Witness Discussion](https://www.reddit.com/r/Bitcoin/comments/3th0py/sipa_proposes_a_fork_of_mainnet_enabling/cx6cunn)
