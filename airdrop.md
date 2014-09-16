
BitShares Toolkit Airdrop Support
=================================

Donation addresses
------------------

- Bitcoin: 1NfGejohzoVGffAD1CnCRgo9vApjCU2viY
- Litecoin: LiUGs9sqH6GBHsvzpNtLCoyXB5aCDi9HsQ
- Dogecoin: DTy5q7uUQ1whqyUrwC1LbhgqwKgovJT5R7
- Protoshares (BitShares-PTS): PZxpdC8RqWsdU3pVJeobZY7JFKVPfNpy5z
- BTSX: drltc

The problem
-----------

In [this thread](https://bitsharestalk.org/index.php?topic=8858.0), xeroc proposes airdropping a custom BitAsset called FREE to coin holders.
But an airdrop to all holders of some altcoin would of course be prohibitively expensive in terms of transaction fees.

Allowing "private airdrops" would obviously be very useful functionality for marketing purposes.  In this whitepaper, I will show that
private airdrops are possible in practice with suitable modifications to the BTSX blockchain code.

My solution
-----------

I propose implementing a built-in mechanism for "private airdrops" to support sending BTSX or any BitAsset proportionally to all holders of
some other coin.

Requirements
------------

Here are the desirable features achieved by my solution:

- Allow sending coins to an "airdrop distribution list", an immutable mapping of public keys to weights
- An airdrop distribution list is stored as a single hash (root of a Merkle tree), so is extremely light-weight
- An airdrop can be issued proportionally to all holders of some altcoin(s), or any other method of assigning weights to public keys
- Airdrop distribution list can be verified for fairness if it uses a published algorithm on publicly available blockchain data
- Anyone can contribute to any airdrop; allows anonymous crowdfunding of airdrops
- Airdrop distribution lists persist, allowing cheap future airdrops to same snapshot
- Zero storage / processing required of BTSX nodes / delegates for altcoin balances that never claim airdrop funds, including dust
- Airdrop funds have a limited claim period; no funds are wasted on recipients that don't show up

Any solution should avoid undesirable consequences:

- Do not print BTSX or any BitAsset; airdrop funds must already exist
- Do not impose ruinous fees on the airdrop manager
- Do not impose undue resource burdens on the BTSX blockchain
- Do not put anything in BTSX blockchain permanent storage unless the network receives a fair transaction fee for the bytes used
- Do not require any node or delegate to interface directly to any altcoin, including parsing, storage, or running altcoin daemons
- Do not further centralize power in the hands of delegates

New address types
-----------------

The key is two new types of address which I propose, **snapshot addresses** and **dividend addresses**.  A snapshot address is simply a
Merkle root hash whose leaves are (pubkey, ownership) for each public key in the altcoin blockchain, where "ownership" is the number
of altcoins they own at some fixed snapshot date.  A snapshot address can be thought of as an "airdrop distribution list."

A dividend address is used as a ledger for the financing of a particular airdrop.  The airdrop itself is considered to be an
atomic event that occurs at point in time, the **dividend date**, which is specified when the dividend address is created.

Snapshot addresses
------------------

Anyone can pay the standard transaction fee to publish, on the BTSX chain, a claim against any snapshot address.  A
**snapshot claim** is a transaction consisting of a cryptographic proof that:

- (pubkey, ownership) is a leaf node in the Merkle tree
- The claimant controls the private key corresponding to pubkey

A snapshot address does not control any funds, and a snapshot claim does not itself result in the transfer of any funds
(except for the transaction fee).  Basically a snapshot address represents an immutable weighted list of a very
large set of recipients, most of whom are not expected to participate.

Dividend addresses
------------------

A **dividend address** represents a particular distribution of funds to the recipient list represented by a snapshot address.
A dividend address has an associated snapshot address and a distribution date.
Conceptually, the *distribution* is an atomic proportional transfer of funds from the dividend address to everyone who
published a snapshot claim before the distribution date.

In practice, of course, this is most easily implemented with the following algorithm:

- When the dividend date arrives, record `div_address.total_assets`, the total amount of funds (of all BitAsset types) in the dividend address.
- Set `div_address.unclaimed_assets` to `div_assets.total_assets`.
- Record `div_address.total_shares`, the total amount of snapshot ownership which was claimed prior to the dividend date.
- Shares claimed later are ineligible to receive this dividend, so `div_address.total_shares` is immutable.
- The `div_address.unclaimed_shares` is also immutable.
- Set `div_address.unclaimed_shares` to `div_address.total_shares`.
- In any blocks after the dividend date, transactions *sending funds to* the dividend address become illegal (they will not verify after the dividend date)
- In any blocks after the dividend date, transactions *claiming funds from* the dividend address become legal (they will not verify before the dividend date)
- Any recipient address can only issue one dividend claim transaction per dividend address, in the amount of exactly `floor(div_address.total_assets * recipient_shares / div_address.total_shares)`.
- When funds are claimed, reduce `div_address.unclaimed_shares` by `recipient_shares`
- When funds are claimed, reduce `div_address.unclaimed_assets` by the payout amount.

Unfavorable rounding errors are likely to accumulate in this process.  It would be nice to assess these rounding errors as fees for the network if possible.  It is actually
quite simple to do this with each dividend claim transaction:

- The total remaining users' claims will total at most `ceil(div_address.total_assets * div_address.unclaimed_shares / div_address.total_shares)`.
- Any difference `div_address.unclaimed_assets - ceil(div_address.total_assets * div_address.unclaimed_shares / div_address.total_shares)`
  is the network fee; it is "safe" to assess this fee immediately, because the network can prove that no user will ever claim this residue.

Example
-------

So here's how an individual or organization wishing to market BTSX to Dogecoin holders might use this infrastructure:

- Brag about how you're going to give X of some BitAsset such as BTSX / BitUSD / BitBTC / FREE in Dogecoin forums, subreddits etc. to all who hold Dogecoins on say November 1, 2014.
- When the Dogecoin chain reaches November 1, 2014, publish the Dogecoin block hash of the snapshot.
- Publish a script that builds the Merkle tree from the Dogecoin chain state at the snapshot block.
- The Merkle root hash is the snapshot address.
- Create a dividend address with a funding cutoff date of November 12, 2014, and a start-of-distribution date November 15, 2014.
- The marketer and all his friends send their BTSX / BitUSD / BitBTC / FREE etc. to the dividend address.
- Dogecoin holders see how much is building up in the dividend address, and can tell whether their share of the impending dividend will be bigger than their transaction fees.
- To generate a snapshot claim, Dogecoin holders generate a Merkle proof that (pubkey, ownership) is a leaf node, then sign it with the corresponding private key.
- Generating the claim for a public key could be done with a short script, but it needs access to the entire snapshot state
- It would be simple and inexpensive for the marketer (or I3 team) to create a web service to generate claims for public keys
- On November 15, 2014, the first dividend claims can be published, and some shibes (Dogecoin enthusiasts) begin to participate in the BitShares ecosystem.

Transaction fees
----------------

As I've described my proposal so far, there is a problem where airdrop recipients would have to separately acquire BTSX to pay the transaction fee for their
snapshot claim transaction.

To solve this problem, we can allow a dividend address to have a separate BTSX balance attached as a transaction fee pool.  Whenever someone presents a valid
share claim, the transaction fee for that share claim will be paid by the transaction fee pool (provided the transaction fee pool has sufficient funds).

Of course we must set some minimum dust threshold to keep someone with ten thousand tiny claims from eating up the transaction fee pool.  The dust threshold
may depend on various non-blockchain-verifiable factors like the market price for an altcoin for which there is no BitAsset and the amount of crowdsourced
contributions that will arrive before the dividend is paid, but have not yet been sent.  For that reason, we should simply allow the dividend address's
creator to set, and change at will, the dust threshold for the transaction fee pool.

Alternative:  Mutable lists
---------------------------

If we use the Patricia trie data structure adopted by Ethereum, we can make mutable distribution lists.  I think trie proofs are somewhat longer, the
implementation is more complicated than a Merkle tree, and the use case is not so clear.

Alternative:  Multicast addresses
---------------------------------

The main trade-off of using a Merkle hash is to pay (almost) nothing for users who don't show up, in exchange for paying more for
users who do.  For airdrops to altcoins or other situations with a low hit percentage, this will certainly be a good tradeoff.  (Many users
won't hear the marketing message.  Of those that do, for a variety of reasons some will decline to claim the funds.  Many addresses would
only be able to claim economically useless dust amounts.)

For higher hit percentage situations, it might perhaps be useful to have a "flat" address where the blockchain tracks all recipients; this
would save publishing a Merkle proof for each ownership claim.  One example might be keeping the ownership ledger of some business venture
on the blockchain, then allowing transactions when share ownership changes.  Those wishing to pay the owners in proportion to their ownership
might simply send to a single address instead of lengthy multi-recipient transactions.

Such a multicast system would be more useful if multicast reception shares were themselves tradeable assets.  This is starting to sound
an awful lot like Bitshares ME.

