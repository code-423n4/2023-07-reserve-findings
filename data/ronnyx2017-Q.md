# 1. cbETH UoA should be USD
code lines: https://github.com/reserve-protocol/protocol/blob/9ee60f142f9f5c1fe8bc50eef915cf33124a534f/contracts/plugins/assets/cbeth/CBETHCollateral.sol#L53-L56

According to the cbeth README.md, the UoA and ref of the chETH are both ETH, but the `CBETHCollateral.tryPrice()` uses a chainlinkFeed to get the `UoA/ref`. It is contradictory. The UoA should be USD.


# 2. SafeERC20.safeApprove can lead to Morpho deposit revert
code lines: https://github.com/reserve-protocol/protocol/blob/9ee60f142f9f5c1fe8bc50eef915cf33124a534f/contracts/2023-07-reserve/protocol/contracts/plugins/assets/morpho-aave/MorphoTokenisedDeposit.sol#L59

The `SafeERC20.safeApprove` can only use to init an allowance, if the `allowance!=0` it will revert.
```solidity
require(
    (value == 0) || (token.allowance(address(this), spender) == 0),
    "SafeERC20: approve from non-zero to non-zero allowance"
);
```

# 3. MorphoTokenisedDeposit _claimAssetRewards
code lines: https://github.com/reserve-protocol/protocol/blob/9ee60f142f9f5c1fe8bc50eef915cf33124a534f/contracts/2023-07-reserve/protocol/contracts/plugins/assets/morpho-aave/MorphoTokenisedDeposit.sol#L44

The `MorphoTokenisedDeposit._claimAssetRewards()` function is empty:
```solidity
function _claimAssetRewards() internal virtual override {}
```
Although the Morpho uses a rewards scheme that requires the results of off-chain computation to be piped into an on-chain function, a empty `_claimAssetRewards` function will cause users to lose part of rewards when they withdraw if rewards are not synchronized for a long time. 

Suggest to add a rewards last update timestamp check in the _claimAssetRewards function.

# 4. RewardableERC20Wrapper ERC777 reentrancy attack
code lines: https://github.com/reserve-protocol/protocol/blob/9ee60f142f9f5c1fe8bc50eef915cf33124a534f/contracts/2023-07-reserve/protocol/contracts/plugins/assets/erc20/RewardableERC20Wrapper.sol#L44-L46

In the `RewardableERC20Wrapper.deposit` function, the `_mint` operation is called before underlying tokens transfer:
```solidty
_mint(_to, _amount); // does balance checkpointing
underlying.safeTransferFrom(msg.sender, address(this), _amount);
_afterDeposit(_amount, _to);
```
And there is not a reentrancy guard. So if the underlying token is an ERC777 standard token, an attacker can register a `tokensToSend` hook on the ERC-1820 Registry. And the attacker can mint and use any number of wrapper tokens before transfering any underlying token to the RewardableERC20Wrapper.

# 5.The low price of RTokenAsset from price()/lotPrice() is lower than the actual value when the basket is not ready

code lines: https://github.com/reserve-protocol/protocol/blob/9ee60f142f9f5c1fe8bc50eef915cf33124a534f/contracts/plugins/assets/RTokenAsset.sol#L178-L214

`RTokenAsset.price()/lotPrice()` uses `basketRange()` to get the BU range. And the `basketRange()` gets the basketsHeld from BasketHandler and uses `RecollateralizationLibP1.basketRange` to get the range with all the assets when the RToken is undercollateralized.

But if the basket is not ready ( BasketHandler.disabled = true which occurs during the basket switching, or there is any collateral is CollateralStatus.DISABLED ), the `BasketHandler.basketsHeldBy` will return `BasketRange(FIX_ZERO, FIX_MAX)`. A range.bottom = 0 leads to every collateral uses the total balance to lift the range.bottom. And because the range.bottom is calculated in the pessimistic case:
```solidity
uint192 anchor = ctx.quantities[i].mul(ctx.basketsHeld.bottom, FLOOR);

uint192 val = low.mul(bal - anchor, FLOOR);

// (3) Buy BUs at their high price with the remaining value
// (4) Assume maximum slippage in trade
// {BU} = {UoA} * {1} / {UoA/BU}
range.bottom += val.mulDiv(FIX_ONE.minus(ctx.maxTradeSlippage), buPriceHigh, FLOOR);
```
The delta value is divided by the high BU price as the increments of range.bottom. So using the total collateral balance, instead of the `bal - anchor` vaule(bacuase the anchor is 0 now), will lead to the range.bottom is lower than the actual value.

# 6. The interface IRewardable is not specified in the contract definition of ConvexStakingWrapper
code lines: https://github.com/reserve-protocol/protocol/blob/9ee60f142f9f5c1fe8bc50eef915cf33124a534f/contracts/plugins/assets/curve/cvx/vendor/ConvexStakingWrapper.sol#L44

`CurveStableCollateral.claimRewards` needs the config.erc20 should be an IRewardable implementation:
```
function claimRewards() external override(Asset, IRewardable) {
    IRewardable(address(erc20)).claimRewards();
}
```
But ConvexStakingWrapper doesn't specify this in the definition. Although it has implemented the claimRewards function.

# 7. Curve Read-only Reentrancy can increase the price of some CurveStableCollateral
If the curve pool of a CurveStableCollateral is a Plain Pool with a native gas token, just like eth/stETH pool: https://etherscan.io/address/0xdc24316b9ae028f1497c275eb9192a3ea0f67022#code 
the price can be manipulated by Curve Read-only Reentrancy.

A example is eth/stETH pool, in its `remove_liquidity` function:
```soldity
# snippet from remove_liquidity
CurveToken(lp_token).burnFrom(msg.sender, _amount)
for i in range(N_COINS):
    value: uint256 = amounts[i] * _amount / total_supply
    if i == 0:
        raw_call(msg.sender, b"", value=value)
    else:
    assert ERC20(self.coins[1]).transfer(msg.sender, value)
```

First, LP tokens are burned. Next, each token is transferred out to the msg.sender. Given that ETH will be the first coin transferred out, token balances and total LP token supply will be inconsistent during the execution of the fallback function.

The `CurveStableCollateral` uses `total underlying token balance value / lp supply` to calculate the lp token price:
```
    (uint192 aumLow, uint192 aumHigh) = totalBalancesValue();

    // {tok}
    uint192 supply = shiftl_toFix(lpToken.totalSupply(), -int8(lpToken.decimals()));
    // We can always assume that the total supply is non-zero

    // {UoA/tok} = {UoA} / {tok}
    low = aumLow.div(supply, FLOOR);
    high = aumHigh.div(supply, CEIL);
```

So the price will be higher than the actual value becuase the other assets(except eth) are still in the pool but the lp supply has been cut down during the `remove_liquidity` fallback.

