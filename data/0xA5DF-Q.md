## `RTokenAsset.price()` should return FIX_MAX if this is the BU price

When the `basketHandler.price()` high price is FIX_MAX then the returned high price should be FIX_MAX as well.
This isn't always the case since the BU price [is then multiplied by the basket range and divided by total supply](https://github.com/reserve-protocol/protocol/blob/9ee60f142f9f5c1fe8bc50eef915cf33124a534f/contracts/plugins/assets/RTokenAsset.sol#L69), in case that baskets needed is less than total supply (i.e. there was a previous haircut) then the returned value might be lower than FIX_MAX.

This might cause issues when trying to detect a faulty asset by the returned price value.

## Replace the `assert()` at `RTokenAsset.tryPrice()` with a string or custom error
When assert reverts it reverts with an empty error, this means that when this is called from `price()` [it'd be mistaken to be a revert due to out of gas error](https://github.com/reserve-protocol/protocol/blob/9ee60f142f9f5c1fe8bc50eef915cf33124a534f/contracts/plugins/assets/Asset.sol#L118).
Therefore it's best to replace it with a non-empty revert.

## `STATIC_ATOKEN_IMPL` might be too long for a symbol
[`StaticATokenLM`](https://github.com/reserve-protocol/protocol/blob/9ee60f142f9f5c1fe8bc50eef915cf33124a534f/contracts/plugins/assets/aave/StaticATokenLM.sol#L33) uses `STATIC_ATOKEN_IMPL` as the ERC20 symbol.
While I'm not aware of any issue with using 18 characters for a symbol, most ERC20s use 3-6 characters, so it might be better to follow the common length and shorten the symbol.

## Compound V3’s accrue account runs after the balance check
[code](https://github.com/reserve-protocol/protocol/blob/9ee60f142f9f5c1fe8bc50eef915cf33124a534f/contracts/plugins/assets/compoundv3/CusdcV3Wrapper.sol#L78-L91)
The check for the balance check is done before calling `accrueAccount()`, so there might be a case where the user would have enough balance once `accrueAccount()` is called, but the function would revert because the balance is checked beforehand.

Side notes:
* Changing this would break CEI pattern
* My assumption that `accrueAccount()` increases user's balance might be wrong, I'm not very familiar with the way Compound V3 works.