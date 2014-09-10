
drltc's BitShares interest analysis

September 10, 2014

About
-----

If you like this whitepaper, please donate:

    Bitcoin: 1NfGejohzoVGffAD1CnCRgo9vApjCU2viY
    Litecoin: LiUGs9sqH6GBHsvzpNtLCoyXB5aCDi9HsQ
    Dogecoin: DTy5q7uUQ1whqyUrwC1LbhgqwKgovJT5R7
    Protoshares (BitShares-PTS): PZxpdC8RqWsdU3pVJeobZY7JFKVPfNpy5z
    BTSX / BitUSD / BitBTC: drltc

Introduction
------------

This is my proposal in response to bytemaster's proposal here:  https://bitsharestalk.org/index.php?topic=8520.msg112027#msg112027

There are basically two different problems:  The equity-zeroing problem and the non-exponential problem.  My prior forum post was trying to solve both of them,
but I basically realized that the non-exponential problem is not actually a problem; it's an implementation of a supply-side yield curve!  I proposed a
fixed-interest version of this myself in my BitBonds proposal.

Equity-zeroing
--------------

If everyone with a yield claim cashed in the claim in the next block, how much of the yield fund would be left over (assuming no fees or rounding errors)?

Ideally the answer should be zero; I call such a yield algorithm "equity-zeroing" since all of the yield fund "belongs" to users
and none is left over for the network.  The first problem I was trying to solve was making an equity-zeroing yield algorithm.

We can conceptualize each balance as generating an equal number of "yield shares".  The amount of yield given to each yield share then
implies the number of shares the network "thinks" are in existence:

    yield_per_share = total_yield_fund / yield_shares_authorized

For example, with bytemaster's scheme, a balance B will generate yield shares equal to

    B.share_count = B.amount * (0.8 * B.t + 0.2 * B.t^2)

where B.t is the age of the balance in years (which will be between 0 and 1 because balances must move once per year).
I let f(t) = (0.8 * t + 0.2 * t^2) which simplifies the balance to:

    B.share_count = B.amount * f(t)

If we set

    yield_shares_authorized = total_bitusd_in_existence

The fund's total liability -- what it would pay out if everyone requested their yield in the next block -- is given by:

    total_liability = sum_B ( B.share_count * yield_per_share )
                    = yield_per_share * sum_B ( B.amount * f(B.t) )

Since f(B.t) <= 1 we know that

    total_liability <= yield_per_share * sum_B ( B.amount )
    total_liability <= yield_per_share * total_bitusd_in_existence
    total_liability <= (total_yield_fund / total_bitusd_in_existence) * total_bitusd_in_existence
    total_liability <= total_yield_fund

Verifying this choice results in the fund's solvency (whew!)

CYD algorithm
-------------

A careful reading of the math in the prior section will show that total_liability is equal to the total_yield_fund when
f(B.t) = 1 for everybody.

This is almost never the case!  And unfortunately, this is farthest from the case when launched (BitShares has been in
existence for less than a month so everyone's f(t) will be less than 0.07 or so.

This section will solve that problem, all it requires is keeping track of aggregate BitUSD CYD in each block.

A balance which hasn't moved for some period of time t, is said to have coin-years-detroyed (CYD) given by:

    cyd = balance * t

We can re-compute the share count in terms of CYD as follows:

    B.share_count = 0.8 * B.cyd + 0.2 * B.cyd * B.t
                  = ( 0.8 + 0.2 * B.t ) * B.cyd

We can then define g(t) = 0.8 + 0.2 * t, obtaining

    B.share_count = g(B.t) * B.cyd

We know that B.t <= t_max, where

    t_max = min(1, years_since_first_bitusd_created)

so we can bound

    B.share_count <= g_max * B.cyd

where g_max = g(t_max).  I claim that the following choice yields a solvent fund:

    yield_shares_authorized = g_max * total_cyd_in_existence 

Let's verify that claim:

    total_liability = sum_B ( B.share_count * yield_per_share )
                   <= yield_per_share * sum_B ( g_max * B.cyd )
                    = yield_per_share * g_max * sum_B ( B.cyd )
                    = yield_per_share * g_max * total_cyd_in_existence
                    = (total_yield_fund / yield_shares_authorized) * g_max * total_cyd_in_existence
                    = total_yield_fund

Solvency!  Basically CYD are worth somewhere between 0.8 shares (for new balances) and
g_max (for first-issue BitUSD balances, or year-old balances when BitUSD is more than
one year old).  This bound conservatively assumes all CYD are held by balances that
are old enough to get that maximum worth.

The maximum error is g_max - 0.8, which is bounded by a number which increases linearly
from 0 (at the BitUSD issue block) to 0.2 (when we reach the one-year mark and balances
begin rolling).  So on average the network's equity will be 10% of the fund (assuming
uniformly distributed balance ages).  But it is mathematically impossible for the
network's equity to exceed 20% of the fund, and equally impossible for the fund to
become insolvent (provided that paying yield is the only purpose of the fund).

Closer bounds
-------------

The most promising idea for a closer bound is to track the actual distribution of CYD.
For example, we can keep track of how many BitUSD are in balances 0-1 days old,
1-2 days old, 2-3 days old, etc.  Giving us ~365 "containers" (we may
actually need 366 or 367 to deal with leap years and balances that have not quite
rolled over).  As transactions come in, the moving money resets from its current
container back to the first container.  For the first block after some hard-coded
time of day (such as midnight UTC), shift all containers to the right.

Then you can get a g_max for each container with very tight bound (at most 24 hours error).
Then

    yield_shares_authorized = sum_container ( container.g_max * container.cyd )

should give you a tight upper bound on the number of yield shares.

Yield curve
-----------

The function g(t) above can actually be interpreted as a yield curve.

Let's consider a simplified model of the yield fund.  Suppose Alice and Bob are
long-term long BitUSD investors who let their BitUSD balance accumulate interest.
Alice and Bob do not participate in active BitUSD trading, transaction, and
shorting activity which generates income for the yield fund.  As a simplifying
assumption, Alice and Bob pay no transaction fees.

The model runs for n periods of t years each, where n*t < 1 (so
we don't need to model what happens when they are required to
move their funds and reset their yield).  During each period, other
traders spend some fraction z of their wealth as fees, which are completely
redistributed as yield at the end of the period.  The growth rate after
one period is then:

    r = z / (1-z) 

Alice holds her money for n periods.  Assuming Alice has a starting balance
of 1 BitUSD, at the end of n periods her wealth is then:

    alice.growth = 1 + g(n*t) * n * r
                 = 1 + g(n*t) * (g(t) / g(t)) * n * r
                 = 1 + (g(n*t) / g(t)) * g(t) * n * r

Then Alice has a *yield advantage*:

    alice_advantage = g(n*t) / g(t)

Bob on the other hand tries to compound his wealth by re-investment.  We can
use a binomial approximation:

    bob.growth = (1 + g(t) * r)^n
               = 1 + n * g(t) * r + ...

The question is whether the boost Alice gets from her yield advantage is greater
than the boost Bob gets from the higher-order terms in the binomial approximation.
Before resorting to numerical computations, let's analyze the first of Bob's
higher-order terms.  Setting n ~ 365 and g(t) ~ 0.8, we have:

    bob.growth = 1 + n * g(t) * r + ( 0.5 * n * (n-1) * (g(t) * r)^2 ) + ...
               = 1 + n * g(t) * r + ( n * g(t) * r * (0.5 * (n-1) * g(t) * r) ) + ...
               = 1 + n * g(t) * r * (1 + (0.5 * (n-1) * g(t) * r) ) + ...

Blithely truncating the higher-order terms, Bob's advantage is:

    bob_advantage = 1 + (0.5 * (n-1) * g(t) * r)

This is actually a lower bound, of course.  Let's consider more general g(t) of the form

    g(t) = alpha + (1 - alpha) * t

and try to find alpha which makes Alice and Bob advantage equal.  We get:

    g(n*t) / g(t) = 1 + (0.5 * (n-1) * g(t) * r)
    g(n*t) = g(t) + (0.5 * (n-1) * g(t)^2 * r)

The above expression is quadratic equation in alpha.  With the help of a computer algebra system, we obtain

    A * alpha**2 + B * alpha + C = 0

where

    A = 1/2*(n*r*t^2 - 2*n*r*t - r*t^2 + n*r + 2*r*t - r)
    B = n*r*t - n*r*t^2 + r*t^2 + n*t - r*t - t
    C = 1/2*n*r*t^2 - 1/2*r*t^2 - n*t + t

Numerical experiments with the resulting solution shows that the crossover alpha value seems to be influenced
surprisingly little by changes in n and t.  But it exhibits a strong dependence on r.  The alpha = 0.80 chosen
by bytemaster seems to be the crossover value for an APY of approximately 86.5%.

Real alpha values
-----------------

Eyeballing Wikipedia's example at http://en.wikipedia.org/wiki/File:USD_yield_curve_09_02_2005.JPG of a real
USD yield curve I estimate to have an alpha value of about 0.69 (as determined by dividing the yield of 2.5%
at 0 years by the yield of 3.6% at 1 year.)

It would be interesting to examine the data for other years with higher or lower interest rates and see what the
g(1) / g(0) ratio is.

Prior criticism
---------------

I stated that the prior interest proposal (alpha = 0) destroys the fungibility of BitUSD.  I still believe that to be the case;
in that scenario, the long-term yield has such enormous advantage over the short-term yield as to encourage the latter
to establish fiduciary institutions to trade demand deposits against immobile balances.  As I have shown, an alpha value
much higher than 0.90 (the crossover rate for 28% APY) may make spammy "manual compounding" transactions profitable when
the network income is high enough.  It seems to me that the proposed value of 0.8 is reasonable for accomplishing policy
objectives.

