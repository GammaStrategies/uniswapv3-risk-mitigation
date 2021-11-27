
## Notes on Uniswap v3 Arbitrage

###

**All current Visor LP managed positions are secured;, deposits for all management contract have been paused, and withdrawals available as always.**

On Thursday, Nov 25th 1:18pm UTC an economic attack was carried out on [Visor's OHM-ETH 1% LP management contract](https://etherscan.io/token/0x65bc5c6a2630a87c2b494f36148e338dd76c054f) by a malicious arbitrageur. Users funds were secured in full and deposit caps lowered were to put a hold on new deposits until a deposit proxy is deployed into production.  

*Later on that day several internal test contracts being used to test management strategies with Visor treasury assets were also arbitraged, this time by an EOA. Our test contracts operated without the usual deposit safeguards (whitelist, restrictive deposit caps, limiting the share of the pool the vault represents), making these profitable transactions possible. As these transactions occurred under a looser safety standard, outside the Visor platform, we will focus on the public OHM-ETH Hypervisor.* 

Here we outline the sequence of events carried out by the malicious OHM-ETH attacker.

### Liquidity Distribution
The malicious contract carried out its attack in repeated operations of diminishing returns.
https://etherscan.io/address/0x1a252684a15f07c97ec20b2d6bb380d7410058da

While we cannot see the contract code, it is easy to understand that the attack vector relied on large swaps to manipulate the Uniswap V3 pool's current price. Given the steep gradient in the liquidity distribution of this pair, we see that it is an attractive candidate for price manipulation, as after an initial threshold where liquidity is highly concentrated, impacting the price requires a relatively small amount of assets.

![OHM-ETH Liquidity Distribution](https://i.ibb.co/wWNvvDm/Screenshot-2021-11-27-13-49-10.png)'

In block [13683700](https://etherscan.io/block/13683700), in the contract's first [attack iteration](https://etherscan.io/tx/0x4208ef772b9ecb7a0494510101525e765240568d3788bab555942d344b984f67) 

 - 615.00 WETH were obtained from SushiSwap, presumably flashloan
 - 606.12 WETH were then swapped for 2,734.19 OHM in the V3 pool causing significant price impact
 - A 10 OHM single sided deposit was made, due to price manipulation, 49.77 LP tokens were minted at 68x their proper rate 
 - The remaining OHM was then swapped back for WETH, restoring current price
 - 49.77 LP tokens were then burned for  349.72 OHM, 26.14 WETH
 - Flashloan repaid
 
This process was repeated in 2 additional instances, with diminishing returns until the profitable liquidity gradient was exhausted.

|Iteration|Shares|OHM|WETH|
|--|--|--|--|
|1|47.77|349.72|26.14  |
|2|32.32|175.99|14.01  |
|3|15.97|55.86|4.67  |

It is notable that despite significant asset movement, current tick between blocks remains relatively stable

| block |tick  |
|--|--|
|13683699  |190503  |
|13683700  |190516  |
|13683726  |190539  |

However, using visualizations via dextools, the swaps are hard to miss
![enter image description here](https://i.ibb.co/Z68PJrB/candle.png)

Since the minting quantity of our vault's ERC-20 receipt tokens is determined via consultation of Uniswap v3 pool's current price:

    uint160 sqrtPrice = TickMath.getSqrtRatioAtTick(currentTick());

the attacker needed only to perform his deposit immediately succeeding his first swap.

It is interesting to note that a [potential second attacker](https://etherscan.io/tx/0x331a76215adbf75c02c715e6de2cb723022596e4fe7bd3eccd26e02421cc957a), what appears to be a generalized MEV bot, conducted a similar attack in between the second and third instances of the attack, in block ```1368725```. The details are slightly different but the effects were the same, as this bot was able to extract the following assets from the contract, with the same pattern of diminishing returns:

|Shares | OHM |ETH  |
|--|--|--|
| 18.26 | 60.48 | 6.19 |

## The Problem

Price manipulation is a classic economic exploit. Visor relied on floating deposit caps and total supply caps to mitigate this possibility by prohibiting large single-sided deposits and large shares of LP tokens to be minted at once. 

While this has been sufficient for pools like WBTC-ETH 0.3% where liquidity distribution is deep and symmetric, it can only go so far at safeguarding pools offering steep gradient opportunities for (relatively) cheap tick traversal.

## The Solution

**Safeguards**

- Forcing dual-sided deposits at the time of an LP's initiation of a position is a  sufficient countermeasure to prevent this attack, forcing any would be arbitrageur to match his small 10 OHM deposits with large amount of WETH to obtain the same quantity of LP tokens. Additionally, this mitigation allows Visor's relatively low deposit limits to be raised significantly without raising arbitrage exposure.

- Checking deposit pricing against significant deviation from a TWAP would additionally safeguard single sided deposits. 

Both safeguards mentioned are included in Visor's L2 management contracts currently under audit by Quantstamp.

## Immediate Steps

Due to the composable nature of Visor's management contracts, deposit proxies can be deployed and permissioned as gatekeepers. Varieties of these proxies have been used consistently in the past when deposit restrictions were desired.

The team will be deploying such proxy into production in the coming days and resume deposits.
