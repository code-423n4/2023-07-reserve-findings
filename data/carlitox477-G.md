# `FiatCollateral::status`: Cache `_whenDefault` to avoid unnecessary storage access
```diff
    // FiatCollateral
    function status() public view returns (CollateralStatus) {
+       uint48 __whenDefault = _whenDefault
-       if (_whenDefault == NEVER) {
+       if (__whenDefault == NEVER) {
            return CollateralStatus.SOUND;
-       } else if (_whenDefault > block.timestamp) {
+       } else if (__whenDefault > block.timestamp) {
            return CollateralStatus.IFFY;
        } else {
            return CollateralStatus.DISABLED;
        }
    }
```

# `AppreciatingFiatCollateral::refresh` cache `exposedReferencePrice` to avoid unnecessary storage access

```diff
    function refresh() public virtual override {
        if (alreadyDefaulted()) {
            // continue to update rates
            exposedReferencePrice = _underlyingRefPerTok().mul(revenueShowing);
            return;
        }

        CollateralStatus oldStatus = status();

        // Check for hard default
        // must happen before tryPrice() call since `refPerTok()` returns a stored value

        // revenue hiding: do not DISABLE if drawdown is small
        uint192 underlyingRefPerTok = _underlyingRefPerTok();

        // {ref/tok} = {ref/tok} * {1}
        uint192 hiddenReferencePrice = underlyingRefPerTok.mul(revenueShowing);

+       uint192 _exposedReferencePrice = exposedReferencePrice;
        // uint192(<) is equivalent to Fix.lt
-       if (underlyingRefPerTok < exposedReferencePrice) {
+       if (underlyingRefPerTok < _exposedReferencePrice) {
            exposedReferencePrice = hiddenReferencePrice;
            markStatus(CollateralStatus.DISABLED);
-       } else if (hiddenReferencePrice > exposedReferencePrice) {
+       } else if (hiddenReferencePrice > _exposedReferencePrice) {
            exposedReferencePrice = hiddenReferencePrice;
        }

        // Check for soft default + save prices
        try this.tryPrice() returns (uint192 low, uint192 high, uint192 pegPrice) {
            // {UoA/tok}, {UoA/tok}, {target/ref}
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

            // If the price is below the default-threshold price, default eventually
            // uint192(+/-) is the same as Fix.plus/minus
            if (pegPrice < pegBottom || pegPrice > pegTop || low == 0) {
                markStatus(CollateralStatus.IFFY);
            } else {
                markStatus(CollateralStatus.SOUND);
            }
        } catch (bytes memory errData) {
            // see: docs/solidity-style.md#Catching-Empty-Data
            if (errData.length == 0) revert(); // solhint-disable-line reason-string
            markStatus(CollateralStatus.IFFY);
        }

        CollateralStatus newStatus = status();
        if (oldStatus != newStatus) {
            emit CollateralStatusChanged(oldStatus, newStatus);
        }
    }
```

# `RewardableERC20::_claimAccountRewards` cache `accumulatedRewards[account]` and `lastRewardBalance` to prevent double storage access
```diff
    function _claimAccountRewards(address account) internal {
+       uint256 _accumulatedRewards = accumulatedRewards[account];
-       uint256 claimableRewards = accumulatedRewards[account] - claimedRewards[account];
+       uint256 claimableRewards = _accumulatedRewards - claimedRewards[account];

        emit RewardsClaimed(IERC20(address(rewardToken)), claimableRewards);

        if (claimableRewards == 0) {
            return;
        }

-       claimedRewards[account] = accumulatedRewards[account];
+       claimedRewards[account] = _accumulatedRewards

        uint256 currentRewardTokenBalance = rewardToken.balanceOf(address(this));

        // This is just to handle the edge case where totalSupply() == 0 and there
        // are still reward tokens in the contract.
+       uint256 _lastRewardBalance = lastRewardBalance;
-       uint256 nonDistributed = currentRewardTokenBalance > lastRewardBalance
-           ? currentRewardTokenBalance - lastRewardBalance
-           : 0;
+       uint256 nonDistributed = currentRewardTokenBalance > _lastRewardBalance
+           ? currentRewardTokenBalance - _lastRewardBalance
+           : 0;

        rewardToken.safeTransfer(account, claimableRewards);

        currentRewardTokenBalance = rewardToken.balanceOf(address(this));
        lastRewardBalance = currentRewardTokenBalance > nonDistributed
            ? currentRewardTokenBalance - nonDistributed
            : 0;
    }
```

# `CTokenFiatCollateral::_underlyingRefPerTok` Substract `10` instead of adding `8` and substracting `18`

```diff
    function _underlyingRefPerTok() internal view override returns (uint192) {
        uint256 rate = cToken.exchangeRateStored();
-       int8 shiftLeft = 8 - int8(referenceERC20Decimals) - 18;
+       int8 shiftLeft = - int8(referenceERC20Decimals) - 10;
        return shiftl_toFix(rate, shiftLeft);
    }
```

# `WrappedERC20:_mint`: `_balances[account] += amount` can be unchecked
If we reach this line, it is because the total supply did not overflow, therefore the increased balance cannot overflow. We can use an unchecked block to save gas.

```diff
-   _balances[account] += amount;
+   unchecked{
+       _balances[account] += amount;
+   }
```

# `WrappedERC20:_burn`: `_totalSupply -= amount` can be unchecked
If we reach this line, it is because the account individual balance did not underflow, therefore the decreased total supply cannot underflow. We can use an unchecked block to save gas. More over, we can refactor part of this code to use just one unchecked block
```diff
-   if (amount > accountBalance) revert ExceedsBalance(amount);
-    unchecked {
-       _balances[account] = accountBalance - amount;
-   }
+    _balances[account] = accountBalance - amount;
+     unchecked {
+       _totalSupply -= amount;
+   }
```

# `StaticATokenLM::claimRewards`: Cache `REWARD_TOKEN`
The state variable is accessed 3 times, this can be reduced to just one:

```diff
    function claimRewards() external virtual nonReentrant {
        if (address(INCENTIVES_CONTROLLER) == address(0)) {
            return;
        }
+       IERC20 _REWARD_TOKEN = REWARD_TOKEN;
-       uint256 oldBal = REWARD_TOKEN.balanceOf(msg.sender);
+       uint256 oldBal = _REWARD_TOKEN.balanceOf(msg.sender);
        _claimRewardsOnBehalf(msg.sender, msg.sender, true);
-       emit RewardsClaimed(REWARD_TOKEN, REWARD_TOKEN.balanceOf(msg.sender) - oldBal);
+       emit RewardsClaimed(_REWARD_TOKEN, _REWARD_TOKEN.balanceOf(msg.sender) - oldBal);
    }
```


# `StaticATokenLM`: For state variables assigned only in the constructor, change them to immutable variable
Currently, state variables `LENDING_POOL`, `INCENTIVES_CONTROLLER`, `ATOKEN`, `ASSET`, `REWARD_TOKEN` are only modified in the constructor, they should change their name, change its visibility to private/internal, be declared as immutable variables. Then, in order to implement the functions from `IStaticATokenLM` with the same name than current variables, just return the new variables.

# `StaticATokenLM::_updateRewards`, `StaticATokenLM::_collectAndUpdateRewards` and `StaticATokenLM::_getPendingRewards`: `rewardsAccrued` assignation can be simplified
`rewardsAccrued` happens to be the same than:
* `freshRewards.wadToRay()` in `StaticATokenLM::_updateRewards`, therefore the assignation replaced for: `uint256 rewardsAccrued = freshRewards.wadToRay();` 
* `freshReward.wadToRay()` in `StaticATokenLM::_collectAndUpdateRewards`, therefore the assignation _getPendingRewards for: `uint256 rewardsAccrued = freshReward.wadToRay();` 
