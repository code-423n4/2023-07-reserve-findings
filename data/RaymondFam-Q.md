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
## `FiatCollateral.alreadyDefaulted` turns obsolete when block.timestamp > type(uint48).max
Although this will take a very long time to happen, the following function will default to `true`, i.e. `DISABLED` when `block.timestamp` exceeds `type(uint48).max` considering `_whenDefault` would at most be assigned `NEVER`, which is `type(uint48).max`.  

https://github.com/reserve-protocol/protocol/blob/9ee60f142f9f5c1fe8bc50eef915cf33124a534f/contracts/plugins/assets/FiatCollateral.sol#L194-L196

```solidity
    function alreadyDefaulted() internal view returns (bool) {
        return _whenDefault <= block.timestamp;
    }
``` 
## Comment and code mismatch
The [unit of `pegPrice`](https://github.com/reserve-protocol/protocol/blob/9ee60f142f9f5c1fe8bc50eef915cf33124a534f/contracts/plugins/assets/EURFiatCollateral.sol#L64-L65) in `EURFiatCollateral.tryPrice` is `target/ref` but it is stated otherwise in the function NatSpec.

https://github.com/reserve-protocol/protocol/blob/9ee60f142f9f5c1fe8bc50eef915cf33124a534f/contracts/plugins/assets/EURFiatCollateral.sol#L37

```diff
-    /// @return pegPrice {UoA/ref}
+    /// @return pegPrice {target/ref}
```