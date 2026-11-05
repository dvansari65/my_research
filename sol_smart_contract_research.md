# Solidity smart contract best practices and vulnerable code 

## Ownership and Access Control Pattern

### Not everyone allowed to :

1. Mint tokens
2. Upgrade contracts
3. Pause systems
4. Withdraw funds

- Hardcoding msg.sender checks everywhere quickly becomes messy and dangerous.
```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

contract AccessControlled {
    address public owner;

    modifier onlyOwner() {
        require(msg.sender == owner, "Not authorized");
        _;
    }

    constructor() {
        owner = msg.sender;
    }

    function changeOwner(address newOwner) external onlyOwner {
        owner = newOwner;
    }
}
```
- In this code, ownership logic is centralized, predictable, and easy to audit.
- But in production , use OpenZepelin AccessControl instead of rolling your own complex systems.

## Checks-Effects-Interactions Pattern

### The Problem
- Reentrancy attacks remains one of the most dangerous vulnerabilities in solidity.

### The Pattern
Always follow this pattern:

1. Check the conditions
2. Update the states
3. Call external smart contracts

- Vulnerable code 
```solidity
function withdraw() external{
    uint amount = balances[msg.sender];
    require(amount > 0,"No funds!");
    (bool success) = msg.sender.call{value:amount}(");
    require(success,"transfer failed");
    balances[msg.sender] = 0;
}
```
- Correct code
```solidity
function withdraw()external {
    uint amount = balances[msg.sender];
    require(amount > "No funds");
    balances[msg.sender] = 0;
    (bool  success) = msg.sender.call{value:amount}("");
    require(success,"transfer failed");
}
```
## Pull Over Push Payments Pattern

### The Problem
- Sending payments automatically can fail:

1. Gas issues 
2. Reverting Recievers
3. Unexpexted contract logic

### The Pattern
- Let users withdraw funds themselves

- Correct Practice
```solidity
mapping(address -> uint256) public pendingWithdrawels;
function credit(address user,uint amount) internal {
    pendingWithdrawels[user] += amount
}
function withdraw() external {
    uint amount = pendingWithdrawels[msg.sender];
    require(amount > 0 ,"No funds");
    pendingWithdrawels[msg.sender] = 0;
    (bool success) = msg.sender.call{value:amount}("");
    require(success,"Transfer failed!")
}
```

## Upgradeability Pattern (Proxy Contracts)

### The Problem
- Smart contract are immutable but software evolves

### The Pattern
Seperate:
1. Storage ( Proxy )
2. Logic ( Implementation )

- Simplified Proxy Structure
```solidity
contract Proxy {
    address public implementation;

    function upgrade(address newImplementation) external {
        implementation = newImplementation;
    }

    fallback() external payable {
        address impl = implementation;
        assembly {
            delegatecall(gas(), impl, 0, calldatasize(), 0, 0)
            returndatacopy(0, 0, returndatasize())
            return(0, returndatasize())
        }
    }
}
```
#### But Best Practice is :

Use UUPS or Transparent Proxy patterns via OpenZeppelin.
Never write raw proxy logic in production unless you deeply understand storage layouts.

## Pausable Circuit Breaker Pattern

### The Problem
- When exploits happen, minutes matter.

### The Pattern
- Allow authorised roles to pause critical functions

```solidity
bool public paused;

modifier whenNotPaused() {
    require(!paused, "Paused");
    _;
}

function pause() external onlyOwner {
    paused = true;
}

function unpause() external onlyOwner {
    paused = false;
}
```
## Factory Pattern

### The Problem
Deploying multiple similar contracts manually is inefficient and error-prone.

### The Pattern
Use a factory contract to create standardized instances.

#### solidity example

```solidity
contract Wallet {
    address public owner;

    constructor(address _owner) {
        owner = _owner;
    }
}

contract WalletFactory {
    Wallet[] public wallets;

    function createWallet() external {
        Wallet wallet = new Wallet(msg.sender);
        wallets.push(wallet);
    }
}
```
## Gas Limit and Unbounded Loops

### The Problem
- Every transaction has a limited gas budget 
- Loops consume gas on every iteration
- If a loop depends on dynamically growing array , its execution cost can increase over time
- Eventually , loop may need more gas than transaction can afford to execute it 

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

contract Payment {
    address[] public users;

    function payAll() external {
        for (uint256 i = 0; i < users.length; i++) {
            payable(users[i]).transfer(0.01 ether);
        }
    }
```
- it will work for small number of users
- if number of users become large then the loops requires more gas 
- eventually , payAll() becomes impossible to execute withing the gas limit

### The Pattern
#### Avoid processing unlimited number of records in a single transaction

- Bounded loops with maximum iteration count
- Batch processing to divide the work across multiple transactions
- Pull payments so users withdraw funds independently


## `msg.sender` vs `tx.origin` (Authorization Vulnerability)

### What is `msg.sender`?

-   `msg.sender` is the address of the account or contract that directly
    called the current contract.
-   It represents the **immediate caller**, not necessarily the person
    who initiated the transaction.
-   Its value can change at every external contract call.

### What is `tx.origin`?

-   `tx.origin` is the address of the original EOA that initiated the
    transaction.
-   It remains the same throughout the transaction's entire call chain.

### Example

Consider this call chain:

``` text
Alice (EOA)
    |
    v
TxUserWallet
    |
    v
TxAttackWallet
    |
    v
TxUserWallet
```

When `TxAttackWallet` calls `TxUserWallet.transferTo()`:

``` solidity
msg.sender == TxAttackWallet;
tx.origin == Alice;
```

Why?

-   `msg.sender` is `TxAttackWallet` because it directly called
    `transferTo()`.
-   `tx.origin` is Alice because she initiated the original transaction.

### The Vulnerability

Never use `tx.origin` for authorization.

**Vulnerable code:**

``` solidity
require(tx.origin == owner);
```

A malicious contract can trick the owner into interacting with it. It
can then call the owner's wallet contract again.

Since `tx.origin` still contains the owner's address, the authorization
check passes, allowing the malicious contract to misuse the owner's
permissions.

**Correct approach:**

``` solidity
require(msg.sender == owner, "Not authorized");
```

Now the check verifies the immediate caller rather than the original
transaction initiator.

### Key Takeaways

-   `msg.sender` → Who directly called this contract?
-   `tx.origin` → Who initiated the original transaction?
-   `msg.sender` can change across external calls.
-   `tx.origin` remains the original EOA throughout the transaction.
-   Use `msg.sender` for access control, never `tx.origin`.

## Integer Overflow, Underflow & Two's Complement

Solidity integers have fixed ranges and cannot store unlimited values.

- **Overflow:** When a value exceeds the maximum limit of its integer type.
- **Underflow:** When a value goes below the minimum limit.

### Example

```solidity
uint8 x = 255;
uint8 y = 1;

uint8 result = x + y; // Reverts in Solidity 0.8+
```

`uint8` can store values from `0` to `255`. Adding `1` to `255` causes an overflow.

### Checked vs Unchecked Arithmetic

- **Checked (default):** Solidity 0.8.0+ automatically detects overflow and underflow and reverts.
- **Unchecked:** Disables these checks and allows values to wrap around.

```solidity
unchecked {
    uint8 result = 255 + 1; // 0
}
```

### Two's Complement

Solidity uses two's complement to represent signed integers (`int8`, `int256`, etc.).

For example, `int8` ranges from `-128` to `127`. Negating its minimum value (`-128`) causes an overflow because `128` cannot be represented by `int8`.

### Security Best Practices

- Validate user inputs using `require`.
- Avoid `unchecked` unless overflow and underflow are mathematically impossible.
- Pay special attention to balance, fee, and reward calculations.
- Use Solidity's SMTChecker to identify potential arithmetic issues.
- Remember that Solidity versions before 0.8.0 require additional overflow protection, commonly through SafeMath.

## Clearing Mappings

Solidity mappings do not keep track of their stored keys. Therefore, Solidity cannot automatically clear all mapping entries.

**Important:** Deleting an array or struct containing a mapping does not delete the mapping's stored values.

### Example

```solidity
mapping(uint => uint)[] public array;

function eraseMaps() public {
    delete array;
}
```

If `array[4][128]` contains `256`, deleting `array` resets its length to zero, but the mapping data remains in storage.

If the array is allocated again, the old mapping value may become accessible:

```solidity
array[4][128]; // Returns 256
```

### Security Best Practices

- Do not assume `delete` clears nested mappings.
- Track mapping keys separately if you need to delete individual entries.
- Use an iterable mapping pattern to maintain a list of keys and delete their values.
- Consider using a unique ID or mapping generation/version counter to logically invalidate old entries instead of deleting every key.

**Key takeaway:** Deleting a storage array or struct does not automatically clear mappings inside it. Old mapping values can become accessible again.

## Denial of Service

A function that transfers funds to a third party before completing the user's action can be blocked by that third party. If the transfer reverts, the whole function reverts.

**Important:** Sending to the owner inside a user-facing function makes every user depend on the owner's address accepting tokens.

### Example

Putty V2 (Code4rena, June 2022). `withdraw` sends the owner's fee first, then the user's strike:

```solidity
function withdraw(Order memory order) public {
    ..SNIP..
    if (fee > 0) {
        feeAmount = (order.strike * fee) / 1000;
        ERC20(order.baseAsset).safeTransfer(owner(), feeAmount);
    }
    ERC20(order.baseAsset).safeTransfer(msg.sender, order.strike - feeAmount);
    ..SNIP..
}
```

If the fee transfer reverts, the user's transfer never runs. The owner can cause this by setting `owner()` to the zero address (OpenZeppelin ERC20 reverts on transfer to zero) or, for an ERC777 base asset, by pointing `owner()` at a contract whose `tokensReceived` hook reverts. No user can withdraw their strike.

### Fix

Accumulate the fee in a mapping and let the owner pull it separately:

```solidity
mapping(address => uint256) public ownerFees;

function withdraw(Order memory order) public {
    ..SNIP..
    if (fee > 0) {
        feeAmount = (order.strike * fee) / 1000;
        ownerFees[order.baseAsset] += feeAmount;
    }
    ERC20(order.baseAsset).safeTransfer(msg.sender, order.strike - feeAmount);
    ..SNIP..
}

function withdrawFee(address baseAsset) public onlyOwner {
    uint256 _feeAmount = ownerFees[baseAsset];
    ownerFees[baseAsset] = 0;
    ERC20(baseAsset).safeTransfer(owner(), _feeAmount);
}
```

### Security Best Practices

- Do not transfer to a third party inside a user-facing function.
- Record what each party is owed in a mapping and let them withdraw it.
- Do not assume `owner()` or any recipient can always receive tokens.

**Key takeaway:** A failed transfer to one recipient should never block another user's withdrawal. Use the withdrawal pattern.

Source: [Code4rena finding](https://github.com/code-423n4/2022-06-putty-findings) · [PuttyV2.sol#L500](https://github.com/code-423n4/2022-06-putty/blob/3b6b844bc39e897bd0bbb69897f2deff12dc3893/contracts/src/PuttyV2.sol#L500)

## Denial of service 
Blacklisted creditor can block all repayment besides emergency closure Real Wagmi

```solidity 
 address creditor = underlyingPositionManager.ownerOf(loan.tokenId);
        // Increase liquidity and transfer liquidity owner reward
        _increaseLiquidity(cache.saleToken, cache.holdToken, loan, amount0, amount1);
        uint256 liquidityOwnerReward = FullMath.mulDiv(
            params.totalfeesOwed,
            cache.holdTokenDebt,
            params.totalBorrowedAmount
        ) / Constants.COLLATERAL_BALANCE_PRECISION;

Vault(VAULT_ADDRESS).transferToken(cache.holdToken, creditor, liquidityOwnerReward);
```

### Problem
Tokens like USDC/USDT supports blacklisting. Which prevents the users from sending or recieving tokens.
If creditor blacklisted by USDC/USDT then fee transfer will always revert. This will cause all repayments to revert, preventing the user from ever repaying their loan and forcing them to default.

## Recommendation approach
Create an escrow to hold funds in the event that the creditor cannot receive their funds. Implement a try-catch block around the transfer to the creditor. If it fails then send the funds instead to an escrow account, allowing the creditor to claim their tokens later and for the transaction to complete.

Source: ( https://solodit.cyfrin.io/issues/m-4-blacklisted-creditor-can-block-all-repayment-besides-emergency-closure-sherlock-real-wagmi-2-git )


## Unhandled Oracle Revert
A direct call to an external price feed can revert. If nothing handles the revert then every function that depends on the price
feed reverts too.

**important:** Chainlink's multisig signing can block access to a price feed at any time. A direct call to 'latestRoundData()' then reverts

### Example

Juicebox V2 (Code4rena, July 2022). `JBChainlinkV3PriceFeed` calls Chainlink directly:

```solidity
function currentPrice(uint256 _decimals) external view override returns (uint256) {
  // Get the latest round information. Only need the price is needed.
  (, int256 _price, , , ) = feed.latestRoundData();
```

`JBPrices` relies on it for every price query:

```solidity
if (_feed != IJBPriceFeed(address(0))) return _feed.currentPrice(_decimals);
```

If `latestRoundData()` reverts, no price can be queried. Feeds cannot be changed after they are configured, so the denial of service is permanent.

### Fix

Surround the call to `latestRoundData()` with `try/catch` instead of calling it directly. If the call reverts, the catch block can call a fallback oracle or handle the error another way.

### Security Best Practices

- Query external price feeds with `try/catch` so the calling contract stays in control.
- Have a fallback for when the feed fails.
- Allow the feed address to be updated after it is configured.

**Key takeaway:** An external oracle call that reverts should not be able to lock your contract. Handle the failure explicitly.

Source: [Solodit: M-09 Unhandled chainlink revert](https://solodit.cyfrin.io/issues/m-09-unhandled-chainlink-revert-would-lock-all-price-oracle-access-code4rena-juicebox-juicebox-v2-contest-git) · [Code4rena report](https://code4rena.com/contests/2022-07-juicebox-v2-contest) · [JBPrices.sol#L69](https://github.com/jbx-protocol/juice-contracts-v2-code4rena/blob/828bf2f3e719873daa08081cfa0d0a6deaa5ace5/contracts/JBPrices.sol#L69) · [JBChainlinkV3PriceFeed.sol#L42-L44](https://github.com/jbx-protocol/juice-contracts-v2-code4rena/blob/828bf2f3e719873daa08081cfa0d0a6deaa5ace5/contracts/JBChainlinkV3PriceFeed.sol#L42-L44)

