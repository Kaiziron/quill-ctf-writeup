# QuillCTF : Gate

## Objective of CTF

You need to set the opened flag to true via the open function

You need to handwrite the bytecode opcode by opcode and stay within the size of less than 33 bytes

### Contract code
```solidity
pragma solidity ^0.8.17;

interface IGuardian {
    function f00000000_bvvvdlt() external view returns (address);

    function f00000001_grffjzz() external view returns (address);
}

contract Gate {
    bool public opened;

    function open(address guardian) external {
        uint256 codeSize;
        assembly {
            codeSize := extcodesize(guardian)
        }
        require(codeSize < 33, "bad code size");

        require(
            IGuardian(guardian).f00000000_bvvvdlt() == address(this),
            "invalid pass"
        );
        require(
            IGuardian(guardian).f00000001_grffjzz() == tx.origin,
            "invalid pass"
        );

        (bool success, ) = guardian.call(abi.encodeWithSignature("fail()"));
        require(!success);

        opened = true;
    }
}
```

We have to call `open()` with our contract address in order to set `opened` to `true`

Our contract's code size need to be less than 33 bytes, so we have to write our bytecode by hand

First, it should return `msg.sender` when `f00000000_bvvvdlt()` is called, it has a function selector of `0x00000000`

Second, it should return `tx.origin` when `f00000001_grffjzz()` is called, it has a function selector of `0x00000001`

Finally, it should revert if `fail()` is called

We can shorten the bytecode by substituting something like `PUSH1 0x00` with `RETURNDATASIZE` if there's no previous call, also by caching a copy of the function selector from calldata in the stack with `DUP1` instead of parsing it from the calldata again

### Bytecode (32 bytes)
```
RETURNDATASIZE
CALLDATALOAD
PUSH1 0x03
BYTE
DUP1
PUSH1 0x01
EQ
PUSH1 0x12
JUMPI
RETURNDATASIZE
EQ
PUSH1 0x19
JUMPI
REVERT
JUMPDEST
ORIGIN
RETURNDATASIZE
MSTORE
MSIZE
RETURNDATASIZE
RETURN
JUMPDEST
CALLER
RETURNDATASIZE
MSTORE
MSIZE
RETURNDATASIZE
RETURN

3d3560031a806001146012573d14601957fd5b323d52593df35b333d52593df3
```

We can test it on evm.codes playground :

https://www.evm.codes/playground?fork=arrowGlacier&unit=Wei&callData=0x00000001&codeType=Mnemonic&code='~CALLrLOADy03zBYTEzDUP1y01zq2vIz~q9vIzREVERTtORIGINz~wtCALLERz~w'ursz%5CnyzPUSH1%200xwMSTOREzMsuvzJUMPu~RETURNtvDESTzsSIZEzrDATAqEQy1%01qrstuvwyz~_

Then I will use this contract constructor to deploy the bytecode :
```solidity
// SPDX-License-Identifier: UNLICENSED

pragma solidity ^0.8.0;

contract GateSolveDeployer {
    constructor(bytes memory code) { assembly { return (add(code, 0x20), mload(code)) } }
}
```

## Proof of concept

### Hardhat test

```js
const { expect } = require("chai");
const { ethers } = require("hardhat");

describe('QuillCTF : Gate', () => {

  before(async () => {
    [owner, attacker] = await ethers.getSigners();
    contract = await ethers.getContractFactory('Gate', owner).then(f => f.deploy());
    await contract.deployed();
  });

  it('opened should be set to true', async () => {
    // deploy the attacker contract
    bytecode = '0x3d3560031a806001146012573d14601957fd5b323d52593df35b333d52593df3';
    attackerContract = await ethers
      .getContractFactory('GateSolveDeployer', attacker)
      .then(f => f.deploy(bytecode));
    await attackerContract.deployed();

    await contract.connect(attacker).open(attackerContract.address);
    expect(await contract.opened()).to.be.true;
  });
});
```

### Hardhat test output

```
# npx hardhat test test/gate.js 


  QuillCTF : Gate
true
    ✔ opened should be set to true (193ms)


  1 passing (2s)
```