L-01: StaticATokenLM if out-of-gas may result in INCENTIVES_CONTROLLER == address(0)


In the constructor method of `StaticATokenLM`, a try/catch is executed

```solidity
    constructor(
        ILendingPool pool,
        address aToken,
        string memory staticATokenName,
        string memory staticATokenSymbol
    ) public {
...
        try IAToken(aToken).getIncentivesController() returns (
            IAaveIncentivesController incentivesController
        ) {
            if (address(incentivesController) != address(0)) {
@>              INCENTIVES_CONTROLLER = incentivesController;
@>              REWARD_TOKEN = IERC20(INCENTIVES_CONTROLLER.REWARD_TOKEN());
            }
@>      } catch {}

        ASSET = IERC20(IAToken(aToken).UNDERLYING_ASSET_ADDRESS());
        ASSET.safeApprove(address(pool), type(uint256).max);
    }    
```

If `try` gets `out-of-gas` due to a gas estimation error, but the contract deploys normally due to the `1/64` reserved gas, then the contract will deploy normally.

This may result in the contract deploying and actually having `INCENTIVES_CONTROLLER`, but getting `INCENTIVES_CONTROLLER==address(0)`.

Suggestion.
If `out-of-gas` then revert

```solidity
    constructor(
        ILendingPool pool,
        address aToken,
        string memory staticATokenName,
        string memory staticATokenSymbol
    ) public {
...
        try IAToken(aToken).getIncentivesController() returns (
            IAaveIncentivesController incentivesController
        ) {
            if (address(incentivesController) != address(0)) {
                INCENTIVES_CONTROLLER = incentivesController;
                REWARD_TOKEN = IERC20(INCENTIVES_CONTROLLER.REWARD_TOKEN());
            }
-       } catch {}
+       } catch (bytes memory errData) {
+           if (errData.length == 0) revert(); // out-of-gas
+       }

        ASSET = IERC20(IAToken(aToken).UNDERLYING_ASSET_ADDRESS());
        ASSET.safeApprove(address(pool), type(uint256).max);
    
```

L-02: MorphoFiatCollateral._underlyingRefPerTok() ERC4626 first user inflation attack


`MorphoFiatCollateral._underlyingRefPerTok()` using `convertToAssets()` as `_underlyingRefPerTok()`

First user executable, common `ERC4626` First user `inflation attack`

1. donate `asset`, raising `totalAssets()`
2. perform MorphoFiatCollateral.refresh() to get high `_underlyingRefPerTok()`
Formula: `shares.mulDiv(totalAssets() + 1, totalSupply() + 10**_decimalsOffset(), rounding);`
`_underlyingRefPerTok()` = 1e18 * totalAssets() / 1 
3. Execute `ERC4626.deposit()` and `ERC4626.draw()` to retrieve all `assets`.
4. run MorphoFiatCollateral.refresh()` to get the normal `_underlyingRefPerTok()`, but lower than the value in step 2
5. MorphoFiatCollateral goes `DISABLED`.

Recommendation.
Deploy with pre-stored shares


L-03: StargatePoolFiatCollateral missing `revenueHiding` mechanisms

Currently `StargatePoolFiatCollateral` does not have a `revenueHiding` mechanism similar to `AppreciatingFiatCollateral` to retaliate against minor revenue drops (e.g. caused by loss of precision).

It is recommended to inherit directly from `AppreciatingFiatCollateral.sol`, do not override `refresh`, and directly use `AppreciatingFiatCollateral.sol`'s refersh() method.


L-04: MorphoTokenisedDeposit override _decimalsOffset() ==0 increase ERC4626 `inflation attack` risk

`MorphoTokenisedDeposit._decimalsOffset` override `RewardableERC4626Vault._decimalsOffset()`
and return 0
```solidity
    function _decimalsOffset() internal view virtual override returns (uint8) {
        return 0;
    }
```


It is recommended not to override, but to continue to use `RewardableERC4626Vault._decimalsOffset() ==9`



L-05: MorphoTokenisedDeposit._deposit() It is not recommended to use safeApprove, it is recommended to use the `safeIncreaseAllowance()`

`SafeERC20.safeApprove()` Determines if the current allowance is equal to 0, applies to the first `approve`, since `_deposit()` will be used repeatedly it is recommended to use `safeIncreaseAllowance()`.
to avoid the risk of revert if the allowance is not used up.

```solidity
    /**
     * @dev Deprecated. This function has issues similar to the ones found in
     * {IERC20-approve}, and its usage is discouraged.
     *
     * Whenever possible, use {safeIncreaseAllowance} and
     * {safeDecreaseAllowance} instead.
     */
    function safeApprove(
        IERC20 token,
        address spender,
        uint256 value
    ) internal {
        // safeApprove should only be called when setting an initial allowance,
        // or when resetting it to zero. To increase and decrease it, use
        // 'safeIncreaseAllowance' and 'safeDecreaseAllowance'
        require(
@>           (value == 0) || (token.allowance(address(this), spender) == 0),
            "SafeERC20: approve from non-zero to non-zero allowance"
        );
        _callOptionalReturn(token, abi.encodeWithSelector(token.approve.selector, spender, value));
    }
```