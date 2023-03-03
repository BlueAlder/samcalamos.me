+++ 
draft = true
date = 2023-03-01T16:33:31+11:00
title = "Puppet V3 - DamnVulnerableDeFi v3 #14"
description = "Writeup for the Puppet v3 challenge"
slug = ""
authors = []
tags = ["defi", "DamnVulnerableDeFi", "puppet v3", "writeup"]
categories = []
externalLink = ""
series = []
+++

# Writeup

This challenge is the third "puppet" challenge relating to Uniswap Pools and is
very similar to Puppet V2, so I will not be covering those concepts but will be
focusing on the Uniswap V3 concepts. To catchup I reccomend watching my [YouTube
video on Puppet V2](https://www.youtube.com/watch?v=M9s8wWOP9LU) (at the start
of the video I also reccomend watching Puppet V1)

## Challenge Overview

In this challenge we are given a Lending Pool which uses the oracle price from a
Uniswap V3 liquidity pool to calculate the deposit required to lend the pool's
DVT tokens. 

The goal of the challenge is to take all 1,000,000 DVT tokens from the Lending
pool with only 10 starting ETH.

## Exploit Overview

This is very similar to the puppet V2 exploit except we are now using the
Uniswap V3 protocol which introduces the idea of concentrated liquidity by
seperating the price range of the liquidity pool into discrete ticks. A great
resource in in learning the new Uniswap V3 protocol is the [Uniswap v3
Development Book](https://uniswapv3book.com/).

To exploit this challenge we just need to manipulate the price to heavily
devalue DVT relative to WETH, then wait some time so that our new price will be
weighted a bit heavier in the TWAP calculation. Then when we call the lending
pool, the price is low enough that we can extract all 1,000,000 tokens from the
pool.  

The main concept here I will cover is how the Oracle price is calculated. If you
don't care about the Uniswap V3 details and just want to get to the exploit you
can skip over the next section :).

### TWAP Oracle

The lending pool contract uses the Uniswap V3 liquidity pool's Oracle function
to calculate the price of the price of DVT in terms of WETH. In v3 it uses a
Time Weighted Average Price (TWAP) to calculate the price. This means instead of
using the current reserve amounts to calculate the price, it uses historical
data over a specified period (in this case 10 minutes) to calculate the average
price in that time. 

To do this Uniswap V3 uses a concept of `observations`. Observations are made
whenever a swap is made which changes the tick price of a pool. 

```sol
struct Observation {
        // the block timestamp of the observation
        uint32 blockTimestamp;
        // the tick accumulator, i.e. tick * time elapsed since the pool was first initialized
        int56 tickCumulative;
        // the seconds per liquidity, i.e. seconds elapsed / max(1, liquidity) since the pool was first initialized
        uint160 secondsPerLiquidityCumulativeX128;
        // whether or not the observation is initialized
        bool initialized;
    }
```

In these observations we store the tickCumulative values, which is the sum of
ticks at each second the history of a pool contract. The accumulated tick is
calculated by 

```
tickCumulative = lastObservation.tickCumulative  + (currentTick *
deltaTimeSinceLastObservation)
```

When a pool is initialised the first observation will have a tickCumulative of 0
since no time has passed.

When a swap happens and say the new tick of the pool becomes 12345 and it has
been 10 seconds since the last observation. Then the latest observation will
have a tickCumulative of:

`0 + (12345 * 10) = 123450`

Then another swap happens and the new tick price is 54321 and it has been 100
seconds since the last observation. Then the latest observsation will have a
tickCumulative of:

`123450 + (54321 * 100) = 5555550`

So now we have 3 observations stored.

| Observation Index | timestamp| tickCumulative|
|----------|--------|-----------|
| 0| 0| 0|
|1| 10| 123450|
|2|110| 5555550|

So now say we want to calculate the TWAP over the last 50 seconds and we call
this at timestamp 130. In other words we are asking, what was the average price
between timestamps 130 and 80?

We calculate the tickCumulative at timestamp=130, since the latest observation
*should* be the same as the current tick value we multiple the currentTick by
the deltaTime (which is 20 seconds) since the last obsevation and add it to the
accumulator to get our new tickCumulative.

```
last.tickCumulative + (currentTick * deltaSeconds)
5555550 + (54321 + 20) = 5609891
```

This gives us our current tickCumulative reading of 5609891.

We then want to calculate the tickCumulative at timestamp=80 (which is 50
seconds ago, in other words at the start of the TWAP period). This is done by
binary searching until either we find an observation which was taken at that
exact timestamp or we find two observations which are next to eachother which would contain the timestamp.

In this case we would find the two observations at indexes 1 and 2, I am going
to refer to these as the left and right boundary respectivly. Since we don't
have a exact price point at that timestamp we **interpolate** the data between
those two observations to calculate the estimated price at that time (this is
fancy for drawing a line between the two points). 

To calculate this we first calculate the time difference between the left and
right observsation's timestamps.

```
observationTimeDelta = right.timestamp - left.timestmap
100 = 110 - 10
```

Then the time delta from the left observation to our desired timestamp

```
targetDelta = target - left.timestamp
70 = 80 - 10
```

Then to "draw a line" between the two observsations and calculate the
interpolated value at that timestamp we calculate the average change in
tickCumulative per second between the two observations.

```
averageChangeTickCumulative = (right.tickCumulative - left.tickCumulative) / observationTimeDelta
54321 = (5555550 - 123450) / 100
```

We then multiply this by the timeDelta  and add the left.tickCumulative to get
the interpolated tickCumulative at timestamp=80.

```
tickCumulative = left.tickCumulative + (averageChangeTickCumulative * targetDelta)
3925920 = 123450 + (54321 * 70)
```

So we now have the tickCumulatives at t=80 and t=130 of  3925920 and 5609891
respectively. So now let's calculate the average tick for this period. We do this by subtracting the two tickCumulatives and then divide by the number of seconds in the period.

```
timeWeightedAverageTick = (t130TickCumulative - t80TickCumulative) / period
33679.42  = (5609891 - 3925920) / 50
```

From there we can calculate the price from the tick. The main concept here is
that we are using the accumulated ticks which creates heavier weights for
observations that cover a longer period of time which mitigates Oracle
inaccuracy during times of high violatilty.



<!-- *V3 actually stores the ticks, but this is easier to understand conceptually. To
understand the maths behind how this works, check out [this page in the Uniswap
v3 Development Book.](https://uniswapv3book.com/docs/milestone_5/price-oracle/).
But I am going to use prices for the rest of the explaination. -->


### Let's do the exploit

But in this challenge we manipulate the Time Weighted Average Price (TWAP) by
first heavily devaluing the DVT token relative to WETH, then waiting an amount
of time such that the oracle price is within our range of depositing the
collatoral required. This is possible because the TWAP period is only 10 minutes 

I won't go into all the details of how Uniswap V3 works as that is not is super
relevant to the exploit. The main difference that we need to know in this
challenge in how Uniswap calculates oracle prices and that 