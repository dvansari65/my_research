## Solidity smart contract gas optimisation best practices

### Most important: avoid zero to one storage writes where possible

Initializing a storage variable is one of the most expensive operations a contract can do.

When a storage variable goes from zero to non-zero, the user must pay 22,100 gas total (20,000 gas for a zero to non-zero write and 2,100 for a cold storage access).

This is why the OpenZeppelin reentrancy guard registers functions as active or not with 1 and 2 rather than 0 and 1. It only costs 5,000 gas to alter a storage variable from non-zero to non-zero.


###  Cache storage variables: write and read storage variables exactly once

You will see the following pattern frequently in efficient solidity code. Reading from a storage variable costs at least 100 gas as Solidity does not cache the storage read. Writes are considerably more expensive. Therefore, you should manually cache the variable to do exactly one storage read and exactly one storage write.

```solidity
// SPDX-License-Identifier: MIT
pragma solidity 0.8.20;

contract Counter1 {
    uint256 public number;

    function increment() public {
        require(number < 10);
        number = number + 1;
    }
}

contract Counter2 {
    uint256 public number;

    function increment() public {
        uint256 _number = number;
        require(_number < 10);
        number = _number + 1;
    }
}
```
The first function reads counter twice, the second code reads it once.

### Pack related variables
Packing related variables into same slot reduces gas costs by minimizing costly storage related operations.

#### Manual packing is the most efficient
We store and retrieve two uint80 values in one variable (uint160) by using bit shifting. This will use only one storage slot and is cheaper when storing or reading the individual values in a single transaction.

```solidity
// SPDX-License-Identifier: MIT
pragma solidity 0.8.20;

contract GasSavingExample {
    uint160 public packedVariables;

    function packVariables(uint80 x, uint80 y) external {
        packedVariables = uint160(x) << 80 | uint160(y);
    }

    function unpackVariables() external view returns (uint80, uint80) {
        uint80 x = uint80(packedVariables >> 80);
        uint80 y = uint80(packedVariables);
        return (x, y);
    }
}
```

##  Pack structs
Packing struct items, like packing related state variables, can help save gas.
(It is important to note that in Solidity, struct members are stored sequentially in the contract’s storage, starting from the slot position where they are initialized).

Consider the following examples:

### Unpacked Struct
The unpackedStruct has three items which will be stored in three seperate slot. However, if this items were packed, only two slots would be used and this will make reading and writing to the struct’s items cheaper.

```solidity
// SPDX-License-Identifier: MIT
pragma solidity 0.8.20;

contract Unpacked_Struct {
    struct unpackedStruct {
        uint64 time; // Takes one slot - although it  only uses 64 bits (8 bytes) out of 256 bits (32 bytes).
        uint256 money; // This will take a new slot because it is a complete 256 bits (32 bytes) value and thus cannot be packed with the previous value.
        address person; // An address occupies only 160 bits (20 bytes).
    }

    // Starts at slot 0
    unpackedStruct details = unpackedStruct(53_000, 21_000, address(0xdeadbeef));

    function unpack() external view returns (unpackedStruct memory) {
        return details;
    }
}
```

### Packed Struct
We can make the example above use less gas by packing the struct items like this.

```solidity
contract Packed_Struct {
    struct packedStruct {
        uint64 time; // In this case, both `time` (64 bits) and `person` (160 bits) are packed in the same slot since they can both fit into 256 bits (32 bytes)
        address person; // Same slot as `time`. Together they occupy 224 bits (28 bytes) out of 256 bits (32 bytes).
        uint256 money; // This will take a new slot because it is a complete 256 bits (32 bytes) value and thus cannot be packed with the previous value.
    }

    // Starts at slot 0
    packedStruct details = packedStruct(53_000, address(0xdeadbeef), 21_000);

    function unpack() external view returns (packedStruct memory) {
        return details;
    }
}
```