### [L-01] Unsafe max approve
It's not recommended to approve `type(uint256).max` for safety. We should approve a relevant amount every time.

- https://github.com/reserve-protocol/protocol/blob/9ee60f142f9f5c1fe8bc50eef915cf33124a534f/contracts/plugins/assets/aave/StaticATokenLM.sol#L105

```solidity
ASSET.safeApprove(address(pool), type(uint256).max);
```

- https://github.com/reserve-protocol/protocol/blob/9ee60f142f9f5c1fe8bc50eef915cf33124a534f/contracts/plugins/assets/stargate/StargateRewardableWrapper.sol#L39

```solidity
pool_.approve(address(stakingContract_), type(uint256).max);
```

### [L-02] Possible reentrancy
Users might manipulate `wrapperPostPrinc` inside the transfer hook. With the current `underlyingComet` token, there is no impact as it doesn't have any hook but it's recommended to add a `nonReentrant` modifier to `_deposit()/_withdraw()`.

- https://github.com/reserve-protocol/protocol/blob/9ee60f142f9f5c1fe8bc50eef915cf33124a534f/contracts/plugins/assets/compoundv3/CusdcV3Wrapper.sol#L96

```solidity
IERC20(address(underlyingComet)).safeTransferFrom(src, address(this), amount);
```

- https://github.com/reserve-protocol/protocol/blob/9ee60f142f9f5c1fe8bc50eef915cf33124a534f/contracts/plugins/assets/compoundv3/CusdcV3Wrapper.sol#L154

```solidity
IERC20(address(underlyingComet)).safeTransfer(dst, (amount / 10) * 10);
```

### [L-03] Unsafe downcasting
`feeds[i].length` might be downcasted wrongly when it's greater than `type(uint8).max`. `maxFeedsLength()` might pass the [requirement](https://github.com/reserve-protocol/protocol/blob/9ee60f142f9f5c1fe8bc50eef915cf33124a534f/contracts/plugins/assets/curve/PoolTokens.sol#L105) when `config.feeds` has many elements(like 256).

- https://github.com/reserve-protocol/protocol/blob/9ee60f142f9f5c1fe8bc50eef915cf33124a534f/contracts/plugins/assets/curve/PoolTokens.sol#L330

```solidity
maxLength = uint8(Math.max(maxLength, feeds[i].length));
```

### [L-04] Reverts on 0 transfer
It deposits 0 amount to the staking contract to claim rewards but it might revert [during the 0 transfer](https://github.com/stargate-protocol/stargate/blob/main/contracts/LPStaking.sol#L161). There is no problem with the current `lpToken` but good to keep in mind.

- https://github.com/reserve-protocol/protocol/blob/9ee60f142f9f5c1fe8bc50eef915cf33124a534f/contracts/plugins/assets/stargate/StargateRewardableWrapper.sol#L48

```solidity
stakingContract.deposit(poolId, 0);
```

### [L-05] Wrong comment
- https://github.com/reserve-protocol/protocol/blob/9ee60f142f9f5c1fe8bc50eef915cf33124a534f/contracts/plugins/assets/Asset.sol#L146

```solidity
// oracleTimeout <= delta <= oracleTimeout + priceTimeout ==========> oracleTimeout < delta < oracleTimeout + priceTimeout
```

- https://github.com/reserve-protocol/protocol/blob/9ee60f142f9f5c1fe8bc50eef915cf33124a534f/contracts/plugins/assets/compoundv3/WrappedERC20.sol#L298

```solidity
     * - when `from` is zero, `amount` tokens will be minted for `to`. //@audit should remove, no hook if `from` or `to` is zero
     * - when `to` is zero, `amount` of ``from``'s tokens will be burned.
```

### [L-06] Unsafe permission
`WrappedERC20` uses a permission mechanism and operators can have an infinite allowance once approved. It might be inconvenient/dangerous for some users.

- https://github.com/reserve-protocol/protocol/blob/9ee60f142f9f5c1fe8bc50eef915cf33124a534f/contracts/plugins/assets/compoundv3/CusdcV3Wrapper.sol#L84

```solidity
    function _deposit(
        address operator,
        address src,
        address dst,
        uint256 amount
    ) internal {
        if (!hasPermission(src, operator)) revert Unauthorized(); //@audit infinite allowance
        ...
    }
```

- https://github.com/reserve-protocol/protocol/blob/9ee60f142f9f5c1fe8bc50eef915cf33124a534f/contracts/plugins/assets/compoundv3/CusdcV3Wrapper.sol#L140

```solidity
    function _withdraw(
        address operator,
        address src,
        address dst,
        uint256 amount
    ) internal {
        if (!hasPermission(src, operator)) revert Unauthorized(); //@audit infinite allowance
        ...
    }
```

### [L-07] Typical first depositor issue in `RewardableERC4626Vault`
It's recommended to follow the [instructions](https://github.com/reserve-protocol/protocol/blob/9ee60f142f9f5c1fe8bc50eef915cf33124a534f/contracts/vendor/oz/ERC4626.sol#L39-L43).

- https://github.com/reserve-protocol/protocol/blob/9ee60f142f9f5c1fe8bc50eef915cf33124a534f/contracts/plugins/assets/erc20/RewardableERC4626Vault.sol#L20

```solidity
abstract contract RewardableERC4626Vault is ERC4626, RewardableERC20 {}
```

### [L-08] Needless accrue for src
We don't need to accrue for `src` because we don't use any information of `src` and that info will be accrued during the [underlyingComet transfer](https://github.com/compound-finance/comet/blob/main/contracts/Comet.sol#L942).

- https://github.com/reserve-protocol/protocol/blob/9ee60f142f9f5c1fe8bc50eef915cf33124a534f/contracts/plugins/assets/compoundv3/CusdcV3Wrapper.sol#L91

```solidity
    underlyingComet.accrueAccount(address(this));
    underlyingComet.accrueAccount(src); //@audit needless accrue

    CometInterface.UserBasic memory wrappedBasic = underlyingComet.userBasic(address(this));
    int104 wrapperPrePrinc = wrappedBasic.principal;

    IERC20(address(underlyingComet)).safeTransferFrom(src, address(this), amount); 
```

### [N-01] Typo
`_accumuatedRewards` => `_accumulatedRewards`

- https://github.com/reserve-protocol/protocol/blob/9ee60f142f9f5c1fe8bc50eef915cf33124a534f/contracts/plugins/assets/erc20/RewardableERC20.sol#L57

```solidity
uint256 _accumuatedRewards = accumulatedRewards[account];
```