## Immutable over constant
The use of constant `keccak` variables results in extra hashing whenever the variable is used, increasing gas costs relative to just storing the output hash. Changing to immutable will only perform hashing on contract deployment which will save gas.

Here are some of the instances entailed.

https://github.com/reserve-protocol/protocol/blob/9ee60f142f9f5c1fe8bc50eef915cf33124a534f/contracts/plugins/assets/aave/StaticATokenLM.sol#L44-L60

```solidity
    bytes public constant EIP712_REVISION = bytes("1");
    bytes32 internal constant EIP712_DOMAIN =
        keccak256(
            "EIP712Domain(string name,string version,uint256 chainId,address verifyingContract)"
        );
    bytes32 public constant PERMIT_TYPEHASH =
        keccak256(
            "Permit(address owner,address spender,uint256 value,uint256 nonce,uint256 deadline)"
        );
    bytes32 public constant METADEPOSIT_TYPEHASH =
        keccak256(
            "Deposit(address depositor,address recipient,uint256 value,uint16 referralCode,bool fromUnderlying,uint256 nonce,uint256 deadline)"
        );
    bytes32 public constant METAWITHDRAWAL_TYPEHASH =
        keccak256(
            "Withdraw(address owner,address recipient,uint256 staticAmount,uint256 dynamicAmount,bool toUnderlying,uint256 nonce,uint256 deadline)"
        );
```
## Unreachable code lines
The following else block in the function logic of `refresh()` is never reachable considering [`tryPrice()`](https://github.com/reserve-protocol/protocol/blob/9ee60f142f9f5c1fe8bc50eef915cf33124a534f/contracts/plugins/assets/Asset.sol#L64-L84) does not have (0, FIX_MAX) catered for. Any upriced data would have been sent to the catch block. Other than Asset.sol, similarly wasted logic is also exhibited in [`AppreciatingFiatCollateral.refresh`](https://github.com/reserve-protocol/protocol/blob/9ee60f142f9f5c1fe8bc50eef915cf33124a534f/contracts/plugins/assets/AppreciatingFiatCollateral.sol#L113-L116), [`FiatCollateral.refresh`](https://github.com/reserve-protocol/protocol/blob/9ee60f142f9f5c1fe8bc50eef915cf33124a534f/contracts/plugins/assets/FiatCollateral.sol#L136-L139), [`CTokenV3Collateral.refresh`](https://github.com/reserve-protocol/protocol/blob/9ee60f142f9f5c1fe8bc50eef915cf33124a534f/contracts/plugins/assets/compoundv3/CTokenV3Collateral.sol#L105-L110), [`CurveStableCollateral.refresh`](https://github.com/reserve-protocol/protocol/blob/9ee60f142f9f5c1fe8bc50eef915cf33124a534f/contracts/plugins/assets/curve/CurveStableCollateral.sol#L110-L115), [`StargatePoolFiatCollateral.refresh`](https://github.com/reserve-protocol/protocol/blob/9ee60f142f9f5c1fe8bc50eef915cf33124a534f/contracts/plugins/assets/stargate/StargatePoolFiatCollateral.sol#L87-L90).  

https://github.com/reserve-protocol/protocol/blob/9ee60f142f9f5c1fe8bc50eef915cf33124a534f/contracts/plugins/assets/Asset.sol#L86-L106

```solidity
    /// Should not revert
    /// Refresh saved prices
    function refresh() public virtual override {
        try this.tryPrice() returns (uint192 low, uint192 high, uint192) {
            // {UoA/tok}, {UoA/tok}
            // (0, 0) is a valid price; (0, FIX_MAX) is unpriced

            // Save prices if priced
            if (high < FIX_MAX) {
                savedLowPrice = low;
                savedHighPrice = high;
                lastSave = uint48(block.timestamp);
            } else {
                // must be unpriced
                assert(low == 0);
            }
        } catch (bytes memory errData) {
            // see: docs/solidity-style.md#Catching-Empty-Data
            if (errData.length == 0) revert(); // solhint-disable-line reason-string
        }
    }
```
## Redundant `return` in the `try` clause
This `try` clause has already returned `low` and `high` and is returning the same things again in its nested logic, which is inexpedient and unnecessary. Similar behavior is also found in [`Asset.price`](https://github.com/reserve-protocol/protocol/blob/9ee60f142f9f5c1fe8bc50eef915cf33124a534f/contracts/plugins/assets/Asset.sol#L112-L121). 

https://github.com/reserve-protocol/protocol/blob/9ee60f142f9f5c1fe8bc50eef915cf33124a534f/contracts/plugins/assets/RTokenAsset.sol#L86-L94

```diff
    function price() public view virtual returns (uint192, uint192) {
        try this.tryPrice() returns (uint192 low, uint192 high) {
-            return (low, high);
        } catch (bytes memory errData) {
            // see: docs/solidity-style.md#Catching-Empty-Data
            if (errData.length == 0) revert(); // solhint-disable-line reason-string
            return (0, FIX_MAX);
        }
    }
```
## Unneeded import in `RTokenAsset.sol`
`IRToken.sol` has already been imported by `IMain.sol`, making the former an unneeded import. 

https://github.com/reserve-protocol/protocol/blob/9ee60f142f9f5c1fe8bc50eef915cf33124a534f/contracts/plugins/assets/RTokenAsset.sol#L5-L6

```diff
 import "../../interfaces/IMain.sol";
- import "../../interfaces/IRToken.sol";
```
## Cached variables not efficiently used
In `StaticATokenLM._updateRewards`, the state variable `_lifetimeRewardsClaimed` is doubly used in the following code logic when `rewardsAccrued` could simply/equally be assigned `freshRewards.wadToRay()`. This extra gas incurring behaviour is also exhibited in [`StaticATokenLM._collectAndUpdateRewards`](https://github.com/reserve-protocol/protocol/blob/9ee60f142f9f5c1fe8bc50eef915cf33124a534f/contracts/plugins/assets/aave/StaticATokenLM.sol#L428-L434), and [`StaticATokenLM._getPendingRewards`](https://github.com/reserve-protocol/protocol/blob/9ee60f142f9f5c1fe8bc50eef915cf33124a534f/contracts/plugins/assets/aave/StaticATokenLM.sol#L581-L583).

https://github.com/reserve-protocol/protocol/blob/9ee60f142f9f5c1fe8bc50eef915cf33124a534f/contracts/plugins/assets/aave/StaticATokenLM.sol#L406-L408

```diff
            uint256 freshRewards = INCENTIVES_CONTROLLER.getRewardsBalance(assets, address(this));
            uint256 lifetimeRewards = _lifetimeRewardsClaimed.add(freshRewards);
-            uint256 rewardsAccrued = lifetimeRewards.sub(_lifetimeRewards).wadToRay();
+            uint256 rewardsAccrued = freshRewards.wadToRay();
```
## Identical for loop check
The if block in the constructor of PoolTokens.sol entails identical if block checks that may be moved outside the loop to save gas.

https://github.com/reserve-protocol/protocol/blob/9ee60f142f9f5c1fe8bc50eef915cf33124a534f/contracts/plugins/assets/curve/PoolTokens.sol#L123-L130

```diff
        IERC20Metadata[] memory tokens = new IERC20Metadata[](nTokens);

+        if (config.poolType == CurvePoolType.Plain) revert("invalid poolType");

        for (uint8 i = 0; i < nTokens; ++i) {
-            if (config.poolType == CurvePoolType.Plain) {
                tokens[i] = IERC20Metadata(curvePool.coins(i));
-            } else {
-                revert("invalid poolType");
-            }
        }
```
## Unneeded ternary logic, booleans, and second condition checks
In the constructor of PoolTokens.sol, the second condition of the following require statement mandates that at least one feed is associated with each token.

https://github.com/reserve-protocol/protocol/blob/9ee60f142f9f5c1fe8bc50eef915cf33124a534f/contracts/plugins/assets/curve/PoolTokens.sol#L106-L109

```solidity
        require(
            config.feeds.length == config.nTokens && minFeedsLength(config.feeds) > 0,
            "each token needs at least 1 price feed"
        );
```
Additionally, the following code lines signify that a minimum of 2 tokens will be associated with the Curve base pool. Otherwise, if `nTokens` is less than 2 or 1, assigning `token0` and/or `token1` will revert due to accessing out-of-bound array elements. 

https://github.com/reserve-protocol/protocol/blob/9ee60f142f9f5c1fe8bc50eef915cf33124a534f/contracts/plugins/assets/curve/PoolTokens.sol#L132-L135

```solidity
        token0 = tokens[0];
        token1 = tokens[1];
        token2 = (nTokens > 2) ? tokens[2] : IERC20Metadata(address(0));
        token3 = (nTokens > 3) ? tokens[3] : IERC20Metadata(address(0));
```
Under this context, the following ternary logic along with the use of boolean `more` is therefore deemed unnecessary.

https://github.com/reserve-protocol/protocol/blob/9ee60f142f9f5c1fe8bc50eef915cf33124a534f/contracts/plugins/assets/curve/PoolTokens.sol#L143-L154

```diff
        // token0
-        bool more = config.feeds[0].length > 0;
        // untestable:
        //     more will always be true based on previous feeds validations
-        _t0feed0 = more ? config.feeds[0][0] : AggregatorV3Interface(address(0));
+        _t0feed0 = config.feeds[0][0];
        _t0timeout0 = more && config.oracleTimeouts[0].length > 0 ? config.oracleTimeouts[0][0] : 0;
        _t0error0 = more && config.oracleErrors[0].length > 0 ? config.oracleErrors[0][0] : 0;
-        if (more) {
            require(address(_t0feed0) != address(0), "t0feed0 empty");
            require(_t0timeout0 > 0, "t0timeout0 zero");
            require(_t0error0 < FIX_ONE, "t0error0 too large");
-        }
```
https://github.com/reserve-protocol/protocol/blob/9ee60f142f9f5c1fe8bc50eef915cf33124a534f/contracts/plugins/assets/curve/PoolTokens.sol#L166-L177

```diff
        // token1
        // untestable:
        //     more will always be true based on previous feeds validations
-        more = config.feeds[1].length > 0;
-        _t1feed0 = more ? config.feeds[1][0] : AggregatorV3Interface(address(0));
+        _t1feed0 = config.feeds[1][0];
        _t1timeout0 = more && config.oracleTimeouts[1].length > 0 ? config.oracleTimeouts[1][0] : 0;
        _t1error0 = more && config.oracleErrors[1].length > 0 ? config.oracleErrors[1][0] : 0;
-        if (more) {
            require(address(_t1feed0) != address(0), "t1feed0 empty");
            require(_t1timeout0 > 0, "t1timeout0 zero");
            require(_t1error0 < FIX_ONE, "t1error0 too large");
-        }
```
Similarly, the following second conditional checks are also not needed.

https://github.com/reserve-protocol/protocol/blob/9ee60f142f9f5c1fe8bc50eef915cf33124a534f/contracts/plugins/assets/curve/PoolTokens.sol#L189-L190

```diff
        // token2
-        more = config.feeds.length > 2 && config.feeds[2].length > 0;
+        more = config.feeds.length > 2;
```
https://github.com/reserve-protocol/protocol/blob/9ee60f142f9f5c1fe8bc50eef915cf33124a534f/contracts/plugins/assets/curve/PoolTokens.sol#L210-L211

```diff
        // token3
-        more = config.feeds.length > 3 && config.feeds[3].length > 0;
+        more = config.feeds.length > 3;
```
## Use of named returns for local variables saves gas
You can have further advantages in terms of gas cost by simply using named return values as temporary local variables.

For instance, the code block below may be refactored as follows:

https://github.com/reserve-protocol/protocol/blob/9ee60f142f9f5c1fe8bc50eef915cf33124a534f/contracts/plugins/assets/compoundv3/CusdcV3Wrapper.sol#L225

```diff
-    function underlyingBalanceOf(address account) public view returns (uint256) {
+    function underlyingBalanceOf(address account) public view returns (uint256 _balance) {
        uint256 balance = balanceOf(account);
        if (balance == 0) {
            return 0;
        }
-        return convertStaticToDynamic(safe104(balance));
+        _balance = convertStaticToDynamic(safe104(balance));
    }
```
## += and -= cost more gas
`+=` and `-=` generally cost 22 more gas than writing out the assigned equation explicitly. The amount of gas wasted can be quite sizable when repeatedly operated in a loop.

For instance, the `+=` instance below may be refactored as follows:

https://github.com/reserve-protocol/protocol/blob/9ee60f142f9f5c1fe8bc50eef915cf33124a534f/contracts/plugins/assets/compoundv3/CusdcV3Wrapper.sol#L307

```diff
-            baseSupplyIndex_ += safe64(mulFactor(baseSupplyIndex_, supplyRate * timeDelta));
+            baseSupplyIndex_ = baseSupplyIndex_ + safe64(mulFactor(baseSupplyIndex_, supplyRate * timeDelta));
```
## Function order affects gas consumption
The order of function will also have an impact on gas consumption. Because in smart contracts, there is a difference in the order of the functions. Each position will have an extra 22 gas. The order is dependent on method ID. So, if you rename the frequently accessed function to more early method ID, you can save gas cost. Please visit the following site for further information:

https://medium.com/joyso/solidity-how-does-function-name-affect-gas-consumption-in-smart-contract-47d270d8ac92

## Activate the optimizer
Before deploying your contract, activate the optimizer when compiling using “solc --optimize --bin sourceFile.sol”. By default, the optimizer will optimize the contract assuming it is called 200 times across its lifetime. If you want the initial contract deployment to be cheaper and the later function executions to be more expensive, set it to “ --optimize-runs=1”. Conversely, if you expect many transactions and do not care for higher deployment cost and output size, set “--optimize-runs” to a high number.

```
module.exports = {
solidity: {
version: "0.8.19",
settings: {
  optimizer: {
    enabled: true,
    runs: 1000,
  },
},
},
};
```
Please visit the following site for further information:

https://docs.soliditylang.org/en/v0.5.4/using-the-compiler.html#using-the-commandline-compiler

Here's one example of instance on opcode comparison that delineates the gas saving mechanism:

```
for !=0 before optimization
PUSH1 0x00
DUP2
EQ
ISZERO
PUSH1 [cont offset]
JUMPI

after optimization
DUP1
PUSH1 [revert offset]
JUMPI
```
Disclaimer: There have been several bugs with security implications related to optimizations. For this reason, Solidity compiler optimizations are disabled by default, and it is unclear how many contracts in the wild actually use them. Therefore, it is unclear how well they are being tested and exercised. High-severity security issues due to optimization bugs have occurred in the past. A high-severity bug in the emscripten -generated solc-js compiler used by Truffle and Remix persisted until late 2018. The fix for this bug was not reported in the Solidity CHANGELOG. Another high-severity optimization bug resulting in incorrect bit shift results was patched in Solidity 0.5.6. Please measure the gas savings from optimizations, and carefully weigh them against the possibility of an optimization-related bug. Also, monitor the development and adoption of Solidity compiler optimizations to assess their maturity.

## Constructors can be marked payable
Payable functions cost less gas to execute, since the compiler does not have to add extra checks to ensure that a payment wasn't provided. A constructor can safely be marked as payable, since only the deployer would be able to pass funds, and the project itself would not pass any funds.

Here are some of the instances entailed:

https://github.com/reserve-protocol/protocol/blob/9ee60f142f9f5c1fe8bc50eef915cf33124a534f/contracts/plugins/assets/aave/ATokenFiatCollateral.sol#L42

```solidity
    constructor(CollateralConfig memory config, uint192 revenueHiding)
```
https://github.com/reserve-protocol/protocol/blob/9ee60f142f9f5c1fe8bc50eef915cf33124a534f/contracts/plugins/assets/EURFiatCollateral.sol#L23-L27

```solidity
    constructor(
        CollateralConfig memory config,
        AggregatorV3Interface targetUnitChainlinkFeed_,
        uint48 targetUnitOracleTimeout_
    ) FiatCollateral(config) {
```
https://github.com/reserve-protocol/protocol/blob/9ee60f142f9f5c1fe8bc50eef915cf33124a534f/contracts/plugins/assets/FiatCollateral.sol#L62-L70

```solidity
    constructor(CollateralConfig memory config)
        Asset(
            config.priceTimeout,
            config.chainlinkFeed,
            config.oracleError,
            config.erc20,
            config.maxTradeVolume,
            config.oracleTimeout
        )
```
## Use assembly for small keccak256 hashes, in order to save gas
If the arguments to the encode call can fit into the scratch space (two words or fewer), then it's more efficient to use assembly to generate the hash (80 gas): keccak256(abi.encodePacked(x, y)) -> assembly {mstore(0x00, a); mstore(0x20, b); let hash := keccak256(0x00, 0x40); }

Here are some of the instances entailed:

https://github.com/reserve-protocol/protocol/blob/9ee60f142f9f5c1fe8bc50eef915cf33124a534f/contracts/plugins/assets/aave/StaticATokenLM.sol#L285-L286

```solidity
                    keccak256(bytes(name())),
                    keccak256(EIP712_REVISION),
```
## Reduce gas usage by moving to Solidity 0.8.19 or later
Please visit the following link for substantiated details:

https://soliditylang.org/blog/2023/02/22/solidity-0.8.19-release-announcement/#preventing-dead-code-in-runtime-bytecode

And, here are some of the instances entailed:

https://github.com/reserve-protocol/protocol/blob/9ee60f142f9f5c1fe8bc50eef915cf33124a534f/contracts/plugins/assets/aave/StaticATokenLM.sol#L2

```solidity
pragma solidity 0.6.12;
```
https://github.com/reserve-protocol/protocol/blob/9ee60f142f9f5c1fe8bc50eef915cf33124a534f/contracts/plugins/assets/aave/StaticATokenErrors.sol

```solidity
pragma solidity 0.6.12;
```
## Using this to access functions results in an external call, wasting gas
External calls have an overhead of 100 gas, which can be avoided by not referencing the function using this. Contracts are [allowed](https://docs.soliditylang.org/en/latest/contracts.html#function-overriding) to override their parents' functions and change the visibility from external to public, so make this change if it's required in order to call the function internally.

Here are some of the instances entailed:

https://github.com/reserve-protocol/protocol/blob/9ee60f142f9f5c1fe8bc50eef915cf33124a534f/contracts/plugins/assets/AppreciatingFiatCollateral.sol#L104

```solidity
        try this.tryPrice() returns (uint192 low, uint192 high, uint192 pegPrice) {
```
https://github.com/reserve-protocol/protocol/blob/9ee60f142f9f5c1fe8bc50eef915cf33124a534f/contracts/plugins/assets/Asset.sol

```solidity
89:        try this.tryPrice() returns (uint192 low, uint192 high, uint192) {

113:        try this.tryPrice() returns (uint192 low, uint192 high, uint192) {

129:        try this.tryPrice() returns (uint192 low, uint192 high, uint192) {
```