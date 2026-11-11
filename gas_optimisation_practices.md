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

## Keep strings smaller than 32 bytes
In Solidity, strings are variable length dynamic data types, meaning their length can change and grow as needed.

If the length is 32 bytes or longer, the slot in which they are defined stores the length of the string * 2 + 1, while their actual data is stored elsewhere (the keccak hash of that slot).

However, if a string is less than 32 bytes, the length * 2 is stored at the least significant byte of it’s storage slot and the actual data of the string is stored starting from the most significant byte in the slot in which it is defined.

## Variables that are never updated should be immutable or constant
In Solidity, variables which are not intended to be updated should be constant or immutable.

This is because constants and immutable values are embedded directly into the bytecode of the contract which they are defined and does not use storage because of this.

```solidity 
// SPDX-License-Identifier: MIT
pragma solidity 0.8.20;

contract Constants {
    uint256 constant MAX_UINT256 = 0xffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffff;

    function get_max_value() external pure returns (uint256) {
        return MAX_UINT256;
    }
}

// This uses more gas than the above contract
contract NoConstants {
    uint256 MAX_UINT256 = 0xffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffff;

    function get_max_value() external view returns (uint256) {
        return MAX_UINT256;
    }
}
```
This saves a lot of gas as we do not make any storage reads which are costly.

## Using mappings instead of arrays to avoid length checks

Solidity arrays perform an extra bounds check whenever you access an element, whereas mappings don't. If your application already guarantees that an index is valid, a mapping can sometimes save gas.

```solidity
/// get(0) gas cost: 4860
contract Array {
    uint256[] a;

    constructor() {
        a.push() = 1;
        a.push() = 2;
        a.push() = 3;
    }

    function get(uint256 index) external view returns(uint256) {
        return a[index];
    }
}

/// get(0) gas cost: 2758
contract Mapping {
    mapping(uint256 => uint256) a;

    constructor() {
        a[0] = 1;
        a[1] = 2;
        a[2] = 3;
    }

    function get(uint256 index) external view returns(uint256) {
        return a[index];
    }
}
```
Just by using a mapping, we get a gas saving of 2102 gas. Why? Under the hood when you read the value of an index of an array, solidity adds bytecode that checks that you are reading from a valid index (i.e. an index strictly less than the length of the array) else it reverts with a panic error (Panic(0x32) to be precise). This prevents it from reading unallocated or worse, allocated storage/memory locations.

Due to the way mappings are (simply a key => value pair), no check like that exists and we are able to read from the a storage slot directly. It’s important to note that when using mappings in this manner, your code should ensure that you are not reading an out of bound index of your canonical array.