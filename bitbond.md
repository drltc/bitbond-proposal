
drltc's BitShares interest proposal

September 6, 2014

About
-----

If you like this whitepaper, please donate:

    Bitcoin: 1NfGejohzoVGffAD1CnCRgo9vApjCU2viY
    Litecoin: LiUGs9sqH6GBHsvzpNtLCoyXB5aCDi9HsQ
    Dogecoin: DTy5q7uUQ1whqyUrwC1LbhgqwKgovJT5R7
    Protoshares (BitShares-PTS): PZxpdC8RqWsdU3pVJeobZY7JFKVPfNpy5z
    BTSX / BitUSD / BitBTC: drltc

Fungibility is important
------------------------

bytemaster's interest proposal [here](https://bitsharestalk.org/index.php?topic=8396.0) is
problematic because it destroys the fungibility of BitUSD.  BitUSD that haven't
moved are suddenly different from, and more valuable than, other BitUSD.  His proposal encourages "banks" to
spring up -- businesses that allow users to deposit their BitUSD on a fiduciary basis, and try really
hard to keep from moving their BitUSD.  Users then have some outside mechanism to move around claims
against BitUSD deposits while the deposits themselves sit in the bank vaults and accumulate the maximum
interest since they never move.  The users and banks would then split the enhanced interest they extract,
making this arrangement economically sustainable.

My proposal
-----------

I propose something different that avoids these problems, should be easy to implement on top of
the current code, and accomplishes some important objectives:

- Maintain a "FDIC fund" which can buy up "underwater" shorts in the case of BitUSD rally (BTSX crash)
- Pay interest to BitUSD holders as simply as possible
- Implement some "yield curve" to give greater rewards to long-term investors

My proposal accomplishes all of those objectives with minimum additions to the BitShares codebase.

Three objectives, three funds
-----------------------------

I propose three separate funds in which network-controlled BitUSD balances (from fees, etc.) will live.

- Insurance fund, used to recapitalize underwater shorts
- BitUSD interest fund, used to pay a low hard-coded interest rate to BitUSD holders
- BitBond interest fund, used to pay a higher market-determined interest rate to long-term long BitUSD investors

BitBonds are a new type of BitAsset which I will describe in this whitepaper.

All three funds derive income from fees and short interest (if that is ever implemented).
BitUSD in the funds does not receive interest.

The insurance and BitUSD funds' purpose is self-explanatory.  Those two funds have certain minimum
and maximum capitalization levels.  A fund below its minimum capitalization level has first priority
for income, and gradually takes a smaller fraction of the available income until it claims zero
income at its maximum capitalization level.  A fund above its maximum capitalization level will have
negative income, actually paying its balance over time to one or both of the other funds until it
is again at or below its maximum capitalization level.

The exact formulas for determining the capitalization levels and capital flows are somewhat
arbitrary policy choices; I may make suggestions later.

The interest on BitUSD has no "lock-in" period; as soon as you receive a BitUSD transfer, it starts
accruing interest, which is compounded every block (implemented by exponentiation of the per-block
interest rate).  This hopefully reduces people trying to "game" the system and gets rid of the
incentive to form "banks."

Also, having a fixed interest rate is much easier to market to financially un-sophisticated
users, as compared to a variable interest rate determined by complex macro-economic factors
that are difficult or impossible for ordinary people to understand, and difficult or
impossible for even large, financially savvy market participants to control.
(Perhaps we could call it a "reward percentage" or "reward rate" if using the term
"interest rate" would have unpleasant regulatory consequences.)

The unpredictability of the insurance needs and fee income is reflected by changes
in the interest rate offered to BitBond holders.

BitBonds
--------

The bond fund sells BitBonds.  A single unit of BitBond is a promise to pay 1 BitUSD over
the course of a year in equal installments.  Every BitBond is fully backed by an existing
BitUSD that is destroyed when the BitBond is created, thus there is never any net printing
of BitUSD.

BitBonds have an "inactive" and "active" form.  The inactive form has interest payments
suspended, and is tradeable and fungible against all other inactive BitBonds.
The active form pays interest, is non-tradeable and non-fungible, but may be
converted to an inactive BitBond (by publishing a transaction and
paying the transaction fee).

BitBonds must be redeemed for interest.  If Alice has 96 BitBonds, after one month
1/12 of the BitBonds (8 BitBonds) will have been redeemed into 8 BitUSD.

Increasing an account's balance of active BitBonds will cause the timetable for
interest payments on *all* BitBonds owned by that account to reset.  Thus if Alice
owns 100 BitBonds and takes six months of interest payments, then inactivates the
remaining 50 BitBonds and sells them to Bob, Bob will receive 50 BitUSD over a
full year.  I.e. the inactivation "resets the clock" and spreads the remaining
interest out over a full year.

This makes BitBonds fungible, and punishes bond investors who "change their
mind" about being long-term investors and "cash out" by selling their BitBonds
on the secondary market.  While still *providing* an active, liquid, on-chain
secondary market, so investors can be assured that they will have a quick
exit strategy if a sudden need arises to convert their BitBonds to BitUSD.

Interest on interest
--------------------

A BitBond holder has funds dribbling slowly into their BitUSD bucket from
continuous interest payments, but *also* have a slow continuous
exponential increase in their level of their bucket due to the BitUSD itself
earning interest.

Which sounds like an *interesting* exam problem in a differential equations class,
and also an *interesting* exam problem in an economics class trying to figure out
how the BitUSD-to-BitBond exchange rate is affected by the interest rate
of BitUSD.

But fortunately, we can design BitBonds to avoid these *interesting* questions,
which is important because it keeps the valuation simple and makes the BitUSD-to-BitBond
exchange rate independent of the BitUSD interest rate.  Since BitBonds are
fully backed by BitUSD, just have the balance that can accrue interest from the
BitUSD interest fund be equal to the *total* of the address's BitUSD and BitBond
balances!

In other words, the backing BitUSD of a BitBond never stop earning interest,
the interest those backing BitUSD earn is always paid directly to the BitBond
holder regardless of where the BitBond stands in the lifecycle.  BitBonds that
are inactive, active but ineligible for redemption, redeemable, or have already
been redeemed into BitUSD all earn exactly the same interest.

Writing BitBonds
----------------

New BitBonds are sold at auction approximately once every hour (I suggest picking the
block randomly to make it less likely for technical traders to spike their
transaction submissions immediately before an auction.)  Some percentage
of the reserve (perhaps about 1%) is divided into a supply and sold at a
descending price with some supply curve.

For example, suppose there are 8000 BitUSD in the bond interest fund, and the
supply is limited to 0.25% of the fund per auction.  The supply curve is a
hard-coded policy choice, I propose selling all of the supply at parity
(0% APY) and linearly decreasing as a function of APY until zero bonds are
sold at 100% APY (the fund becomes unwilling to write new bonds at the
massive interest rate implied by more than a 50% discount to face value).

The matching algorithm starts with the initial block's bond supply of
block_interest_supply = 8000 BitUSD * 0.25% = 20 BitUSD.  Walk down the bid order
book; before executing each bid, cap the available supply for that offer
using the supply curve (using my suggestion, this would be
block_interest_supply * (2*bid_price-1).)

Suppose there is a bid for 300 BitBonds at
0.99 BitUSD, 400 BitBonds at 0.97 BitUSD and 500 BitBonds at 0.95 BitUSD.
The supply starts at 20 BitUSD.  At the top bid of 0.99 BitUSD, we're
capped to 98% of the total block supply or 19.60 BitUSD; this is enough to fill
the bid, writing 300 BitBonds and leaving 16.60 BitUSD in the supply for
the next bid.  The bidder's funds of 297 BitUSD are destroyed, as are
the 3 BitUSD from the block's interest supply.  Thus the 300 BitBonds
are "authorized" by the network to "print" their full face value of
300 BitUSD when activated, since this will result in no net creation
of BitUSD.

The next bid is capped to 94% of the total supply or 18.80 BitUSD; the cap
is a no-op because the supply has already been depleted below that level by
the first bid.  The interest demanded by the second bid is 12 BitUSD, which
can be fully filled by the remaining supply of 16.60 BitUSD.  This leaves
4.60 BitUSD in the supply.

The third bid is capped to 90% of the total supply which is also a no-op.
The interest demanded by the third bid exceeds the available supply,
which means the third bid will only partially be filled.  The 4.60 BitUSD
remaining in the supply would pay the interest on 92 BitBonds at the rate
this bidder is demanding, so the third bid is partially filled for
92 BitBonds.  The supply has been depleted, so the auction ends; lower bids
remain and will be filled either during the next auction, or by user-to-user
trading if some user places an Ask order.

Note that, in blocks other than the once-per-hour auction, the BitBond
market functions like a normal market.  BitBond buyers and sellers can trade
(inactivated) BitBonds for BitUSD without restriction just like any other BitAsset
market.  Bidders do not really care whether their BitBonds are
newly written from the fund, or are existing BitBonds sold by another user, since
they are identical and completely fungible.

Raiding the bond fund
---------------------

Note that the bond interest fund isn't insuring anything until bonds are actually
written, and at that point the BitUSD needed to guarantee those bonds is no longer
in the fund.  Thus we can include logic to raid the bond fund to recapitalize the
insurance or BitUSD interest funds when they are suffering.  The only consequence
of this is to slow or suppress the writing of new bonds, which will ultimately
lower the interest rate new (or "revolving") bondholders can receive.

A note on activation
--------------------

Since activating a BitBond causes *all* BitBonds owned by that address to reset their
clock, we have to be careful when to allow this to happen.  To prevent "accidents"
we want to allow only the owner of an address to increase that address's active BitBond
balance.  And include fail-safe logic which only allows BitBonds to be activated
in an account with no current active BitBond balance.  (This is not restrictive since
a user can simply transfer their BitBonds to a newly created address for activation;
this move-and-transfer operation will be such a common case that it should be
automated in the client and perhaps have an atomic single-transaction implementation
on the blockchain as well.)

Shorting BitBonds
-----------------

It seems like shorting BitBonds would be a lot of work to implement (how do you match
a redemption against a short?)  And the economic benefit would be questionable.
The main reasons for implementing long BitBonds are selling down network BitUSD
balances when they're overcapitalized, which is basically preventive maintenance
against a margin crisis; and making long BitUSD sensible for long-term investors
(if they have faith in the peg and the interest rate is good enough, BitBonds can
be a great deal).  Allowing shorting of BitBonds actively defeats the first purpose
(pushing bond prices lower and interest rates higher), and doesn't matter with
respect to the second.

So I recommend against allowing shorting of BitBonds.

Perhaps we could have a separate interest rate prediction market where longs and shorts
bet on the average clearing price of BitBond auctions?

Short interest is needed
------------------------

I believe that a fixed interest rate on BitUSD is an important feature for marketing
reasons.  And I believe that shorts must be charged interest in order to guarantee
solvency when paying a fixed interest rate on BitUSD.  If the interest charged
to shorts is greater than the interest paid to longs, this provides a strong
guarantee that the interest will always be paid.  Trying to cover interest payments
from fee income will result in a risk of undercapitalization when transaction
volume is low.

Optional extra:  Rolling BitBonds
---------------------------------

BitUSD investors with a long time horizon may wish to convert their BitBond interest
from BitUSD back into BitBonds.  To do this, we may create a network-managed
"rolling BitBond ETF."  Basically this ETF holds BitBonds and BitUSD on its books.
Anyone can create or cash out ETF shares for a proportional amount of
BitBond plus BitUSD to what the ETF is carrying in its asset column
(the odd Satoshi is always resolved in favor of the ETF, and shares can be created 1:1
for BitBonds or BitUSD when there are no outstanding shares whatsoever.)

At every auction, the ETF redeems all pending interest on its BitBonds and uses
all BitUSD on its books to place a non-competitive bid for BitBonds, then activates
all purchased BitBonds (in a new "address" to keep from resetting its existing BitBonds).

The non-competitive bid is a "market order for BitBond auction", a new type of bid
which can only be used in BitBond auctions. A non-competitive bid will always execute
at the clearing price, and the filled quantity of a non-competitive bid will always be
less than or equal to the quantity of normal ("competitive" / "limit order") bids.

