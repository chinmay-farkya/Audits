![](https://i.imgur.com/zGzXYfT.jpg)


__Audit Firm__: Solidity Lab

__Client Firm__: Poodl

__Prepared By:__ Chinmay Farkya

__Delivery Date:__ March 7th, 2022

<br />

PoodlTech engaged Solidity Lab to review the security of its Smart Contract system. From March 1st 2023 to March 7th 2023, a team of 1 auditors reviewed the source code in scope. All findings have been recorded in the following report.

Notice that the examined smart contracts are not resistant to external/internal exploit. For a detailed understanding of risk severity, source code vulnerability, and potential attack vectors, refer to the complete audit report below.



# Project Overview

| Project Name | PoodlTech Dividend Paying Token                                                                                             |
|--------------|----------------------------------------------------------------------------------------------------------|
| Language     | Solidity                                                                                                 |
| Codebase     | https://github.com/poodlTech/tokenAudit                            |
| Commit       | [eebe267b3fdd75a82e09cc270b3c046b2c9f2c84](https://github.com/poodlTech/tokenAudit/tree/eebe267b3fdd75a82e09cc270b3c046b2c9f2c84) |


| Delivery Date     | March 7th, 2023       |
|-------------------|--------------------------------|
| Audit Methodology | Static Analysis, Manual Review |


| Vulnerability Level | Total | Pending | Declined | Acknowledged | Partially Resolved | Resolved |
|---------------------|-------|---------|----------|--------------|--------------------|----------|
| [Critical](#Critical)| 1    | 0       | 0        | 0            |  0                | 0        |
| [High](#High)        | 2   | 0       | 0        | 0              |   0            | 0        |
| [Medium](#Medium)    | 9    | 0       | 0        | 0            |   0              | 0        |
| [Low](#Low)          | 5   | 0       | 0        | 0              |  0               | 0        |

# Audit Scope & Methodology

## Scope

| ID File | SHA-256      | Checksum                              |
|---------|------------|---------------------------------------|
| A      | contracts/DividendPayingToken.sol |03754dd9bb6810afaeea9f89c43b6587a123bc485aa7383cebe1d89480fe7b12  |
| B      | contracts/Token.sol | 415199e53bce4ef12717900ca4750e5fbf53b3451f0e17702dae3f1a47ba6f76 |
| C      | contracts/Presale.sol | 15e348fa45d6d18fa032638b18e31ea0332532daadcab8aa610a42d6679fa01f |
| D     | contracts/Airdropper.sol | a3c1bfbf048d07dd257994ebf94b1984fd7160f5c0cbd579388da1cf13fc6389 |

## Methodology

The auditing process pays special attention to the following considerations:
- Testing the smart contracts against both common and uncommon attack vectors.
- Assessing the codebase to ensure compliance with current best practices and industry standards.
- Ensuring contract logic meets the specifications and intentions of the client.
- Cross-referencing contract structure and implementation against similar smart contracts produced by industry leaders.
- Thorough line-by-line manual review of the entire codebase by community auditors.

## Vulnerability Classifications

| Vulnerability Level | Classification                                                                               |
|---------------------|----------------------------------------------------------------------------------------------|
| [Critical](#Critical)            | Easily exploitable by anyone, causing loss/manipulation of assets or data.                   |
| [High](#High)                | Arduously exploitable by a subset of addresses, causing loss/manipulation of assets or data. |
| [Medium](#Medium)              | Inherent risk of future exploits that may or may not impact the smart contract execution.    |
| [Low](#Low)                 | Minor deviation from best practices.                                                         |



# Findings & Resolutions

| ID      | Title                                                                                     | Category            | Severity | Status  |
|-------|-------------------------------------------------------------------------------------------|---------------------|----------|---------|
| [C-01](#C01)  | withdrawnDividends Not Updated                                     | Logical Error                 | CRITICAL     | Pending |
| [H-01](#H01)  | Hardcoded swapPath                                                        | Flexibility         | HIGH     | Pending |
| [H-02](#H02)  | Extreme Slippage Tolerance  | Slippage | MEDIUM   | Pending |
| [M-01](#M01)  | Minor deviation from best practices.                                                      | Logical error         | MEDIUM   | Pending |
| [M-02](#M02)  | Uninitialized Values                                                                              | Logical Error | MEDIUM      | Pending |
| [M-03](#M03)  | Address Not Excluded From Dividends                                                                              | Tokenomics          | MEDIUM      | Pending |
| [M-04](#M04)  | Pair Not Updated With Router                                                                              | Logical Error          | MEDIUM      | Pending |
| [M-05](#M05)  | Fees Cap Can Be Bypassed                                                                              | Caps          | MEDIUM      | Pending |
| [M-06](#M06)  | canClaim Can Be Avoided                                                                              | Logical Error          | MEDIUM      | Pending |
| [M-07](#M07)  | User Not Added Back To tokenHolders Map                                                                              | Logical Error          | MEDIUM      | Pending |
| [M-08](#M08)  | buyPresale Not Payable                                                                              | Logical Error          | MEDIUM      | Pending |
| [M-09](#M09)  | Wrong gasleft calculations                                                                              | Logical Error          | MEDIUM      | Pending |
| [L-01](#L01)  | Router not whitelisted automatically                                                                              | Logical Error          | LOW      | Pending |
| [L-02](#L02)  | Not Using SafeMath Functions                                                                              | Safe Functions          | LOW      | Pending |
| [L-03](#L03)  | Uninitialized Values                                                                              | Logical Error          | LOW      | Pending |
| [L-04](#L04)  |  Inconsistent Deadlines                                                                             | Logical Error          | LOW      | Pending |
| [L-05](#L05)  | newDividendTracker owner needs to be updated                                                                              | Logical Error          | LOW      | Pending |




## <a id="Critical"></a>Critical


### <a id="C01"></a> C-01 withdrawnDividends are not properly updated for a user

https://github.com/poodlTech/tokenAudit/blob/eebe267b3fdd75a82e09cc270b3c046b2c9f2c84/contracts/DividendPayingToken.sol#L154]

#### Description:
If user has a customRewardToken address set, the eth is swapped to his chosen reward token by calling swapEthForTokens inside the withdrawDividendOfUser() function at Line 131. If the swap through the router fails, the execution reaches logic where eth is sent to the user instead. But here, if the eth sending fails the withdrawnDividends state instead erases previous record by subtracting amounts, but it was not added like Line 136 does it for non-custom reward token. Thus, if the swap keeps failing the user can keep withdrawing dividends without a limit until the contract has zero funds of the native token. Another issue is that if the swap succeeds or swap fails and eth transfer succeeds, the withdrawnDividends is still not updated to record the amounts user withdrew.

```
function swapETHForTokens(
      address recipient,
      uint256 ethAmount
  ) private returns (uint256) {    
      bool swapSuccess;
      IUniswapV2Router02 swapRouter = uniswapV2Router;
      if(userHasCustomRewardAMM[recipient] && ammIsWhiteListed[userCurrentRewardAMM[recipient]]){
          swapRouter = IUniswapV2Router02(userCurrentRewardAMM[recipient]);
      }
      // generate the pair path of token -> weth
      address[] memory path = new address[](2);
      path[0] = swapRouter.WETH();
      path[1] = userCurrentRewardToken[recipient];
      // make the swap
      _approve(path[0], address(swapRouter), ethAmount);
      try swapRouter.swapExactETHForTokensSupportingFeeOnTransferTokens{value: ethAmount}( //try to swap for tokens, if it fails (bad contract, or whatever other reason, send BNB)
          1, // accept any amount of Tokens above 1 wei (so it will fail if nothing returns)
          path,
          address(recipient),
          block.timestamp + 360
      ){
          swapSuccess = true;
      }
      catch {
          swapSuccess = false;
      }  
      // if the swap failed, send them their BNB instead
      if(!swapSuccess){
          (bool success,) = recipient.call{value: ethAmount, gas: stipend}("");
          if(!success) {
              withdrawnDividends[recipient] = withdrawnDividends[recipient].sub(ethAmount);
              return 0;
          }
      }
      return ethAmount;
  }
  
  ```



#### Recommendation:
Add withdrawnDividends[recipient] = withdrawnDividends[recipient].add(ethAmount); to all logic branches consistently





#### Resolution:


----


## <a id="High"></a> High

### <a id="H01"></a> H-01 Hardcoded swap path should not be used


https://github.com/poodlTech/tokenAudit/blob/eebe267b3fdd75a82e09cc270b3c046b2c9f2c84/contracts/Token.sol#L382


#### Description:
Using a hardcoded swap path is not advised because it may not be the most optimal available path in terms of fees and liquidity. The swap path must be configurable to avoid illiquid pools (maybe LPs are not incentivised as much or they decided to move elsewhere). A more liquid pool has smaller slippage. 

``` 
function swapTokensForEth(uint256 tokenAmount) private {  
        // generate the uniswap pair path of token -> weth
        address[] memory path = new address[](2);
        path[0] = address(this);
        path[1] = uniswapV2Router.WETH();

        _approve(address(this), address(uniswapV2Router), tokenAmount);

        // make the swap
        uniswapV2Router.swapExactTokensForETHSupportingFeeOnTransferTokens(
            tokenAmount,
            1, // accept any amount of ETH
            path,
            address(this),
            block.timestamp
        );      
    }
    
```

#### Recommendation:
Provide path as a list of trusted pools in order to prevent dependence on a single pool, or fetch it as a function parameter





#### Resolution:

----

### <a id="H02"></a> H-02 Slippage tolerance is very high


https://github.com/poodlTech/tokenAudit/blob/eebe267b3fdd75a82e09cc270b3c046b2c9f2c84/contracts/Token.sol#L390


#### Description:
When using a router to swap tokens, slippage tolerance provides a protection mechanism in case of flash moves in the market values. It is user’s responsibility to give a minimum amount of tokens he is ready to accept. In this protocol, the minAmount has been hardcoded as 1 which means 99 % + slippage allowed in case of large amounts. This will make user lose most of his dividend funds when the pool is imbalanced. 




#### Recommendation:
Provide slippage tolerance as a user parameter instead of forcing the user to take dust amounts. 





#### Resolution:


-----------------



## <a id="Medium"></a> Medium

### <a id="M01"></a> M-01 Uninitialized values in Airdropper.sol


https://github.com/poodlTech/tokenAudit/blob/eebe267b3fdd75a82e09cc270b3c046b2c9f2c84/contracts/Airdropper.sol#L51

#### Description:
The newToken address variable has not been initialized thus it has the default zero value. In the airdrop() and emergencyWithdraw() functions, calling IERC20(newToken).transfer/safetransfer will always revert because transfer functions do not exist at the zero address. So the contract becomes unusable  





#### Recommendation:
Make sure to set the newToken address in the constructor before deploying the contract, provided as a function parameter. 





#### Resolution:

-----------------




### <a id="M02"></a> M-02  Wrong Router Contract address used on Polygon


 https://github.com/poodlTech/tokenAudit/blob/eebe267b3fdd75a82e09cc270b3c046b2c9f2c84/contracts/Token.sol#L81




#### Description:
While enquiring about the chain the token.sol contracts will be deployed, I got to know it is Polygon. The contract address passed as the Uniswap router is 0x10ED43C718714eb63d5aA57B78B54704E256024E, which is actually an EOA on the polygon chain. This address is a Router by PancakeSwap on Binance Smart Chain, but calls to this address on Polygon would revert and make our contract unusable.






#### Recommendation:
Use the correct address of the router on Polygon. Uniswap/ pancakeswap doesn’t have it. 






#### Resolution:

-----------------




### <a id="M03"></a> M-03 Token Pair isn’t excluded from Dividends when updating the DividendTracker contract address

https://github.com/poodlTech/tokenAudit/blob/eebe267b3fdd75a82e09cc270b3c046b2c9f2c84/contracts/Token.sol#L113




#### Description:
When updating the DividendTracker contract address to be interacted with from our token contract, the update function calls excludeFromDividends function to exclude certain addresses from dividends in the new DividendTracker’s storage. While doing this, the uniswapV2Pair address has been left out while as per the L#94 it is meant to be excluded. This breaks the assumption of the pair being excluded from dividends and the dividend amount sent to the pair contract may be stuck.





#### Recommendation:
Add the line newDividendTracker.excludeFromDividends(address(_uniswapV2Pair));






#### Resolution:

-----------------




### <a id="M04"></a> M-04 UniswapV2Pair value isn’t changed when updating the Router contract address

https://github.com/poodlTech/tokenAudit/blob/eebe267b3fdd75a82e09cc270b3c046b2c9f2c84/contracts/Token.sol#L127




#### Description:
When updating the UniswapV2Router contract address to be interacted with from our token contract, the update function needs to change the value of the state variable “_uniswapV2Pair” because the new router will have a different address for the token pair. It leaves out this state update again preventing the uniswapV2Pair to be excluded from Dividends. Also, the new uniswapV2Router address needs to be excluded from Dividends. 





#### Recommendation:
The contract should be consistent in its dependence on the admins calling a function explicitly. While all values are being changed in update functions automatically, the pair address should also be changed and excluded from dividends. 






#### Resolution:

-----------------




### <a id="M05"></a> M-05 Fees cap of 15 % can be bypassed

https://github.com/poodlTech/tokenAudit/blob/eebe267b3fdd75a82e09cc270b3c046b2c9f2c84/contracts/Token.sol#L217




#### Description:
According to the docs, the max fees charged can’t be more than 15 %, but this can be bypassed easily. Here is the most extreme scenario. 
1. Owner sets liquidityFee, rewardsFee and marketingFee to 0. Thus totalFee = 0
2. Owner sets sellTopUp to 12 so it passes the capping check at L#218.
3. Then he sets any of the 3 fees mentioned in 1 to values such that it passes the respective checks in those functions. He can try various combinations in order to benefit the most at the cost of users.
4. Now, totalFee can’t be more than 15 % but the actual realised fee comes out to be 30 % (15 totalFee + 15 sellTopUp)

The user thinks that max fees is 15 % but in reality he is being charged upto 30 %






#### Recommendation:
Either change the user docs to mention this variable fee of upto 30 % or incorporate sellTopUp while checking for the totalFee capping in all the functions. 






#### Resolution:

-----------------




### <a id="M06"></a> M-06 canClaim check should be used in all code paths

https://github.com/poodlTech/tokenAudit/blob/eebe267b3fdd75a82e09cc270b3c046b2c9f2c84/contracts/Token.sol#L601




#### Description:
The DividendTracker contract has a check that allows the user to withdraw dividends only if a certain time threshold, claimWait has passed from the last time he claimed. This check can be bypassed by calling the processAccount() and claim() functions in Token contract. At these two places, the claimWait is not checked.






#### Recommendation:
Use claimWait check from Line 689 of Token.sol in the processAccount() function too. 







#### Resolution:

-----------------




### <a id="M07"></a> M-07 The includeInDividends function should add user back to the tokenHoldersMap

https://github.com/poodlTech/tokenAudit/blob/eebe267b3fdd75a82e09cc270b3c046b2c9f2c84/contracts/Token.sol#L541




#### Description:
The excludeFromDividends() function in DividendTRacker Contract removes the address from token Holders list, while the user is not included back when includedInDividends() is called. 






#### Recommendation:
Make sure the user gets added to the token holders list once his address has been allowed to be included for dividends.






#### Resolution:

-----------------



### <a id="M08"></a> M-08 buyPresale() function of Presale.sol doesn’t work

https://github.com/poodlTech/tokenAudit/blob/eebe267b3fdd75a82e09cc270b3c046b2c9f2c84/contracts/Presale.sol#L51




#### Description:
The buyPresale function makes use of msg.value when it doesnt have a payable state mutability. This function gets called in two ways, one internally when anyone sends eth to the contract, or by a user directly interacting with this function. Either ways, it will revert or will not work because we are wanting to send value to a non-payable function. 





#### Recommendation:
Add payable keyword to the function declaration






#### Resolution:

-----------------




### <a id="M09"></a> M-09 Wrong gasleft() calculations in DividendTRacker.process() function

https://github.com/poodlTech/tokenAudit/blob/eebe267b3fdd75a82e09cc270b3c046b2c9f2c84/contracts/Token.sol#L591




#### Description:
The newGasLeft variable is assigned the current execution gas remaining value using gasleft() at Line 591. The loop-controlling gasLeft variable has been assigned this same value at Line 595 but the gas used in operations from line 592 to 594 have not been recorded. Thus the newGasLeft variable has already got an outdated value. This leads to wrong gas calculation. 






#### Recommendation:
Directly assign gasLeft = gasleft() at Line 595 for accurate gas calculations.






#### Resolution:

-----------------


## <a id="Low"></a> Low


### <a id="L01"></a> L-01 When Uniswap Router is updated, it should be added to ammIsWhitelisted

https://github.com/poodlTech/tokenAudit/blob/eebe267b3fdd75a82e09cc270b3c046b2c9f2c84/contracts/Token.sol#L127



#### Description:
The DividendPayingToken.sol contract L#60 shows that the uniswapV2Router has to be included in the ammIsWhitelisted mapping. While changing the uniswapV2Router in Token.sol, the updated address also needs to be whitelisted for consistency of code. 





#### Recommendation:
The admins(ie. owner address) may manually call approveAMM() by passing the new router address every time the uniswap router contract address is changed in Token.sol, but it is better to  automate it inside updateUniswapV2Router() function in Token.sol#L127





#### Resolution:

____


### <a id="L02"></a> L-02 Use .div instead of / consistently
 
https://github.com/poodlTech/tokenAudit/blob/eebe267b3fdd75a82e09cc270b3c046b2c9f2c84/contracts/DividendPayingToken.sol#L117


#### Description:
he SafeMath library has been imported to use for arithmetic calculations. Line 117 uses / when it should be using .div. Similar issue at DividendPayingToken.sol#L245





#### Recommendation:
Change / to .div 





#### Resolution:

____


### <a id="L03"></a> L-03 Unintialized Values in Presale.sol

https://github.com/poodlTech/tokenAudit/blob/eebe267b3fdd75a82e09cc270b3c046b2c9f2c84/contracts/Presale.sol#L54



#### Description:
The variables token and exchangeratio have not been set to any values. So, the address token will take default zero value and IERC20 functions at line 55 will revert. Also, zero exchangeratio will lead to incorrect calculations. 





#### Recommendation:
Initialize these variables to reasonable values in the constructor or create setter functions for them. 





#### Resolution:

____


### <a id="L04"></a> L-04 Inconsistent deadlines used for moving funds

https://github.com/poodlTech/tokenAudit/blob/eebe267b3fdd75a82e09cc270b3c046b2c9f2c84/contracts/DividendPayingToken.sol#L173



#### Description:
Deadline checks are an important part of any functions that interacts with pools through routers or otherwise. Usually protocols use a deadline of block.timestamp to prevent flash slippages, but at one place in the protocol a different deadline has been used which is unusual and can be detrimental in fast market moves.





#### Recommendation:
Change deadline parameter at Line 173 from block.timestamp + 360 to just block.timestamp.





#### Resolution:

____


### <a id="L05"></a> L-05 newDividendTracker owner needs to be changed

https://github.com/poodlTech/tokenAudit/blob/eebe267b3fdd75a82e09cc270b3c046b2c9f2c84/contracts/Token.sol#L113



#### Description:
When a new Dividend Tracker is deployed, its owner would be the deployer address. The admins should remember to transfertheownership of the newDividendTRacker contract to the Token contract the protocol has. All the functionality depends on owner controlled functions, so this is important.





#### Recommendation:
Change newDividendTracker contract’s owner as soon as it is deployed and added to the Token contract.






#### Resolution:

____


### <a id="QA"></a>  QA 


[1] : There is unused code in Airdropper.sol SafeMath and Address libraries, and Reentrancy functionality should be removed 

[2] : In the emergencyWithdraw() function in Airdropper.sol, explicit conversion of msg.sender to payable is not required because the function only sends an ERC20 token and they can be sent to an address directly. This is also the case at many other places.

[3] : The router used in Token.sol is actually by PancakeSwap and not Uniswap. While it works exactly the same, change the terminology if you want to. 

[4] : The marketingWallet address in Token.sol should be set in the constructor itself, like all other values. Also, the token name and symbol need to be set.

[5] : _transfer function in DividendTracker contract is unnecessary code as it can never be executed. Multiple other similar functions.

[6] : Change all the BNB references to MATIC in Token.sol



____
## Disclaimer
> 
> This report is not, nor should be considered, an “endorsement” or “disapproval” of any particular project or team. This report is not, nor should be considered, an indication of the economics or value of any “product” or “asset” created by any team or project that contracts Solidity Lab to perform a security assessment. This report does not provide any warranty or guarantee regarding the absolute bug-free nature of the technology analyzed, nor do they provide any indication of the technologies proprietors, business, business model or legal compliance.
> 
> This report should not be used in any way to make decisions around investment or involvement with any particular project. This report in no way provides investment advice, nor should be leveraged as investment advice of any sort. This report represents an extensive assessing process intending to help our customers increase the quality of their code while reducing the high level of risk presented by cryptographic tokens and blockchain technology.
> 
> Blockchain technology and cryptographic assets present a high level of ongoing risk. Solidity Lab’s position is that each company and individual are responsible for their own due diligence and continuous security. Solidity Lab’s goal is to help reduce the attack vectors and the high level of variance associated with utilizing new and consistently changing technologies, and in no way claims any guarantee of security or functionality of the technology we agree to analyze.
> 
> The assessment services provided by Solidity Lab is subject to dependencies and under continuing
> development. You agree that your access and/or use, including but not limited to any services, reports, and materials, will be at your sole risk on an as-is, where-is, and as-available basis. Cryptographic tokens are emergent technologies and carry with them high levels of technical risk and uncertainty. The assessment reports could include false positives, false negatives, and other unpredictable results. The services may access, and depend upon, multiple layers of third-parties.
> 
> Notice that smart contracts deployed on the blockchain are not resistant from internal/external exploit. Notice that active smart contract owner privileges constitute an elevated impact to any smart contract’s safety and security. Therefore, Solidity Lab does not guarantee the explicit security of the audited smart contract, regardless of the verdict.

<br/>

____

## About

I'm a smart contract auditor who does Solidity code reviews. This security review was done for a client firm of the Guardian Labs. 

DM for audits and enquiries : 

- Twitter - [@dev_chinmayf](https://twitter.com/dev_chinmayf)
- Discord - [Chinmay#4134](https://discordapp.com/users/732959289139789875)
