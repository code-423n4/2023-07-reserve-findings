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