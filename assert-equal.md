# QuillCTF : assertEqual

## Objective of CTF
```
You need to write a smart contract that accepts two unsigned integers as inputs. The contract should return 1 if the input numbers are equal; otherwise, it should return a different number.
```

We need to write the smart contract opcode by opcode by hand, also many opcode are banned

It does not just ban the opcode, it just run a for loop over the creation code to check for each byte if the byte is a badOpcode, so even the byte is argument of PUSH, it is also banned

We will write a smart contract that check if 2 unsigned integers are equal, if they are equal then return 1, otherwise just return 0

However, there are many opcodes that we can't use, like we can't use EQ to compare them directly

In the test file it send 4 wei as value when calling our smart contract, so we can use CALLVALUE to push 4 to the stack as 0x04 byte is banned, but we need it because of the 4 bytes function signature

Also, it is forking mainnet which has a chain id of 1, we can use CHAINID to push 1 to the stack as 0x01 byte is also banned

My approach to solving this without using the banned opcodes is by writing data to a storage slot, and check if the storage slot has been written, 

For example, it is checking if 1 equals 2, then it will write some data to storage slot 1, and it checks if storage slot 2 is written to see if they are equal 

runtime code :
```
CHAINID
CALLVALUE
CALLDATALOAD
SSTORE
PUSH1 0x24
CALLDATALOAD
SLOAD
ISZERO
ISZERO
PUSH0
MSTORE
PUSH1 0x20
PUSH0
RETURN

463435556024355415155f5260205ff3
```

Because the size of the original runtime code is in badOpcodes, so I just pad it with some 0x00 so that the size is not in badOpcodes

creation code :
```
PUSH21 0x463435556024355415155f5260205ff30000000000
PUSH0
MSTORE
PUSH1 0x15
PUSH1 0x0b
RETURN

74463435556024355415155f5260205ff300000000005f526015600bf3
```

Somehow my foundry revert when it reached the PUSH0 opcode, I guess maybe it's not supported yet, so I just rewrite them without the PUSH0 opcode

runtime code without PUSH0 :
```
CHAINID
CALLVALUE
CALLDATALOAD
SSTORE
PUSH1 0x24
CALLDATALOAD
SLOAD
ISZERO
ISZERO
PUSH1 0x00
MSTORE
PUSH1 0x20
PUSH1 0x00
RETURN

4634355560243554151560005260206000f3
```

creation code without PUSH0 :
```
PUSH21 0x4634355560243554151560005260206000f3000000
PUSH1 0x00
MSTORE
PUSH1 0x15
PUSH1 0x0b
RETURN

744634355560243554151560005260206000f30000006000526015600bf3
```

## Proof of concept

### Foundry test

```solidity
// SPDX-License-Identifier: MIT
pragma solidity 0.8.7;

import "forge-std/Test.sol";

contract EQ is Test {
    address isNumbersEQContract;
    bytes1[] badOpcodes;

    function setUp() public {
        badOpcodes.push(hex"01");
        badOpcodes.push(hex"02"); // MUL
        badOpcodes.push(hex"03"); // SUB
        badOpcodes.push(hex"04"); // DIV
        badOpcodes.push(hex"05"); // SDIV
        badOpcodes.push(hex"06"); // MOD
        badOpcodes.push(hex"07"); // SMOD
        badOpcodes.push(hex"08"); // ADDMOD
        badOpcodes.push(hex"09"); // MULLMOD
        badOpcodes.push(hex"18"); // XOR
        badOpcodes.push(hex"10"); // LT
        badOpcodes.push(hex"11"); // GT
        badOpcodes.push(hex"12"); // SLT
        badOpcodes.push(hex"13"); // SGT
        badOpcodes.push(hex"14"); // EQ
        badOpcodes.push(hex"f0"); // create
        badOpcodes.push(hex"f5"); // create2
        badOpcodes.push(hex"19"); // NOT
        badOpcodes.push(hex"1b"); // SHL
        badOpcodes.push(hex"1c"); // SHR
        badOpcodes.push(hex"1d"); // SAR
        vm.createSelectFork(
            "https://rpc.ankr.com/eth"
        );
        address isNumbersEQContractTemp;
        // solution - your bytecode
        bytes
            memory bytecode = hex"744634355560243554151560005260206000f30000006000526015600bf3";
        //
        require(bytecode.length < 40, "try harder!");
        for (uint i; i < bytecode.length; i++) {
            for (uint a; a < badOpcodes.length; a++) {
                if (bytecode[i] == badOpcodes[a]) revert();
            }
        }

        assembly {
            isNumbersEQContractTemp := create(
                0,
                add(bytecode, 0x20),
                mload(bytecode)
            )
            if iszero(extcodesize(isNumbersEQContractTemp)) {
                revert(0, 0)
            }
        }
        isNumbersEQContract = isNumbersEQContractTemp;
    }

    // fuzzing test
    function test_isNumbersEq(uint8 a, uint8 b) public {
        (bool success, bytes memory data) = isNumbersEQContract.call{value: 4}(
            abi.encodeWithSignature("isEq(uint256, uint256)", a, b)
        );
        require(success, "!success");
        uint result = abi.decode(data, (uint));
        a == b ? assert(result == 1) : assert(result != 1);

        // additional tests
        // 1 - equal numbers
        (, data) = isNumbersEQContract.call{value: 4}(
            abi.encodeWithSignature("isEq(uint256, uint256)", 57204, 57204)
        );
        require(abi.decode(data, (uint)) == 1, "1 test fail");
        // 2 - different numbers
        (, data) = isNumbersEQContract.call{value: 4}(
            abi.encodeWithSignature("isEq(uint256, uint256)", 0, 3568)
        );
        require(abi.decode(data, (uint)) != 1, "2 test fail");
    }
}
```

### Foundry test output

```
# forge test --match-path test/assertEqual.t.sol -vv
[⠃] Compiling...
No files changed, compilation skipped

Running 1 test for test/assertEqual.t.sol:EQ
[PASS] test_isNumbersEq(uint8,uint8) (runs: 256, μ: 96630, ~: 99033)
Test result: ok. 1 passed; 0 failed; finished in 730.25ms
```