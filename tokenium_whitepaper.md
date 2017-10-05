# Tokenium Whitepaper

**Version 0.1**

Tokenium is a cryptocurrency exchange protocol for ethereum ERC20 tokens. The protocol describes the interaction of a trusted open source trading client software, the trusted an Tokenium smart contract, and an almost trustless exchange server. The protocol makes possible the creation of efficient high-frequency token exchanges where users need to trust the exchange server only in an extremely limited and well-defined manner. It combines the best of both worlds: Seamless real-time trading of centralized exchanges, and the (almost) trustless manner of decentralized exchange protocols.

## Existing Work

Currently centralized exchanges dominate the cryptocurrency market. These exchanges are very efficient because order-matching is done totally off-chain, but users have to trust these exchanges with their deposited money. This is a huge risk for the user, and also makes deposit/withdrawal processes painful in most cases.

Decentralized exchanges were proposed to solve the risk problem. There are several approaches:

Projects which maintain an order-book on-chain (like EtherDelta) have extremely slow user experience and prone to blockchain congestion and uncertanity because outcomes depend on trasaction order inside a block.

Several improvements have been proposed, some of them:
 * Automatic market maker solutions (e.g. Bancor)
 * Off-chain order placement - on chain *take* and *cancel* (0x)

Both solutions have cons and pros. Automatic market maker solutions seem to be more convenient, but one problem seems to be that users have to be careful when buying/selling bigger amounts, as the calculated price can easily go away from the real market price for tokens.
The main problem with both solutions is that blockchain congestion and transaction order instability is not solved in either of them. More specifically in 0x the success of a *take* or *cancel* command cannot be determined before the block is actually mined because the success of a trnsaction depends on which order the transactions arrive to the ethereum node. So both solutions can face problems with high traffic exchanges.

Another upcoming solution is *Kyber Network*. Kyber Network provides risk-free exchange service for people with fix prices continuously managed by so called *reserve managers*. Reserve managers are professional traders who are supposed to use centralized exchanges to trade, so Kyber Network is not really a competititor, rather a complementary solution to Tokenium.  

## Introduction To Tokenium

In Tokenium order matching happens off-chain in a usual highly optimized order-matching engine that we are used to in centralized exchanges. The protocol ensures that even if the exchange is compromised, its owners or hackers cannot run away with the user's money. All they can do is to not match an order when they could, or match it with not the best possible counter order (although of course within the specified limit!) This presents some limited risk for the user, but this risk cannot be compared to the risk present in case of centralized exchanges, where the whole traded funds can be compromised.

## Actors

There are 4 main actors in the Tokenium protocol.

### User

The person who wants to trade tokens.
The user uses a standalone open source exchange-agnostic client software like the Tokenium Wallet, which the user trusts.

### Exchange Server

A centralized server application.

### Exchange Client

An exchange-server agnostic exchange client. (The Tokenium team develops such an open source exchange client called *Tokenium Wallet*).
In the client the user can select which exchange server she/he wants to connect to. 

### Tokenium Smart Contract

It drives the Tokenium protocol and provides its trust guarantees.

## Tokenium Protocol Details

Below comes a relatively detailed discussion of the protocol, but it is not as detailed as the documentation of the standard that we are working on. This whitepaper is meant to have and understanding of the protocol. 

### Guarantees of the protocol

The 3 main actors in the Tokenium protocol are the **client**, the **server** and the **smart contract**.

* The smart contract is immutable and always behaves correctly.
* The protocol fully protects other clients and the server from a dishonest client.
* The protocol gurantees the following in case of a dishonest server:
	* Only tokens in the *reserve* can be ever traded.
	* The client can request so-called *emergency unreserve* anytime (usually in practice this should only be done when the client detects dishonest or erronous behaviour from the server), and it will be guranteedly done on the blockchain in a defined amount of ethereum steps: untraded reserves guaranteedly go back to the users wallet.
	* Trades will only be done in the limit specified by the client.
	* The client can detect if the server does not cancel the cancelled trade orders on the blockchain and can request an emergency unreserve in that case. It is guaranteed by the smart contract that trade orders cancelled on the blockchain can no longer be used.

The playground for making some loss for a user by a server is extremely limited, and even most of it is automatically detected in the client software, so the reputation of the server can be instantly lost. 

### Reserve

When the user wants to trade token `A` for token `B` on exchange `E`, the first step is that the client *reserves* some amount of token `A` for usage on exhange `E` using the `reserve` method of the Tokenium smart contract:

	uint Tokenium.reserve(address token, uint amount, address exchange)

This will not actually move tokens to the exchange address: The funds are just temporarily moved to the Tokenium contract, and the contract remembers that the fund is locked for usage on `E` until the reservation expires.

This reservation is needed to prevent the attack when a disthonest client wants to attack the honest server and clients. To prevent this kind of attack, the exchange have to be sure that the order of a client has a reserve, so the user cannot spend the tokens on the Ethereum network meanwhile the order-matching algorithm runs inside the exchange.

The return value of the function is a trading session id.

The server will eventually notice this reserve on the blockchain, and report it to the client that it is ready to take orders.

Now the client can start to issue orders on the trading session (until then the server will not allow it).

### Unreserve

When the user presses the *unreserve* button for a trading session, the client will immediately call the *unreserve* method on the server, a honest server will immediately disable all trading on the trading session.
A honest server also calls the *unreserve* method on the smart contact. (We will later discuss how the client detects dishonest server behaviour.)

### Emergency Unreserve

This is done by the client on the smart contract without any server involvement. Usually it is done when the client detects incorrect server behaviour. (The user have the right to initiate this anytime though). It is done in two phases and lasts several ethereum steps so the the server can notice in time when a dishonest client does this and can start not to match its orders before the emergency unreserve is finished.)

### Market Orders, Cancellations and the 'Server Promise List'

On the user interface the user makes an order for trading some amount of token `A` for token `B` at a price limit, by digitally signing this data structure (off-chain):

	{in_session_order_index, from_token, to_token, amount, price_limit}

The user can cancel orders anytime fast (off-chain), just like on centralized exchanges.
The exchange does real-time order matching.

The server collects real-time what kind of submissions it will do to the smart contract when it will be able to. This list is the **promise list** of the server, because basically the server promises, that it will submit these to the smart contract. The list consist of 3 kinds of items:

* cancellations
* matched order-pair submission (These are the most important submissions: The smart contract will examine the signature of the orders in the pairs, and if they are valid and this `in_session_order_index` is not canceled yet, and it will execute the token swaps.)
* unreserve

The client tracks this list and will detect if not the exact same things have been submitted to the smart contract by constantly monitoring the smart contract. This monitoring is done by a kind of blockchain algorithm. For each submission a hash is calculated from some part of the submission data + the previous hash. Only the current hash is stored in the smart contract. The client will read this hash constantly, will look it up in its own promise list where it also calculated the hashes, and if the lookup is not successful, or there is no progress for a certain amount of time, then the client will report to the user that there is a problem with the server, so that the user can do an emergency unreserve. 

It can be seen that the server can only lie with its promises only for a small amount of time, and after that not only the client will do emergency unreserve, but the server will probably lose its reputation too.

Please note that the number of submissions sent to the server is minimized. As only one order can be active in a reserve session at a time, each order can be represented with a serial number (`in_session_order_index`), the `cancel` command can be represented with just one number: orders up until this number are not active anymore. Multiple fast cancels are also moved into one `cancel` command by the server, so `cancel(1)`, `cancel(2)`, `cancel(3)` will be just one `cancel(3)`. And finally a matched order pair submission automatically means cancel for all previous orders.    

### Fees

Fee will be part of the order datastructure.

We create a new token for Tokenium. Fees are denominated in this token and is payed by the `submit` method: 95% goes to the exchange account and 5% to the Tokenium tem's address. The reason for the 5% is that is to secure Tokenium development in the far future, when the team can no longer rely on the token sale money. 

### Gas considerations

It must be considered that the exchange cannot be DDOS-ed into spending all its money on gas.
The `reserve` method in the smart contract will be called by the client, so the user directly pays the gas for it.
Submissions from the server are payed from the server, so we will make sure that the server can always have enough gas from the client so it cannot be ddosed.

### Tokenium Wallet User Interface

The tokenium wallet user interface will be divided into 2 parts. The left part will show data according to the client's local database and confirmed data on the blockchain. The right half will be the orderbook and server promises. This way if there is a contradiction in the data, the user knows that the data on the left is true.

### Plans

It would be hard to write a detailed roadmap at this point yet, but here are the activities that the Tokenium team will pursue:

The most important thing is to have the Tokenium protocol stable and secure. For this to happen our first activities are these:

* Bounty for experts to ancourage them to write as much thoughtful analysis on Tokenium as possible. (maybe raising security or usability issues). 
* Detailed specification of the protocol standard
* Developing proof of concept for the client and server and developing attack simulations. 

If unsolvable problems appear in this phase we might cancel the project before anyone invested money in it (so before token sale).

Further down the road:

* Token Sale (maybe first a token presale)
* We develop the native open source software called *Tokenium Wallet* in Qt.
* We will develop an exchange server. Although we hope that third parties will develop more awesome exchange servers than us, we have to make sure that there is at least one Tokenium exchange server developed as fast as possible. As this server will be open source, anyone can start to host an exchange server without developing one! 

### Token distribution

* 5% is preserved for bounties. (Like the expert bounty mentioned above)
* 30% is preserved for the team and advisors
* 65% is up to token presale and token sale.

## Contact

* Twitter: [https://twitter.com/TokeniumNetwork](https://twitter.com/TokeniumNetwork "TokeniumNetwork") 
* Website: to be deployed (tokenium.org)

* Founder: Ádám Nagy
* LinkedIn: [https://www.linkedin.com/in/adam-nagy-03575834/](https://www.linkedin.com/in/adam-nagy-03575834/ "Ádám Nagy")
* email: nadam60@gmail.com

