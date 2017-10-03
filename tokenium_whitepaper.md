# Tokenium Whitepaper

**Version 0.1**

Tokenium is an ethereum based token exchange protocol. It is enforced by the Tokenium smart contract. It makes possible the creation of efficient high-frequency token exchanges where users need to trust the exchange only in an extremely limited and well-defined manner. It combines the best of both worlds: Seamless high-frequency trading of centralized exchanges, and the (almost) trustless manner of decentralized exchange protocols.

## Existing Work

Currently centralized exchanges dominate the cryptocurrency market. These exchanges are very efficient because order-matching is done totally off-chain, but users have to trust these exchanges with their deposited money. This is a huge risk for the user, and also makes deposit/withdrawal processes painful in most cases.

Decentralized exchanges were proposed to solve the risk problem. There are several approaches:

Projects which maintain an order-book on-chain (like EtherDelta) have extremely slow user experience and prone to blockchain congestion and uncertanity because outcomes depend on trasaction order inside a block.

Several improvements have been proposed, some of them:
 * Automatic market maker solutions (e.g. Bancor)
 * Off-chain orders - on chain settlements (0x)

Both solutions have cons and pros. Automatic market maker solutions seem to be more convenient, but one problem seems to be that users have to be careful when buying/selling bigger amounts, as the calculated price can easily go away from the real market price for tokens.
The main problem with both solutions is that blockchain congestion and transaction order instability is not solved in either of them. More specifically in 0x the success of a *take* or *cancel* command cannot be determined before the block is actually mined because the success of a trnsaction depends on which order the transactions arrive to the ethereum node. So both solutions can face problems with very high traffic exchanges.

Another upcoming solution is *Kyber Network*. Kyber Network provides risk-free exchange service for people with fix prices continuously managed by reserve managers. Reserve managers are professional traders who are supposed to use centralized exchanges to trade, so Kyber Network is not really a competititor, rather a complementary solution to Tokenium.  

Tokenium was motivated by 0x. The contribution of Tokenium is to centrally match orders real-time off-chain in the exchange before submitting the matched order-pairs to the Tokenium smart contract in an extremely low-trust manner.

## Introduction To Tokenium

In Tokenium order matching happens off-chain in a usual highly optimized order-matching engine that we are used to in centralized exchanges. The protocol ensures that even if the exchange is compromised, its owners or hackers cannot run away with the user's money. All they can do is to not match an order when they could, or match it with not the best possible counter order (although of course within the specified limit!) This presents some limited risk for the user, but this risk cannot be compared to the risk present in case of centralized exchanges, where the whole traded funds can be compromised.

## Actors

There are 4 main actors in the Tokenium protocol.

### User

The person who wants to trade tokens.
The user can use a standalone exchange-agnostic client software like the Tokenium Wallet, or can connect to a specific exchange directly through an Ethereum-enabled browser (if the exchange supports this.)

### Exchange Server

A centralized server application. The exchange server must use the Tokenium Smart Contract Protocol. It also has to provide the Tokenium client API to be accessible by Tokenium exchange clients. (Optionally it can have its own web-based user interface too.)

### Exchange Client

An exchange-server agnostic exchange client. (The Tokenium team develops such an open source exchange client called *Tokenium Wallet*).
In the client the user can select which exchange server she/he wants to connect to. 

### Tokenium Smart Contract

It drives the Tokenium protocol and provides its trust guarantees.

## Tokenium Protocol Details

### Reserve / Unreserve

When the user wants to trade token `A` for token `B` on exchange `E`, the first step is that the user *reserves* some amount of token `A` for usage on exhange `E` using the `reserve` method of the Tokenium smart contract:

	Tokenium.reserve(address token, uint amount, address exchange, time expires)

This will not actually move tokens to the exchange address: The funds are just temporarily moved to the Tokenium contract, and the contract remembers that the fund is locked for usage on `E` until the reservation expires.

This reservation is needed because the exchange have to be sure that the order has a reserve, so the user cannot spend the tokens on the Ethereum network meanwhile the order-matching algorithm runs inside the exchange.

After the reservation expires, the client software will guaranteedly unreserve the money directly, completely independently of the exchange: The smart contract gurantees this except of course if the tokens were traded (see later).

	Tokenium.unreserve(address token, uint amount) // can be only called by the user account

Of course this method should only be used if the exchange becomes compromised. Normally the user don't have to wait for expirity: Pressing the *unrserve* button on the UI, a command is sent to the exchange, which will issue the second type of `unreserve` method, which will unreserve the tokens even if the reservation did not expire.

	Tokenium.unreserve(address token, address user, uint amount) // can be only called by the exchange account

The difference is that this second method can only be called by the exchange, so that the exchange will immediately know that the funds cannot be used in orders.

This scheme guarantees that the exchange will always only process orders with enough reserve, and users guranteedly get back their reserved tokens (unless they were traded within user specified limits).

### Market Orders

On the user interface the user makes an order for trading some amount of token `A` for token `B` at a price limit, by digitally signing this data structure (off-chain):

	{from_token, to_token, amount, expires, price_limit}

The user can cancel orders anytime fast (off-chain), just like on centralized exchanges. A compromised exchange can of course refuse to take into account the cancel command, but it is guranteed by the Tokenium smart contract that the order will not be applicable after the signed expritiy, so effectively there is a 'hard cancel' upon expirity.

The exchange does real-time order matching, and once each Ethereum step it will submit a batch of matched order-pairs to the smart contract.

	Tokenium.submit(...)

The smart contract will then examine the signature of the orders in the pairs, and if they are valid it will execute the token swaps.

If the exchange is not compromised, traders can be sure that their order has been filled real-time off-chain. They only have to wait for the Ethereum confirmation to actually have the tokens on the account, but this is an administrative step, which is totally deterministic. If the exchange is compromised this is not guaranteed, but it is guaranteed that their tokens will not be traded cheaper than the specified amount before the specified expirity.

### Fees

Fee will be part of the order datastructure.

We create a new token for Tokenium. Fees are denominated in this token and payed to the exchange account by the `submit` method.

### Gas considerations

It must be considered that the exchange cannot be DDOS-ed into spending all its money on gas.
The `reserve` method in the smart contract will be called by the user, so the user directly pays the gas for it.
The `submit` method is always called by the exchange, but the fee can be chosen by the exchange so that it is worth more than the gas needed for the transaction.
The `unreserve` method is normally called by the exchange, so in theory it can be DDOS-ed by the client. The exchange can rate-limit `unreserve` calls from the client.

### Main differences compared to 0x

The Tokenium protocol has some similarities with the 0x protocol. I would like to highlight the most important differences.

0x aims to be fully trustless while Tokenium makes a compromise here and allows a well defined trust towards the exchange.
0x aims to publish the user-signed orders among different exhanges providing higher liquidity for each exchange. Tokenium makes a compromise here too, and does not allow the publication of these orders: one order is only used in one exchange.
Because of the publication of orders to different exchanges, in case of 0x cancelling an order can be only done on the blockchain, and if the `cancel` happens within one Ethereum step with a `take`, their execution order will only be determined when the block is mined. On Tokenium you can cancel unmatched orders and cannot cancel matched orders real-time (we are speaking about millisecond time-scales here just like in case of centralized exchanges), the outcome of order matching has nothing to do with Ethereum steps. This allows very high frequency trading. Waiting for submitting order-matched transaction lists in the smart contract is merely an administrative step with well known outcome: This 15 sec wait is basically similar to a withdraw phase in case of centralized exchanges, only this is automatic.

The area for the exchange to dishonestly play with the orders is very low, especially if the exchange wants it to be not trivially detectable. Even this trust level can be lowered later when we introduce auditing features into the protocol to detect dishonest behaviour on the exchange side. Exchanges will be incentivized to try to not play with order-matching for small gains, as their whole reputation can be ruined because of it.

### Before token sale: Bounty for experts

We are not concerned with the token sale right now. The current priority is to gain as much security/usability peer review as possible from experts. So the first step of token distribution will be to distribute some portion of the tokens among experts who will contribute by writing thoughtful analysis on Tokenium (maybe raising security or usability issues). (Some of these experts might even join the Tokenium team.)   

### Plans

It would be hard to write a detailed roadmap at this point yet, but here are the activities that the Tokenium team will pursue:

* The most important thing is to have the Tokenium smart contract stable and secure.
* We define the standard API between exchange-server / echange-client.
* We develop the native open source software called *Tokenium Wallet* in Qt.
* We will develop an exchange server. Although we hope that third parties will develop more awesome exchange servers than us, we have to make sure that there is at least one Tokenium exchange server developed as fast as possible. As this server will be open source, anyone can start to host an exchange server without developing one! 
* Exchange auditing service, adding exchange auditing elements to the Tokenium protocol to be able to detect malicious exchanges.

## Contact

* Twitter: [https://twitter.com/TokeniumNetwork](https://twitter.com/TokeniumNetwork "TokeniumNetwork") 
* Website: to be deployed (tokenium.org)

* Founder: Ádám Nagy
* LinkedIn: [https://www.linkedin.com/in/adam-nagy-03575834/](https://www.linkedin.com/in/adam-nagy-03575834/ "Ádám Nagy")
* email: nadam60@gmail.com


## Appendix: Tokenium Smart Contract

Coming soon...
