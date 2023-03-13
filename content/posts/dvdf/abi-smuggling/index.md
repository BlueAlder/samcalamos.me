+++ 
draft = false
date = 2023-03-13T18:18:37+11:00
title = "ABI Smuggling - DamnVulnerableDeFi v3 #15"
description = "Writeup for ABI Smuggling"
slug = ""
authors = []
tags = ["dvdf", "writeup", "defi", "abi smuggling"]
categories = []
externalLink = ""
series = []
toc = true
+++

## Overview

In this challenge we are provided with a single challenge contract which requires us to extract all funds from the contract. It has an Authorization mechanism which is in `AuthorizedExecutor.sol` that only allows certain addresses perform certain actions through permission based authorization. In the setup of the challenge the deployer is setup to be able to call the `sweepFunds()` with signature **0x85fb709d**, and the player is able to call the `withdraw()` function which has a signature of **0xd9caed12**.

Working backwards from the `sweepFunds()` function it has a modifier of `onlyThis` which will only allow the challenge contract itself to call this function. Meaning this will need to come from the execute() function which is limited by the permissions that are configured in the setup of the challenge.

## The execute() function

To begin let's take a closer look at how the execute function works. It takes 2 parameters, that being an address of the target you want to execute on (which because of the _beforeFunctionCall() requires this to be the contract itself) and the `actionData` for the call which is the calldata you want to call that address with.

As permissions are enforced by hashing the (sender, target, functionSelector) the content of the actionData can be anything as long as we are calling the right function on the right contract. However the player's permissions are limited to the `withdraw()` function which does not have any parameters we can play with. So let's take a closer look on how the function retrieves these values from the `actionData`.

The sender and the target are already provided from the function call, so there isn't much we can do to change this, however there is this interesting bit of code which retrieves the function selector from the action data.

```sol
bytes4 selector;
uint256 calldataOffset = 4 + 32 * 3; // calldata position where `actionData` begins
assembly {
    selector := calldataload(calldataOffset)
}
```

This is where a bit of ABI encoding knowledge will come in handy. This code is reading the calldata which was called with the `execute()` function call. The call data will look something like this

![ABI overview](abi-overview.png)