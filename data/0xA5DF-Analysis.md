## Overview of the structure of the project
The project can be categorized into:
* Parent contracts - such as Asset, FiatCollateral etc.
* Simple plugins - plugins that contain very little code on top of the parent contracts (e.g. AnkrETH, cbETH)
* Complicated plugins - those that contain lots of new code (Compound, AAVE etc.)

Some of the parent contracts were reviewed already in the January audit, though some of them had some changes since then.


## Having many external projects
One important thing to notice about this project is that it interacts with many different projects - each plugin has an underlying token that it interacts with it.
That means that in order to find bugs that are related to the specific structure of the projects it's best to have a researcher who has a deep knowledge of that project to review the code.
It might be a good idea to ask the wardens that have participated in this contest what projects they're familiar with and to what extent, and for the projects that not enough wardens are familiar with - get another small audit by somebody who is familiar with it.
This is especially relevant for the more complicated plugins, as the more code the higher the chances of it to contain bugs.

Another recommendation is to closely follow those projects news or updates, as any event that affects the collateral tokens might affect Reserve as well.


## RewardableERC20Wrapper

This contract is used in the following projects:
* Compound V2
* Curve
* Stargate

I was told by the sponsor that this is because the core protocol expects the token to have a `claimReward()` function, so the wrapper was created in order to add this function for assets that don't have it.

This wrapper create some security hassle, as it contains lots of additional code and it holds the assets directly.
It might be a better idea for the core protocol to revert back to the way it was done in V2 (i.e. a delegate call to the `Asset` contract) in order to save this hassle.

### Time spent:
25 hours