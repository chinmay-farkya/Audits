# Introduction

A time-boxed security review of the protocol was done by [Chinmay](https://twitter.com/dev_chinmayf), focusing on the security aspects of the smart contracts.

# About

I'm an independent smart contract security researcher, with a focus on the EVM ecosystem. I participate in audit contests, and do private audits. With Top 5 placings in audit contests of complex protocols like GMX v2 and Ajna Finance, I do my best to contribute to the blockchain ecosystem and its protocols by putting time and effort into security research. I also write educational [articles](https://medium.com/@chinmayf) to make developers more security-aware. Check my previous work [here](https://github.com/chinmay-farkya) or reach out on Twitter [@dev_chinmayf](https://twitter.com/dev_chinmayf), or Telegram [@chinmayf](https://t.me/chinmayf).

# Disclaimer

A smart contract security review can never verify the complete absence of vulnerabilities. This is a time, resource and expertise bound effort where I try to find as many vulnerabilities as possible. I can not guarantee 100% security after the review or even if the review will find any problems with your smart contracts. Subsequent security reviews, bug bounty programs and on-chain monitoring are strongly recommended.

# Protocol Details

ProphetStaking is a staking protocol where users can stake the $PROPHET token and earn rewards in the form of eth. The scope of this audit is the Prophet staking contract which allows users to earn a share of the revenue made by the Prophet swap bot.

# Security Assessment Summary

**_review commit hash_ - [cb257222b01668d25567efcfdc3dfae65abe5324](https://github.com/leeftk/prophetrevshare/tree/cb257222b01668d25567efcfdc3dfae65abe5324)**

A total of 4 issues were found, out of which 2 were of low severity and 2 were Informational findings.

# Findings

## [L-01] Staker might lose out on rewards

**Description :** The getReward() function sends out ether to the msg.sender according to the rewards this address has accrued. But the user might not know that he can't specify a recipient other than the address he used for staking, and if its a contract it might not be able to receive ether (in absence of a receive or payable fallback function).

**Recommendation :** Document that the staker needs to be able to accept ether if it happens to be a contract.

**Resolution** : Acknowledged

## [L-02] A new period cannot be started when an old one is still running, even though the code intends to do so

**Description :** In notifyRewardAmount() function, the else clause gets executed when block.timestamp < perodFinish which means that the current period has not yet ended and we are trying to start a new one. Even though the code has the intention to allow this, this else clause will never be reached because if block.timestamp < periodfinish, the call to receive() => notifyRewardAmount() will always revert due to the check in setRewardsDuration().

**Recommendation :** Either remove that check from setRewardsDuration(), or only allow a new period to start when the previous has ended.

**Resolution** : Acknowledged

---

## [I-01] Event emission uses wrong variable in stake()

**Description :** The stake function employs the balanceAfter - balanceBefore strategy to accurately calculate the actual amount transferred in by the staker. The reason behind this is the PROPHET token is a fee-on-transfer token (according to the tests and ProphetToken contract). The actual amount is consistently used across the function but not in the event.

**Recommendation :** Replace "amount" in event emission with "stakeAmount" for accurate off-chain tracking of data.

**Resolution** : Acknowledged

## [I-02] Caching state variables as local stack variables can save gas

**Description :** In updateReward(), rewardPerTokenStored is once stored and then again loaded from storage. Instead, we can store the returned value of rewardPerToken() in a local variable and then use that in other places to save gas.

**Recommendation :** Cache local variables to save gas. This is also applicabel in notifyRewardAmount() function.

**Resolution** : Acknowledged
