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
