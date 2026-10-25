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