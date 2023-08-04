# `EURFiatCollateral::tryPrice` should change its return natspec
Given that [`target != UoA`](https://github.com/reserve-protocol/protocol/blob/9ee60f142f9f5c1fe8bc50eef915cf33124a534f/contracts/plugins/assets/EURFiatCollateral.sol#L11), next modification to comments should be done

```diff
// EURFiatCollateral
    /// @return high {UoA/tok} The high price estimate
-   /// @return pegPrice {UoA/ref}
+   /// @return pegPrice {target/ref}
    function tryPrice()
    // ...
```

# `MorphoAaveV2TokenisedDeposit::getMorphoPoolBalance` change parameter name to storage variable shadowing
`poolToken` is also the of a state variable, therefore its names should be changed to improve code readability

```diff
-   function getMorphoPoolBalance(address poolToken)
+   function getMorphoPoolBalance(address _poolToken)
        internal
        view
        virtual
        override
        returns (uint256)
    {
        (, , uint256 supplyBalance) = morphoLens.getCurrentSupplyBalanceInOf(
-           poolToken,
+           _poolToken,
            address(this)
        );
        return supplyBalance;
    }
```

# `RewardableERC20Wrapper::deposit` if underlaying token introduce a callback on transfer, it may affect the behaviour of `_afterDeposit`
Given that `_afterDeposit` is not implemented in `RewardableERC20Wrapper`, if `underlying` is a token which introduces a callback on transfers (for instance, an ERC-777), it may affect he behaviour of function `_afterDeposit`. It would be prudent to add a `noReentrant` modifier to `deposit`.

# `MorphoTokenisedDeposit::rewardTokenBalance` behavior can be affected by reentrancy though `RewardableERC20::_claimAssetRewards`
`MorphoTokenisedDeposit::rewardTokenBalance` do a call to `RewardableERC20::_claimAndSyncRewards`. This function first query the supply, and the do ca call to `RewardableERC20::_claimAssetRewards` which is pending to be implemented. If this make an external call, it may allow to reenter to this method. Adding a nonReentrant modifier might to this function should be considered

# Correct comment spelling
In `StaticATokenLM::_getPendingRewards` correct [return comment spelling](https://github.com/reserve-protocol/protocol/blob/9ee60f142f9f5c1fe8bc50eef915cf33124a534f/contracts/plugins/assets/aave/StaticATokenLM.sol#L557)
```diff
-   * @return The amound of pending rewards in RAY
+   * @return The amount of pending rewards in RAY
```

# `CurveGaugeWrapper`: Change parameters `_name` and `_symbol` name to avoid confusion with inherited state variables from `ERC20`
`_name` and `_symbol` are state variables from `ERC20` contract. In order to avoid confusion, it would be advisable to change their names to `__name` and `__symbol`
