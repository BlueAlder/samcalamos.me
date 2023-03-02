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

The main concept here I will cover is how the Oracle price is calculated.

### TWAP Oracle
The lending pool contract uses the Uniswap V3 liquidity pool's Oracle function
to calculate the price of the price of DVT in terms of WETH. In v3 it uses a
Time Weighted Average Price (TWAP) to calculate the price. This means instead of
using the current reserve amounts to calculate the price, it uses historical
data over a specified period (in this case 10 minutes) to calculate the average
price in that time. 

To do this Uniswap V3 simply tracks the previous prices* when swaps were
performed. Then it calculates the current price and then the price at the start
of the TWAP period (10 minutes ago) then simply averages the two to calculate
the TWAP and returns it to the lending contract.

*V3 actually stores the ticks, but this is easier to understand conceptually. To
understand the maths behind how this works, check out [this page in the Uniswap
v3 Development Book.](https://uniswapv3book.com/docs/milestone_5/price-oracle/).
But I am going to use prices for the rest of the explaination.


---

But in this challenge we manipulate the Time Weighted Average Price (TWAP) by
first heavily devaluing the DVT token relative to WETH, then waiting an amount
of time such that the oracle price is within our range of depositing the
collatoral required. This is possible because the TWAP period is only 10 minutes 

I won't go into all the details of how Uniswap V3 works as that is not is super
relevant to the exploit. The main difference that we need to know in this
challenge in how Uniswap calculates oracle prices and that 