# About

Chinmay is an independent smart contract security researcher, with a focus on the EVM ecosystem. He participates in audit contests, and does solo audits. With Top 5 placings in audit contests of complex protocols like GMX v2 and Ajna Finance, he does his best to contribute to the blockchain ecosystem and its protocols by putting time and effort into security research. He also writes educational [articles](https://medium.com/@chinmayf) to make developers more security-aware. Check his previous work [here](https://github.com/chinmay-farkya) or reach out on Twitter [@dev_chinmayf](https://twitter.com/dev_chinmayf), or Telegram [@chinmayf](https://t.me/chinmayf).

# Disclaimer

A smart contract security review can never verify the complete absence of vulnerabilities. This is a time, resource and expertise bound effort where I try to find as many vulnerabilities as possible. I can not guarantee 100% security after the review or even if the review will find any problems with your smart contracts. Subsequent security reviews, bug bounty programs and on-chain monitoring are strongly recommended.

# Introduction 
ProphetSwap is a router contract built to route all trade done through the bot and take a revshare fee that is withdrawable by the team. The scope of this audit is the custom router called ProphetRouter which interacts with Uniswap V2 pools and allows users to swap tokens to ETH and ETH to tokens.

# Security Assessment Summary

**_review commit hash_ - [6cf67250266526b3fab22b579045c8daeca92360](https://github.com/leeftk/prophetrouter/tree/6cf67250266526b3fab22b579045c8daeca92360)**

A total of 7 issues were found, out of which 1 was of Medium severity, 3 were of low severity and 3 were Informational findings. 

# Findings

# [M-01] Single-step ownership transfer is dangerous

**Description :** 
Single-step ownership transfer means that if a wrong address was passed when transferring ownership it can mean that role is lost forever. This can be detrimental in the context of `ProphetRouter`, where if `transferOwnership` method was called with a wrong `newOwner` address, then the fees collection from the contract, and many other methods will be bricked, since `ProphetRouter` relies heavily on owner-only methods.

**Recommendation :**
 It is a best practice to use two-step ownership transfer pattern, meaning ownership transfer gets to a "pending" state and the new owner should claim his new rights, otherwise the old owner still has control of the contract.

**Resolution** :
Fixed.

 ---

# [L-01] ProphetMaxBuy() allows 1 iteration lesser than the intended maxRetry

**Description :** 
`maxRetry` is intended to limit the number of `ProphetBuy` [essentially trying to swap an amount of ETH to a token] calls in `ProphetMaxBuy()` by iteratively reducing the amountIn of ETH to be swapped, if the previous call failed. So it means the execution should allow exactly `maxRetry`times to complete the swap, and if it still fails, then transfer back the ETH to the caller.

But the current logic only allows it to iterate `maxRetry` - 1 times. This is because in an iteration, it first reduces the count for this iteration and then compares it to 0 to find if all trials have ended => then break the loop.

For example, if `maxRetry` = 10, the first trial will have count = 9 and then execute, so on upto when in ninth trial count = 1, now in tenth trial it will first reduce count to 0 and then compare it to 0 and will not allow it to execute for the tenth time and just break the loop by sending back the ETH.

**Recommendation :** 
Place the `maxAttempts = maxAttempts -  1;` statement at the end of the iteration or after comparing it to zero. 

**Resolution** :
Fixed.

---

# [L-02] Directly swapping one token to ETH (or viceversa) is not the most optimal strategy

**Description :** All functions in the `ProphetRouter` always construct direct swapping path from the input token to output token and swap using only these 2 tokens (out of which 1 is ETH). This is not the most optimal strategy because of the following reasons :

 1. The tokenAddress <=> ETH pool might not exist on uniswap v2 in which case the swap is not at possible as the pair contract doesn't exist.
 2. The tokenAddress <=> ETH pool exists but the value returned from this direct swap might not be the most optimal (because price of all pools keep moving and these tokens are also present in other pools which might give a better combined swap result). 

For case 2, in some cases the pool might be very volatile due to broader market movements, which will cause failed transactions due to crossing slippage limits. And in some cases, the direct pool might be imbalanced such that the value returned (and also the off-chain calculated `amountOut`) is not best according to the market dynamics.

This will lead to opportunity loss for the user. 

**Recommendation :** 
To solve both of these, consider implementing a concept of hop tokens where maximum possible amount out is calculated by calculating the amounts returned from various combinations/ series of swaps, so as to get the best value possible.

**Resolution** :
Acknowledged.

---

# [L-03] The getAmountOut and getAmountIn functions will return wrong amounts if called externally

**Description :** 
For `ProphetBuy()`, the user needs to provide a minimum acceptable amount "`amountOutMin`" that is acceptable to him in terms of the returned token. This acts as slippage tolerance. A user would normally call `getAmountOut` or `getAmountIn` off-chain to find out the "amountOut" or "amountIn" and apply a slippage tolerance of lets say 1 % on that amount to get the "`amountOutMin`" (thats also what the DEX frontends do). The problem is that in `ProphetRouterV1.sol` these functions do not consider the prophet bot fees, that is being charged from them when they actually call `ProphetBuy()` etc. This means that this calculated amountOut is not real, and will cause failed transactions.

So if a user is using these functions to calculate "`amountOutMin`" this amount would be overestimated, and when supplied to `ProphetBuy(`), the function will calculate amount out after deducting the fees which may cause the swap to fail due to slippage limit being crossed. 

**Recommendation :** 
Add fee calculation to these functions so that users can correctly estimate "amountOut". Other buy and sell functions that use `getAmountOut` or `getAmountIn` will also need to be refactored to accomodate these changes. This is what `UniswapV2Library` also does in its own getAmount function, but `ProphetRouter` is deploying a separate trading fees on top of that, so that also needs to be accounted for. 

**Resolution** :
Acknowledged.

---

# [I-01] In swapTokensSupportingFeeOnTransferTokensForExactETH() path.length references can be replaced by path[1]

**Description :** 
This function is a private function only called from `ProphetSmartSell()` where the path is only of two tokens and path[1] is fixed as the WETH address when generating path using `getPathForTokenToToken()`. So any `path.length` references are not needed to find weth balance.

**Recommendation :** 
Replace all `path[path.length - 1]` references with `path[1]`. This will help with better code readability, and will also save some gas. 

**Resolution** :
Acknowledged.

---
# [I-02] In ProphetMaxBuy() `else if (msg.value  <= maxInputEtherAmount)` check is not needed

**Description :** 
The if statement above this clause checks if `msg.value  > maxInputEtherAmount` which means that a simple else clause would cover the other condition that we intend to do ie. if `msg.value  <= maxInputEtherAmount`

**Recommendation :**
Change this else if clause to just `else {...`. This will help with better code clarity, and will also save some gas. 

**Resolution** :
Acknowledged.

---

# [I-03] ProphetSmartSell does not need the `require(path[path.length -  1] ==  WETH)` check

**Description :** 
This check is not needed because the path is generated deterministically in the code itself => WETH will always be the path[1] address here because of the parameters supplied to `getPathForTokenToToken()`.

**Recommendation :** 
Remove this check, it is unuseful. Will save some gas. 

**Resolution** :
Acknowledged.

---

