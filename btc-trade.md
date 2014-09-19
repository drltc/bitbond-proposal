e
drltc's BitBTC gateway proposal

September 13, 2014

About
-----

If you like this whitepaper, please donate:

    Bitcoin: 1NfGejohzoVGffAD1CnCRgo9vApjCU2viY
    Litecoin: LiUGs9sqH6GBHsvzpNtLCoyXB5aCDi9HsQ
    Dogecoin: DTy5q7uUQ1whqyUrwC1LbhgqwKgovJT5R7
    Protoshares (BitShares-PTS): PZxpdC8RqWsdU3pVJeobZY7JFKVPfNpy5z
    BTSX / BitUSD / BitBTC: drltc

The goal
--------

Goal:  What we want is a fully decentralized protocol to allow users like Bob to purchase BitBTC with regular BTC.

Assumptions
-----------

I will speak as if the following are true:

- We wish to enable trading BitBTC <-> BTC on the BTSX blockchain
- We are willing to require all delegates and validating clients to run a Bitcoin daemon (or parse the Bitcoin blockchain by some other means)

Whether these are modifications we are willing to make to the BTSX blockchain, or whether these are experiments we want to do on a separate blockchain, is a separate debate which I discuss at the end.

The two problems
----------------

The Goal can be decomposed into two problems:

- Atomic user-to-user BTC -> BitBTC transactions
- Market buying and selling BTC <-> BitBTC

User-to-user atomic exchange
----------------------------

Suppose Bob is buying BitBTC from Alice.  The way this would work is as follows:

- Alice and/or Bob publish time-limited promises on the BTSX blockchain to carry through the exchange.
- Part of Bob's promise is the txid of a "delivery transaction" which sends the right number of Bitcoin to Alice.
- At this point, Bob only publishes a "redacted delivery transaction" containing no signature.
- The redacted delivery transaction *does* contain the txid, destination, amount, and expiration for the transaction.
- Thus Bob's promise does not contain sufficient information for Alice to publish a transaction to take Bob's BTC prematurely.
- Alice includes the delivery transaction details in her own promise.
- Alice's promise tells the BTSX blockchain:  "Lock this many of my BitBTC.  If the delivery transaction is confirmed on
  the BTC blockchain, then transfer the locked BitBTC to Bob.  If you can prove that the delivery transaction can never be
  confirmed, then return the BitBTC to me (Alice)."
- The delivery transaction is provably never confirmable when the BTC blockchain's *confirmed state* does not contain the delivery
  transaction, and will never accept any future transaction with the expiration given in Bob's promise.
- For the delivery transaction to be confirmed, it must be confirmed by the BTC network.  In addition the txid,
  destination, amount, and expiration must match the values in Bob's promise.

Credit goes to https://en.bitcoin.it/wiki/Atomic_cross-chain_trading#Solution_using_specialized_altchain for this algorithm.

Dealing with no-shows
---------------------

The main problem with the above algorithm is that it may take a long time to for a delivery transaction to be proven
unconfirmable.  Thus it would be possible to launch a sort of denial-of-service attack on Alice as follows:

- Eve plays the role of Bob
- Eve promises to deliver 1000 BTC to Alice in exchange for 1000 BitBTC
- Alice agrees, telling the BTSX blockchain to lock 1000 BitBTC
- Alice's BitBTC are then locked for 24 hours and unavailable for transactions with honest traders
- Eve never shows up to carry through the transaction
- Alice gets her 1000 BitBTC back after ~24 hours plus 6 Bitcoin confirmations,
- But Eve made Alice suffer by locking substantial amounts of her funds for that length of time.

Non-performance bond
--------------------

One way to get around this is require Bob to "post a bond" to make himself punishable for non-delivery and arrange
for Alice to be compensated for her suffering if non-delivery occurs.  If the BitBond is payable in BitBTC, we can
modify the basic user-to-user atomic exchange protocol as follows:

- Bob publishes a promise on the BTSX blockchain of the form:  "Lock this many of my BitBTC.  If the delivery
  transaction is confirmed on the BTC blockchain, then transfer the locked BitBTC back to me (Bob).  If you
  can prove that the delivery transaction can never be confirmed, then transfer the BitBTC to Alice."
- Alice refuses to write her own promise until Bob's promise is published.

This bonding transaction is implementationally identical to Alice's promise in the basic user-to-user atomic exchange
algorithm.

Market-based approach
---------------------

Once we have bonded transactions, instead of having Alice and Bob meet privately, it would be possible for Alice
and Bob to be matched in an on-blockchain BTC <-> BitBTC exchange.  When orders are matched on such an exchange,
the participants follow the bonded atomic exchange protocol.  The bond BitBTC must be set aside when the BitBTC
buyer Bob posts his order to the exchange.  The required bond may simply be hard-coded, but I would advise a
more market-based approach:  Force the bond to fall within a hard-coded range (1% to 10%).  Allow Bob to specify
the maximum bond he's willing to give, and Alice the minimum bond she's willing to accept.  If non-delivery occurs,
then any difference is taken by the network as a fee.

When Bob has no BitBTC
----------------------

If Bob only has BTC, then Bob must have some way to get BitBTC in order for him to participate in the above protocol.

My solution to this problem is to allow accounts to act as *gateways*, which will do small unbonded transactions
for a 10% fee paid in advance.  If you want to convert, say, 7.931 BTC to BitBTC here is what you would do:

- Client securely contacts gateway, expressing desire to convert 0.010 BTC to BitBTC.
- Gateway promises to send client 0.010 BitBTC for 0.010 BTC, if client is willing to send gateway 0.001 BTC.
- Client sends 0.001 BTC.
- Gateway does unbonded user-to-user protocol to trade 0.010 BitBTC for 0.010 BTC.
- Client now has 0.010 BitBTC, client uses that 0.010 BitBTC as the bond in a bonded exchange to get 0.100 BitBTC.
- Client now has 0.100 BitBTC, client uses that 0.100 BitBTC as the bond in a bonded exchange to get 1.000 BitBTC.
- Client now has 1.000 BitBTC, client uses that 1.000 BitBTC as the bond in a bonded exchange to get 7.931 BitBTC.

Of course this is all hidden in the implementation; the user interface would just let you import a Bitcoin wallet,
type in "convert 7.931 BTC, paying at most 5% in fees + spreads", and wait a day or so.

This protocol neglects the minor matters of tx fees, and the negotiation of an exchange rate between client and gateway.

Clients of course must trust the gateway with 0.001 BTC.  The user may of course enter a specific TITAN account name
if they have reason to trust a specific person.  But there should also be, by default, a built-in list of
known-trustworthy gateways.  The default client implementation should select a gateway at random from this list.

Each gateway should have a designated BitBTC address for funding unbonded transfers, perhaps derived from their public
key using a known deterministic algorithm.  The default client implementation should choose to use gateways in proportion
to their designated address' BitBTC balance, but refuse to use gateways whose designated BitBTC address contains less
than 10x the desired BitBTC.  This should help prevent "bank runs" on popular gateways and allow the system to self-heal.

Why not BTC to BTSX?
--------------------

The "BTC forex" interface needs to wait for the BTC transaction to be included in a block, then wait for enough
confirmations (6?).  The K non-parallelizable transactions of the no-BTC protocol mean it will take K times
as long.

Substantial movement in the BTSX -> BTC exchange rate is possible during that time.  However, the BitBTC -> BTC
exchange rate should be steady at about 1:1.

Other altcoins
--------------

I would support adding BTC -- and only BTC -- to the BTSX chain.  I would support adding a few of the more popular altcoins such as
LTC, DOGE, PTS to a new BitSharesX derived blockchain (BitSharesY?) with the specialized purpose of trading BitAssets against matching
cryptocurrencies.  In any case, the need for all validating nodes to run the daemon of each external coin will limit the number of
altcoins that can be traded on a single chain.

I understand that there were at one point plans for allowing cross-chain movement of BitAssets between BitShares Toolkit derived chains.
I am not sure what the current state of those plans are, or whether anything has been implemented yet.

Implementing expiration times
-----------------------------

I was surprised to discover that Bitcoin actually has no built-in way to make a transaction become *invalid* on a certain date!  However,
it does have a way to make a transaction become *valid* on a certain date, called `nLockTime`.  We can "bootstrap" `nLockTime` into what
we need by modifying the protocol as follows:

- Bob includes the BTC inputs in the redacted delivery transaction.
- Along with the redacted delivery transaction, Bob sends Alice a *signed* "deadline transaction," locked in the future, sending the same BTC inputs to Bob's own address.
- If the deadline arrives and Bob hasn't delivered, Alice publishes the deadline transaction.
- If the deadline transaction is confirmed, then the delivery transaction is provably never confirmable (since its inputs were already spent in the blockchain's confirmed state).
- If Bob sends Alice the delivery transaction in a way that doesn't publicly disclose its contents, (i.e. by encrypting it with Alice's public key), Alice will have the power to voluntarily grant Bob a deadline extension by simply delaying when she publishes the deadline transaction.

