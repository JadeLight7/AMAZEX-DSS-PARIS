## Operation Rescue POSI Token!
看到题目就激动了，这就是我在前几天做的那个题，太典了，又是把钱转到了另一条链上的相同地址。唯一不同的是这个题用的create2部署，钱转到了A链的B地址，我们自己的钱包在B链B地址，只需要根据部署B链B地址的钱包salt和byte重新在A链B地址部署就行了，因为部署者都是工厂合约，所以不用管deployer。

这个题现在的地址和昨年的不一样，我不知道是故意为之还是怎么样，现在题目的地址怎么爆破都不成功，以前的地址可以成功，额日期额题设透露的员工生日11月的11就是对应的salt，本质都是一样的

###  解题
```solidity
// SPDX-License-Identifier: UNLICENSED
pragma solidity ^0.8.0;

import "forge-std/Test.sol";
import "forge-std/console.sol";

import {VaultFactory} from "../src/4_RescuePosi/myVaultFactory.sol";
import {VaultWalletTemplate} from "../src/4_RescuePosi/myVaultWalletTemplate.sol";
import {PosiCoin} from "../src/4_RescuePosi/PosiCoin.sol";

/*////////////////////////////////////////////////////////////
//          DEFINE ANY NECESSARY CONTRACTS HERE             //
//    If you need a contract for your hack, define it below //
////////////////////////////////////////////////////////////*/
contract help{
    address public unclaimedAddress = 0x70E194050d9c9c949b3061CC7cF89dF9c6782b7F;
    function findCorrectSalt(bytes memory bytecode, address deployer) public view returns (uint256) {
        for (uint256 salt = 0; salt < type(uint256).max; salt++) {
            bytes32 bytecodeHash = keccak256(bytecode);
            bytes32 hash = keccak256(abi.encodePacked(
                bytes1(0xff),   // Fixed value required by CREATE2
                deployer,       // The deployer's address (contract address in this context)
                salt,           // The salt value for creating the address
                bytecodeHash    // Keccak256 hash of the bytecode
            ));

            address computedAddress = address(uint160(uint256(hash)));  // Convert hash to address

            if (computedAddress == unclaimedAddress) {
                return salt;  // Return the salt if the address matches
            }
        }

        revert("No matching salt found.");
    }
}



/*////////////////////////////////////////////////////////////
//                     TEST CONTRACT                        //
////////////////////////////////////////////////////////////*/
contract Challenge4Test is Test {
    VaultFactory public FACTORY;
    PosiCoin public POSI;
    address public unclaimedAddress = 0x70E194050d9c9c949b3061CC7cF89dF9c6782b7F;
    address public whitehat = makeAddr("whitehat");
    address public devs = makeAddr("devs");

    function setUp() public {
        vm.label(unclaimedAddress, "Unclaimed Address");

        // Instantiate the Factory
        FACTORY = new VaultFactory();

        // Instantiate the POSICoin
        POSI = new PosiCoin();

        // OOPS transferred to the wrong address!
        POSI.transfer(unclaimedAddress, 1000 ether);
    }

    function testWhitehatRescue() public {
        vm.deal(whitehat, 10 ether);
        vm.startPrank(whitehat, whitehat);
        /*////////////////////////////////////////////////////
        //               Add your hack below!               //
        //                                                  //
        // terminal command to run the specific test:       //
        // forge test --match-contract Challenge4Test -vvvv //
        ////////////////////////////////////////////////////*/

        help helpcontract = new help();
        
        //利用FACTORY的deploy函数在unclaimedAddress地址部署VaultWalletTemplate合约
        bytes memory creationCode = type(VaultWalletTemplate).creationCode;
        uint256 salt = helpcontract.findCorrectSalt(creationCode,address(FACTORY));
        address addr = address(0);

        addr = FACTORY.deploy(creationCode,salt);
            
        
        require(addr == unclaimedAddress, "Failed to find correct salt");

        
        address payable vault = payable(addr);
        assertEq(vault, unclaimedAddress, "Failed to deploy at the target address");

        VaultWalletTemplate wallet = VaultWalletTemplate(vault);
        wallet.initialize(whitehat);

        wallet.withdrawERC20(address(POSI), 1000 ether, devs);

        vm.stopPrank();

        assertEq(POSI.balanceOf(devs), 1000 ether, "devs' POSI balance should be 1000 POSI");
    }
}

```
