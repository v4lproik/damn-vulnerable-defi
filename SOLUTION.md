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
#### [Click here for code implementation](https://github.com/v4lproik/damn-vulnerable-defi/blob/master/test/unstoppable/unstoppable.challenge.js)

### 2. [Naive receiver](https://www.damnvulnerabledefi.xyz/challenges/2.html)
> As described in the challenge description, the goal of this challenge is to drain all the ETH funds from the user's contract.
#### _Train of thoughts:
After looking at the code, it seems to be a pretty straightforward synchronous workflow which involves two smart contracts:
--> flashLoan -> receiveEther.
Let's look at the two parameters of the entry point function of our workflow.
```solidity
function flashLoan(address borrower, uint256 borrowAmount)
```
There's no check/condition related to the variable borrower. A contract that allows any address should immediately draw your attention.
```solidity
require(balanceBefore >= borrowAmount, "Not enough ETH in pool");
```
There's only one condition related to the variable borrowAmount. It only has to be equal or bigger than the balance of the smart contract address. As it is an unsigned integer, no negative value can be accepted, obviously. However there's no condition rejecting 0 as a value. Slightly uncommon for a loan...

Back to reading the code, flashLoad then calls receiveEther passing the amount of eth asked by the borrower as well as the value of the fixed fee that needs to be paid by the borrower once the flash loan gets paid back. Same as before, let's look at the entry points of the second smart contract function and see how they are manipulated.
```solidity
function receiveEther(uint256 fee) public payable
```
This smart contract cannot be called externally and the only condition being made is:
```solidity
require(msg.sender == pool, "Sender must be pool");
```
Basically anyone can be a pool, there's no proper check towards who specifically made the call. Then the function triggers the business logic related to the flash load and calls back the flashLoan function calculating the amount of eth needed to be paid back following this principle:
```solidity
uint256 amountToBeRepaid = msg.value + fee;
```
Once again, still no condition regarding the amount being borrowed. To sum it all up we can borrow 0 eth and there's an authorization flaw between the smart contracts call. An attacker can create an Ethereum wallet with 0 eth and then simply calls the flashLoan function with the victim's wallet address and a borrow amount of 0 eth. The transaction fee - constant fee of 1 eth - will then be returned to the flashLoan function allowing the attacker to drain 1 eth per flash load call.
#### [Click here for code implementation](https://github.com/v4lproik/damn-vulnerable-defi/blob/master/test/naive-receiver/naive-receiver.challenge.js)
