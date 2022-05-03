## Solution  
### 1. [Unstoppable challenge](https://www.damnvulnerabledefi.xyz/challenges/1.html)
> As described in the challenge description, the goal of this challenge is to stop the pool from offering flash loans.  
#### Train of thoughts:
After looking at the code, we can easily find out that the flash loan function requires a token deposit in order to be executed. It is an async workflow of two steps:
- deposit at least one token in the smart contract pool
- call flash loan
```solidity
//step 1: deposit at least one token
function depositTokens(uint256 amount) external nonReentrant {
    require(amount > 0, "Must deposit at least one token");
    // Transfer token from sender. Sender must have first approved them.
    damnValuableToken.transferFrom(msg.sender, address(this), amount);
    poolBalance = poolBalance + amount;
}

//step 2: call flash loan
function flashLoan(uint256 borrowAmount) external nonReentrant {
    require(borrowAmount > 0, "Must borrow at least one token");

    //check if deposit has been done before continuing any further
    uint256 balanceBefore = damnValuableToken.balanceOf(address(this));
    require(balanceBefore >= borrowAmount, "Not enough tokens in pool");

    //ensured by the protocol via the `depositTokens` function
    assert(poolBalance == balanceBefore);

    //T money from pool to borrower
    damnValuableToken.transfer(msg.sender, borrowAmount);

    //calback flash loan: T money from borrower back to pool
    //create instance lender with this SC address and borrowed amount
    IReceiver(msg.sender).receiveTokens(address(damnValuableToken), borrowAmount);

    //check the balance and check if previous call worked
    uint256 balanceAfter = damnValuableToken.balanceOf(address(this));
    require(balanceAfter >= balanceBefore, "Flash loan hasn't been paid back");
}
```
The condition which verifies step 1 from being called beforehand is checked here:
```solidity
assert(poolBalance == balanceBefore);
```
- poolBalance is the variable set by the depositTokens function
- balanceBefore is the smart contract current balance

Anyone could change the smart contract address balance by transferring some tokens. It's not a variable that should be used as a source of trust.
If an attacker wants to stop the pool from offering flash loans, it is then as trivial as sending some tokens to the smart contract address.
We could imagine a smart contract implementing this single line of code:
```solidity
await this.token.transfer(<<smart_contract_address>>, 0.0001);
```
#### Implementation:
It can be found [here](https://github.com/v4lproik/damn-vulnerable-defi/blob/master/test/unstoppable/unstoppable.challenge.js).


