# Banana_Repo_Project

Smart contract code for processing repos (repurchase agreements). Basic use:

1. Users deposit some currency token A.
2. Someone can then sell repo token B for A, with an option to buy back B within some time frame (repoTimeLength) in the code.
3. If they buy it back, they pay a bit more than they sold it for, which goes as profit for depositors.
4. If they don't buy it back, depositors can claim the currency B sold in the repo.

## Parameters ##

### Constants ###  

* (address) currencyToken - set to the token you want as currency for your repos. ETH / USDC / DAI / USDT make sense.
* (address) repoToken - set to the token you to be able to sell and buy back repos for.
* (uint256) PRECISION - important for division properties. For example, let’s say we were doing OHM and USDC. Then to set 1 OHM = 1 USDC, you actually need to set 1000000000 OHM (9 decimal places) = 1000000 USDC (6 decimal places), or 1 OHM = 0.001 USDC, which you cannot do. So instead we set prices as number of USDC for 10^18 OHM, and then divide by this value later (10^18).

### Storage ###

* (mapping: address => uint256) userCurrencyBalances - stores how much each user has deposited (only includes cleared amounts)
* (array) clearedDepositorsList - a list of each depositor. Depositors are removed when they withdraw their balance. Used whenever we need to iterate through all the depositors (i.e. when calculating new balances on repo repayments or repo defaults), as you can’t iterate through a mapping (i.e. userCurrencyBalances)
* (uint256) totalClearedBalance - sum of all balances in userCurrencyBalances. Storing as a variable instead of calculating whenever needed, as a gas consideration
* (uint256) totalEligibleBalance - totalClearedBalance except removes any supply that’s been borrowed out already. Used to calculate how much can still be used to buy repos
* (mapping: address => uint256) userDefaultBalances - how much each address is owed in repoToken, 
* (mapping: address => uint256) pendingDeposits -  stores a mapping of all deposits that are pending. Deposits are pending for pendingTime before clearing and are able to be used in repo transactions. 
* (array) pendingAddressList - a list of each depositor whose deposit is pending. Once the deposit is cleared, the user is removed from this list. Users can appear in one or both pendingDeposits and clearedDepositorsList. Also used for iterating
* (mapping address => repo) activeRepos - stores all active repos
* (array) reposUsersList - not necessary, but could be useful or debugging by seeing how many active repos there are
* (boolean) reposPaused - stores whether repos are currently paused for deposits
* (uint256) repoSellPrice - number of currencyTokens you can sell 1e18 repoTokens for
* (uint256) repoBuybackPrice - number of currencyTokens required to buy back 1e18 repoTokens
* (uint256) repoTimeLength - time in seconds it will take for repo to expire  
* (uint256) pendingTime - time in seconds it will take for a pending deposit to be eligible to be cleared for repo transactions

## Design Considerations ## 

* When you deposit, your deposit will not be eligible to participate in repos until pendingTime seconds in the future. This is set to prevent bots from depositing right before a repo is sold and withdrawing right after it is bought back to earn all the premium that is repaid.
* If you deposit and you have previous pending deposits whose expiration time has not yet passed, your deposited amount will be added to the pending balance and the activation time will reset for the whole balance. THEY WILL NOT BE CONSIDERED TWO DIFFERENT DEPOSITS WITH TWO DIFFERENT DEPOSIT TIMES.
* However, every time you deposit, the code will first clear any pending deposits of yours whose activationTime have passed automatically. So if you previously deposited and the activationTime has already passed, even if the deposit has not been cleared yet in a transaction, when you deposit again it will first be cleared.
* All deposits that are cleared when a repo is bought back will split any premium. The important nuance here is you must be cleared at time of repayment, not time the repo is sold. Therefore the only way to guarantee participation in the repo is if your deposit's activation time passes before the repo is initiated; if the activation time is after, it is still possible but not guaranteed that your deposit will participate, depending on when the seller repurchases the repo.
* Whenever a repo is initiated or bought back, all pending deposits whose activation time have passed will be cleared automatically in the code.
* You can withdraw before activationTime has passed on a pending deposit; the activation time only applies to if the money can be used in repos.
* If you have both pending and active balances, and withdraw, the balance will first be taken out of your pending balance - LIFO.
* Any time profits from a repo sell and buyback or a repo default are split, they are split proportionately across all cleared balances. All profits are rounded down in division by default in Solidity. Any excess will be considered protocol profits.
