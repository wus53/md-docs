# Uniswap V2 Project Review 🦄

## Introduction

Uniswap V2 is the second version of the popular decentralized exchange (DEX) platform Uniswap. It was introduced as an upgrade to the original version, with improvements such as increased liquidity, lower trading fees, and the ability to trade new types of assets. Uniswap V2 uses an automated market maker (AMM) algorithm to facilitate trading, providing a more efficient and accessible way for users to exchange their digital assets. Additionally, it features token-pair reserves, which allow for more stable pricing and reduced slippage compared to Uniswap V1.

## Smart Contract Code Analysis

Uniswap V2 is divided into two components: **Core** and **Periphery**. The core contracts, which store the assets (ether and tokens), therefore, must be secure, simple, and easy to audit. Periphery contracts can then offer all the additional functionality required by traders. Due to the length constraints, I will only walk through core contracts code in detail.

### UML of the project
- Uniswap V2 Core UML:

![uml-core](https://user-images.githubusercontent.com/118578313/217465255-b84e7724-c1b5-4052-8a4b-7789deb4c4ce.svg)

- Uniswap V2 Core Inheritance UML:
  
![inheritance-core](https://user-images.githubusercontent.com/118578313/217465321-a6cba7ce-d760-4983-a2cd-c656dc2570c8.svg)

- Uniswap V2 Periphery UML:

![uml-periphery](https://user-images.githubusercontent.com/118578313/217465441-db8b54e8-0f87-4139-b501-6956e0b007b5.svg)

- Uniswap V2 Periphery Inheritance UML:

![inheritance-periphery](https://user-images.githubusercontent.com/118578313/217465489-ddfe685b-0f14-4406-81d6-5e34b1ac6f94.svg)

### Use Cases

There are two main use cases for Uniswap V2: 
  
![](https://miro.medium.com/v2/resize:fit:1400/format:webp/1*QC-dT9XnjUp5d6k45pND8g.jpeg)

Image Source: https://medium.com/@chiqing/uniswap-v2-explained-beginner-friendly-b5d2cb64fe0f


**1. Adding & Removing Liquidity**
  - The control flow of [removing liquidity](https://github.com/Uniswap/v2-periphery/blob/master/contracts/UniswapV2Router02.sol#L103) is basically the same as adding liquidity, so I will only walk through ***Adding Liquidity*** here.

    > Note: According to the [Ethereum website](https://ethereum.org/en/developers/tutorials/uniswap-v2-annotated-code/#UniswapV2Router01): UniswapV2Router01 has problems and should no longer be used.
   
    1. Caller first needs to provide the periphery contract with an `allowance` in the amounts to be added to the liquidity pool. Then, caller calls [*addLiquidity*](https://github.com/Uniswap/v2-periphery/blob/master/contracts/UniswapV2Router02.sol#L61) function in the periphery contract (`UniswapV2Router02.sol`).
    2. In `UniswapV2Router02.sol` (periphery) contract:
        ```solidity
        function _addLiquidity(
            address tokenA,
            address tokenB,
            uint amountADesired,
            uint amountBDesired,
            uint amountAMin,
            uint amountBMin
        ) internal virtual returns (uint amountA, uint amountB) {
            // create the pair if it doesn't exist yet
            if (IUniswapV2Factory(factory).getPair(tokenA, tokenB) == address(0)) {
                IUniswapV2Factory(factory).createPair(tokenA, tokenB);
            }
            (uint reserveA, uint reserveB) = UniswapV2Library.getReserves(factory, tokenA, tokenB);
            if (reserveA == 0 && reserveB == 0) {
                (amountA, amountB) = (amountADesired, amountBDesired);
            } else {
                uint amountBOptimal = UniswapV2Library.quote(amountADesired, reserveA, reserveB);
                if (amountBOptimal <= amountBDesired) {
                    require(amountBOptimal >= amountBMin, 'UniswapV2Router: INSUFFICIENT_B_AMOUNT');
                    (amountA, amountB) = (amountADesired, amountBOptimal);
                } else {
                    uint amountAOptimal = UniswapV2Library.quote(amountBDesired, reserveB, reserveA);
                    assert(amountAOptimal <= amountADesired);
                    require(amountAOptimal >= amountAMin, 'UniswapV2Router: INSUFFICIENT_A_AMOUNT');
                    (amountA, amountB) = (amountAOptimal, amountBDesired);
                }
            }
        }

        function addLiquidity(
            address tokenA,
            address tokenB,
            uint amountADesired,
            uint amountBDesired,
            uint amountAMin,
            uint amountBMin,
            address to,
            uint deadline
        ) external virtual override ensure(deadline) returns (uint amountA, uint amountB, uint liquidity) {
            (amountA, amountB) = _addLiquidity(tokenA, tokenB, amountADesired, amountBDesired, amountAMin, amountBMin);
            address pair = UniswapV2Library.pairFor(factory, tokenA, tokenB);
            TransferHelper.safeTransferFrom(tokenA, msg.sender, pair, amountA);
            TransferHelper.safeTransferFrom(tokenB, msg.sender, pair, amountB);
            liquidity = IUniswapV2Pair(pair).mint(to);
        }
        ```
        - Create a new token pair if needed (`createPair` function in `v2-core/contracts/UniswapV2Factory.sol`)
        - If there already exists such a pair, calculate the amount of tokens to add in `_addLiquidity` function to ensure the ratio of new tokens is identical to existing tokens.
        - Check if the amounts are sufficient. Callers can specify a minimum amount `amountAMin` or `amountBMin`, below which they would prefer not to add liquidity.
        - Call functions in the core contract.
  
    3. In `UniswapV2Pair.sol` (core) contract:
        ```solidity
        // this low-level function should be called from a contract which performs important safety checks
        function mint(address to) external lock returns (uint liquidity) {
            (uint112 _reserve0, uint112 _reserve1,) = getReserves(); // gas savings
            uint balance0 = IERC20(token0).balanceOf(address(this));
            uint balance1 = IERC20(token1).balanceOf(address(this));
            uint amount0 = balance0.sub(_reserve0);
            uint amount1 = balance1.sub(_reserve1);

            bool feeOn = _mintFee(_reserve0, _reserve1);
            uint _totalSupply = totalSupply; // gas savings, must be defined here since totalSupply can update in _mintFee
            if (_totalSupply == 0) {
                liquidity = Math.sqrt(amount0.mul(amount1)).sub(MINIMUM_LIQUIDITY);
              _mint(address(0), MINIMUM_LIQUIDITY); // permanently lock the first MINIMUM_LIQUIDITY tokens
            } else {
                liquidity = Math.min(amount0.mul(_totalSupply) / _reserve0, amount1.mul(_totalSupply) / _reserve1);
            }
            require(liquidity > 0, 'UniswapV2: INSUFFICIENT_LIQUIDITY_MINTED');
            _mint(to, liquidity);

            _update(balance0, balance1, _reserve0, _reserve1);
            if (feeOn) kLast = uint(reserve0).mul(reserve1); // reserve0 and reserve1 are up-to-date
            emit Mint(msg.sender, amount0, amount1);
        }
        ```
        - [`mint` function](https://github.com/Uniswap/v2-core/blob/master/contracts/UniswapV2Pair.sol#L110) mints liquidity tokens (UNI) & send them to the caller.
        ```solidity
        // update reserves and, on the first call per block, price accumulators
        function _update(uint balance0, uint balance1, uint112 _reserve0, uint112 _reserve1) private {
            require(balance0 <= uint112(-1) && balance1 <= uint112(-1), 'UniswapV2: OVERFLOW');
            uint32 blockTimestamp = uint32(block.timestamp % 2**32);
            uint32 timeElapsed = blockTimestamp - blockTimestampLast; // overflow is desired
            if (timeElapsed > 0 && _reserve0 != 0 && _reserve1 != 0) {
                // * never overflows, and + overflow is desired
                price0CumulativeLast += uint(UQ112x112.encode(_reserve1).uqdiv(_reserve0)) * timeElapsed;
                price1CumulativeLast += uint(UQ112x112.encode(_reserve0).uqdiv(_reserve1)) * timeElapsed;
            }
            reserve0 = uint112(balance0);
            reserve1 = uint112(balance1);
            blockTimestampLast = blockTimestamp;
            emit Sync(reserve0, reserve1);
        }
        ```
        - Calling [`_update` function](https://github.com/Uniswap/v2-core/blob/master/contracts/UniswapV2Pair.sol#L73) to update the reserve amounts

**2. Swapping**

   1. Caller first needs to provide the periphery contract with an `allowance` in the amounts to be swapped. Then, caller calls one of the swap functions in the [*UniswapV2Router02.sol*](https://github.com/Uniswap/v2-periphery/blob/master/contracts/UniswapV2Router02.sol) contract. Each `swap` function accepts a `path`, which is an array of exchanges to go through.
   2. In `UniswapV2Router02.sol` (periphery) contract, let's take `swapExactTokensForTokens` function as an example:
      ```solidity
      function swapExactTokensForTokens(
          uint amountIn,
          uint amountOutMin,
          address[] calldata path,
          address to,
          uint deadline
      ) external virtual override ensure(deadline) returns (uint[] memory amounts) {
          amounts = UniswapV2Library.getAmountsOut(factory, amountIn, path);
          require(amounts[amounts.length - 1] >= amountOutMin, 'UniswapV2Router: INSUFFICIENT_OUTPUT_AMOUNT');
          TransferHelper.safeTransferFrom(
              path[0], msg.sender, UniswapV2Library.pairFor(factory, path[0], path[1]), amounts[0]
          );
          _swap(amounts, path, to);
      }
      ```
      - It first calculates the amounts that need to be traded (either `In` or `Out` in other functions) on each exchange along the path by calling functions in `UniswapV2Library` contract.

      <br/>
  
      ```solidity
      // requires the initial amount to have already been sent to the first pair
      function _swap(uint[] memory amounts, address[] memory path, address _to) internal virtual {
          for (uint i; i < path.length - 1; i++) {
              (address input, address output) = (path[i], path[i + 1]);
              (address token0,) = UniswapV2Library.sortTokens(input, output);
              uint amountOut = amounts[i + 1];
              (uint amount0Out, uint amount1Out) = input == token0 ? (uint(0), amountOut) : (amountOut, uint(0));
              address to = i < path.length - 2 ? UniswapV2Library.pairFor(factory, output, path[i + 2]) : _to;
              IUniswapV2Pair(UniswapV2Library.pairFor(factory, input, output)).swap(
                  amount0Out, amount1Out, to, new bytes(0)
              );
          }
      }
      ```
      - Function `_swap` will then iterate through the `path`. For every pair exchange along the `path`, it sends the input token and then calls the `swap` function of that pair exchange. The final destination token address is the one specified by the trader.
  3. In `UniswapV2Pair.sol` (core) contract:
      ```solidity
      // this low-level function should be called from a contract which performs important safety checks
      function swap(uint amount0Out, uint amount1Out, address to, bytes calldata data) external lock {
          require(amount0Out > 0 || amount1Out > 0, 'UniswapV2: INSUFFICIENT_OUTPUT_AMOUNT');
          (uint112 _reserve0, uint112 _reserve1,) = getReserves(); // gas savings
          require(amount0Out < _reserve0 && amount1Out < _reserve1, 'UniswapV2: INSUFFICIENT_LIQUIDITY');

          uint balance0;
          uint balance1;
          { // scope for _token{0,1}, avoids stack too deep errors
          address _token0 = token0;
          address _token1 = token1;
          require(to != _token0 && to != _token1, 'UniswapV2: INVALID_TO');
          if (amount0Out > 0) _safeTransfer(_token0, to, amount0Out); // optimistically transfer tokens
          if (amount1Out > 0) _safeTransfer(_token1, to, amount1Out); // optimistically transfer tokens
          if (data.length > 0) IUniswapV2Callee(to).uniswapV2Call(msg.sender, amount0Out, amount1Out, data);
          balance0 = IERC20(_token0).balanceOf(address(this));
          balance1 = IERC20(_token1).balanceOf(address(this));
          }
          uint amount0In = balance0 > _reserve0 - amount0Out ? balance0 - (_reserve0 - amount0Out) : 0;
          uint amount1In = balance1 > _reserve1 - amount1Out ? balance1 - (_reserve1 - amount1Out) : 0;
          require(amount0In > 0 || amount1In > 0, 'UniswapV2: INSUFFICIENT_INPUT_AMOUNT');
          { // scope for reserve{0,1}Adjusted, avoids stack too deep errors
          uint balance0Adjusted = balance0.mul(1000).sub(amount0In.mul(3));
          uint balance1Adjusted = balance1.mul(1000).sub(amount1In.mul(3));
          require(balance0Adjusted.mul(balance1Adjusted) >= uint(_reserve0).mul(_reserve1).mul(1000**2), 'UniswapV2: K');
          }

          _update(balance0, balance1, _reserve0, _reserve1);
          emit Swap(msg.sender, amount0In, amount1In, amount0Out, amount1Out, to);
      }
      ```
     - Ensure that the core contract is called from a secure contract, the `Out` output amount of tokens is valid, and the liquidity is sufficient for completing this swap.
     - Transfer the output tokens to the destination (i.e., next pair exchange or the final destination specified by the trader).
     - Calculate `In` input amount of tokens by getting the difference between current token balances and known token reserves.
     ```solidity
      // update reserves and, on the first call per block, price accumulators
      function _update(uint balance0, uint balance1, uint112 _reserve0, uint112 _reserve1) private {
          require(balance0 <= uint112(-1) && balance1 <= uint112(-1), 'UniswapV2: OVERFLOW');
          uint32 blockTimestamp = uint32(block.timestamp % 2**32);
          uint32 timeElapsed = blockTimestamp - blockTimestampLast; // overflow is desired
          if (timeElapsed > 0 && _reserve0 != 0 && _reserve1 != 0) {
              // * never overflows, and + overflow is desired
              price0CumulativeLast += uint(UQ112x112.encode(_reserve1).uqdiv(_reserve0)) * timeElapsed;
              price1CumulativeLast += uint(UQ112x112.encode(_reserve0).uqdiv(_reserve1)) * timeElapsed;
          }
          reserve0 = uint112(balance0);
          reserve1 = uint112(balance1);
          blockTimestampLast = blockTimestamp;
          emit Sync(reserve0, reserve1);
      }
      ```
     - Calling [`_update` function](https://github.com/Uniswap/v2-core/blob/master/contracts/UniswapV2Pair.sol#L73) to update the reserve amounts

  4. (Potential) In `UniswapV2Router02.sol` (periphery) contract:
     - The contract might need to do some necessary wrap-up (e.g., in some swap functions that involving ETH, like [swapTokensForExactETH](https://github.com/Uniswap/v2-periphery/blob/master/contracts/UniswapV2Router02.sol#L267) WETH tokens will be burned and ETH will be sent to the trader).

## Tokenomics

- Uniswap released its native token, UNI, on September 16th, 2020. The token was created as a way to reward users for their contributions to the Uniswap ecosystem and to provide a governance mechanism for the platform. The initial distribution of UNI was done through airdrops and liquidity provision rewards, which encouraged users to participate in the Uniswap platform and provide liquidity to its pools. UNI tokens have since become a popular asset in the decentralized finance (DeFi) space, with its value reflecting the growth and adoption of the Uniswap platform.
- The [current price](https://coinmarketcap.com/currencies/uniswap/) of the UNI token is 6.77 USD (as the time of writing 11a.m. Feb. 7)
- Here is the Genesis UNI token allocation plan:

  ![](https://uniswap.org/images/posts/uni/Genesis.png)
- There are three main ways to earn the Uniswap UNI tokens rather than just buying/swapping them on exchanges (either decentralized or centralized):
  - **Liquidity provision**: One of the easiest ways to get UNI is by providing liquidity to Uniswap pools. By adding funds to the platform, users are eligible for UNI rewards, which are distributed periodically to liquidity providers.
  - **Airdrops**: Uniswap has conducted several airdrops in the past, where UNI tokens are distributed to users for free. Keep an eye out for announcements from Uniswap or the wider DeFi community to learn about upcoming airdrops.
  - **Note**: for the above two ways, according to Uniswap's [UNI token blog](https://uniswap.org/blog/uni), back when UNI token was launched, 15% of UNI `150,000,000 UNI` can immediately be claimed by **historical liquidity providers, users, and SOCKS redeemers/holders** based on a snapshot ending September 1, 2020, at 12:00 am UTC.
  - **Participating in yield farming**: UNI can also be earned through yield farming, where users lend or stake their tokens to earn rewards. Check the available yield farming opportunities on platforms like Yearn.finance and Aave.
- Read more about the UNI tokenomics [here](https://uniswap.org/blog/uni).

## Disclaimer
The information provided in this document is provided solely for educational purposes and does not constitute any advice, including but not limited to, investment advice, trading advice or financial advice.

## Credits

- [UNISWAP-V2 CONTRACT WALK-THROUGH](https://ethereum.org/en/developers/tutorials/uniswap-v2-annotated-code/)
- [Solidity Visual Developer VS Code Extension](https://marketplace.visualstudio.com/items?itemName=tintinweb.solidity-visual-auditor)
- [Uniswap V2 Core](https://github.com/Uniswap/v2-core)
- [Uniswap V2 Periphery](https://github.com/Uniswap/v2-periphery)
- [Martin's Study Notes](https://docs.page/ymart1n/study-notes/UniSwap_AMM_Study)
