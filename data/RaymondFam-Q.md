## Inexpedient require statement in `StaticATokenLM._withdraw`
The following require statement might need some refactoring to better accomplish its intended purpose. First off, it does not capture whether or not both `staticAmount` and `dynamicAmount` are zeroes at the same time.

https://github.com/reserve-protocol/protocol/blob/9ee60f142f9f5c1fe8bc50eef915cf33124a534f/contracts/plugins/assets/aave/StaticATokenLM.sol#L332-L335

```
        require(
            staticAmount == 0 || dynamicAmount == 0,
            StaticATokenErrors.ONLY_ONE_AMOUNT_FORMAT_ALLOWED
        );
```
Secondly, `staticAmount` and `dynamicAmount` have separately been hardcoded to `0` in the following two calling functions.

https://github.com/reserve-protocol/protocol/blob/9ee60f142f9f5c1fe8bc50eef915cf33124a534f/contracts/plugins/assets/aave/StaticATokenLM.sol#L128

```solidity
        return _withdraw(msg.sender, recipient, amount, 0, toUnderlying);
```
https://github.com/reserve-protocol/protocol/blob/9ee60f142f9f5c1fe8bc50eef915cf33124a534f/contracts/plugins/assets/aave/StaticATokenLM.sol#L139

```solidity
        return _withdraw(msg.sender, recipient, 0, amount, toUnderlying);
```
Here's a suggested fix:

```solidity
require(
    (staticAmount == 0 && dynamicAmount != 0) || (dynamicAmount == 0 && staticAmount != 0),
    StaticATokenErrors.ONLY_ONE_AMOUNT_FORMAT_ALLOWED
);
```
## Residual rewards irretrievable
The plugin rewards are claimable via the following methods:
- `accRewardsPerToken` multiplied by the pro-rate static share as adopted in [StaticATokenLM](https://github.com/reserve-protocol/protocol/blob/9ee60f142f9f5c1fe8bc50eef915cf33124a534f/contracts/plugins/assets/aave/StaticATokenLM.sol#L590)
- the delta of `accrued - claimed` as adopted in [CusdcV3Wrapper.sol](https://github.com/reserve-protocol/protocol/blob/9ee60f142f9f5c1fe8bc50eef915cf33124a534f/contracts/plugins/assets/compoundv3/CusdcV3Wrapper.sol#L202)
- the delta of `balanceAfterClaimingRewards - _previousBalance` featured by [RewardableERC20.sol](https://github.com/reserve-protocol/protocol/blob/9ee60f142f9f5c1fe8bc50eef915cf33124a534f/contracts/plugins/assets/erc20/RewardableERC20.sol#L84) and inherited by other plugins like [MorphoTokenisedDeposit.sol](https://github.com/reserve-protocol/protocol/blob/9ee60f142f9f5c1fe8bc50eef915cf33124a534f/contracts/plugins/assets/morpho-aave/MorphoTokenisedDeposit.sol#L17)

While no loss of funds is involved here, all residual balances due to accidentally sent in, failure of transfers, etc will be forever stuck in the contract.

## Typo mistakes
https://github.com/reserve-protocol/protocol/blob/9ee60f142f9f5c1fe8bc50eef915cf33124a534f/contracts/plugins/assets/FiatCollateral.sol#L22
```diff
-    uint48 delayUntilDefault; // {s} The number of seconds an oracle can mulfunction
+    uint48 delayUntilDefault; // {s} The number of seconds an oracle can malfunction
```
https://github.com/reserve-protocol/protocol/blob/9ee60f142f9f5c1fe8bc50eef915cf33124a534f/contracts/plugins/assets/FiatCollateral.sol#L47
```diff
-    //                In this case, the asset may recover, reachiving _whenDefault == NEVER.
+    //                In this case, the asset may recover, reachieving _whenDefault == NEVER.
```
https://github.com/reserve-protocol/protocol/blob/9ee60f142f9f5c1fe8bc50eef915cf33124a534f/contracts/plugins/assets/aave/StaticATokenLM.sol#L368

```diff
-     * @notice Updates rewards for senders and receiver in a transfer (not updating rewards for address(0))
+     * @notice Updates rewards for senders and receivers in a transfer (not updating rewards for address(0))
```
https://github.com/reserve-protocol/protocol/blob/9ee60f142f9f5c1fe8bc50eef915cf33124a534f/contracts/plugins/assets/aave/StaticATokenLM.sol#L553

```diff
-     * @notice Compute the pending in RAY (rounded down). Pending is the amount to add (not yet unclaimed) rewards in RAY (rounded down).
+     * @notice Compute the pending in RAY (rounded down). Pending is the amount to add (not yet claimed) rewards in RAY (rounded down).
```
https://github.com/reserve-protocol/protocol/blob/9ee60f142f9f5c1fe8bc50eef915cf33124a534f/contracts/plugins/assets/compoundv3/CusdcV3Wrapper.sol#L73

```diff
-    /// Takes `amount` fo cUSDCv3 from `src` and deposits to `dst` account in the wrapper.
+    /// Takes `amount` of cUSDCv3 from `src` and deposits to `dst` account in the wrapper.
```
https://github.com/reserve-protocol/protocol/blob/9ee60f142f9f5c1fe8bc50eef915cf33124a534f/contracts/plugins/assets/curve/PoolTokens.sol#L117

```diff
-        // an immutable array so we do not have store the token feeds in the blockchain. This is
+        // an immutable array so we do not have to store the token feeds in the blockchain. This is
```
https://github.com/reserve-protocol/protocol/blob/9ee60f142f9f5c1fe8bc50eef915cf33124a534f/contracts/plugins/assets/curve/PoolTokens.sol#L138

```diff
-        // I know this lots extremely verbose and quite silly, but it actually makes sense:
+        // I know this is extremely verbose and quite silly, but it actually makes sense:
```
## `FiatCollateral.alreadyDefaulted` turns obsolete when block.timestamp > type(uint48).max
Although this will take a very long time to happen, the following function will default to `true`, i.e. `DISABLED` when `block.timestamp` exceeds `type(uint48).max` considering `_whenDefault` would at most be assigned `NEVER`, which is `type(uint48).max`.  

https://github.com/reserve-protocol/protocol/blob/9ee60f142f9f5c1fe8bc50eef915cf33124a534f/contracts/plugins/assets/FiatCollateral.sol#L194-L196

```solidity
    function alreadyDefaulted() internal view returns (bool) {
        return _whenDefault <= block.timestamp;
    }
``` 
## Comment and code mismatch on the `@return` NatSpec of `EURFiatCollateral.tryPrice`
The [unit of `pegPrice`](https://github.com/reserve-protocol/protocol/blob/9ee60f142f9f5c1fe8bc50eef915cf33124a534f/contracts/plugins/assets/EURFiatCollateral.sol#L64-L65) in `EURFiatCollateral.tryPrice` is `target/ref` but it is stated otherwise in the function NatSpec.

https://github.com/reserve-protocol/protocol/blob/9ee60f142f9f5c1fe8bc50eef915cf33124a534f/contracts/plugins/assets/EURFiatCollateral.sol#L37

```diff
-    /// @return pegPrice {UoA/ref}
+    /// @return pegPrice {target/ref}
```
## Comment and code mismatch on the `else` clause of `Asset.lotPrice`
`<=` and `>=` have respectively been used in the `if` and `else if` clauses, making the `=` sign of the inequalities in the commented `else` clause erroneous. 

https://github.com/reserve-protocol/protocol/blob/9ee60f142f9f5c1fe8bc50eef915cf33124a534f/contracts/plugins/assets/Asset.sol#L139-L154

```diff
            uint48 delta = uint48(block.timestamp) - lastSave; // {s}
            if (delta <= oracleTimeout) {
                lotLow = savedLowPrice;
                lotHigh = savedHighPrice;
            } else if (delta >= oracleTimeout + priceTimeout) {
                return (0, 0); // no price after full timeout
            } else {
-                // oracleTimeout <= delta <= oracleTimeout + priceTimeout
+                // oracleTimeout < delta < oracleTimeout + priceTimeout

                // {1} = {s} / {s}
                uint192 lotMultiplier = divuu(oracleTimeout + priceTimeout - delta, priceTimeout);

                // {UoA/tok} = {UoA/tok} * {1}
                lotLow = savedLowPrice.mul(lotMultiplier);
                lotHigh = savedHighPrice.mul(lotMultiplier);
            }
```
## Comment and code mismatch on dimensional analysis
The comment below is missing out the units pertaining to the division by [`one`](https://github.com/reserve-protocol/protocol/blob/9ee60f142f9f5c1fe8bc50eef915cf33124a534f/contracts/plugins/assets/erc20/RewardableERC20.sol#L22). 

https://github.com/reserve-protocol/protocol/blob/9ee60f142f9f5c1fe8bc50eef915cf33124a534f/contracts/plugins/assets/erc20/RewardableERC20.sol#L65-L66

```diff
-            // {qRewards} = {qRewards/share} * {qShare}
-            // {qRewards} = {qRewards/share} * {qShare} / {qShare/share}
            _accumuatedRewards += (delta * shares) / one;
```
## Capitalized state variables
State variable names should be Camel-cased instead of fully capitalized (typically used on constants and immutables).

Here are some of the instances entailed:

https://github.com/reserve-protocol/protocol/blob/9ee60f142f9f5c1fe8bc50eef915cf33124a534f/contracts/plugins/assets/aave/StaticATokenLM.sol#L64-L68

```solidity
    ILendingPool public override LENDING_POOL;
    IAaveIncentivesController public override INCENTIVES_CONTROLLER;
    IERC20 public override ATOKEN;
    IERC20 public override ASSET;
    IERC20 public override REWARD_TOKEN;
```
## Inadequate NatSpec
Solidity contracts can use a special form of comments, i.e., the Ethereum Natural Language Specification Format (NatSpec) to provide rich documentation for functions, return variables, and more. Please visit the following link for further details:

https://docs.soliditylang.org/en/v0.8.16/natspec-format.html

It's generally incomplete throughout all codebases. Consider fully equipping all contracts with a complete set of NatSpec to better facilitate users/developers interacting with the protocol's smart contracts.

## Non-compliant contract layout with Solidity's Style Guide
According to Solidity's Style Guide below:

https://docs.soliditylang.org/en/v0.8.17/style-guide.html

In order to help readers identify which functions they can call, and find the constructor and fallback definitions more easily, functions should be grouped according to their visibility and ordered in the following manner:

constructor, receive function (if exists), fallback function (if exists), external, public, internal, private

And, within a grouping, place the `view` and `pure` functions last.

Additionally, inside each contract, library or interface, use the following order:

type declarations, state variables, events, modifiers, functions

Consider adhering to the above guidelines for all contract instances entailed.