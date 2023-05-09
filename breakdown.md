# AAVE Code Explained Part 1
Aave is a decentralized non-custodial liquidity market protocol where users can participate as suppliers or borrowers.

## Suppliers & Borrowers
Suppliers provide liquidity to the market and earn a passive income, while borrowers are able to borrow unexpected expenses, leveraging their holding.

Liquidity is at the heart of the Aave Protocol as it enables the protocol's operation and user experience.

The liquidity of the protocol is measured by the availability of assets for basic protocol operations such as borrowing of assets backed by collateral and claiming of supplied assets along with accrued yield.
- A lack of liquidity will block operations.

## Contracts Overview
The Aave Porotocol V3 contracts are divided into two repositories
1. aave-v3-core
2. aave-v3-periphery

### aave-v3-core
The aave-v3-core host or contains the core protocol V3 contracts that contains the logic for supply, borrow, liquidation, flashloans, a/s/v tokens, portal, pool configuration, oracles and interest rate strategies.

The core protocol contracts fall in the following 4 categories:
- Configuration
- Pool Logic
- Tokenization
- Misc

### aave-v3-periphery
In the periphery repository, you will find the contracts related towards rewards, UI data provider, wallet balance provider and WETH gateway.

The periphery contracts have two categories
- Rewards
- Misc

## What can we do with Aave Protocol
- Supply and Earn:
By supplying you will earn passive income based on the market borrowing demand

- Borrow:
Additionally, supplying assets allows you to borrow by using your supplied assets as a collateral


### Pool.sol Smart Contract
The Pool.sol contract is the main user facing contract of the protocol. It exposes the liquidity management methods.

The Pool.sol is owned by the PoolAddressedProvider of the specific market.

All admin functions are callable by the PoolConfigurator contract, which is defined in the PoolAddressesProvider.

![Pool.sol Contract Flow Diagram](./images/Pool_Flow.webp)

It is worth mentioning here that in the Pool.sol contract itself, it will mainly be calling to either internal functions such as:
- executeSupply() which is inside the SupplyLogic.sol or
- libraries like DataTypes which holds and stores the function's main parameters in structs.

### funtion supply()
========================================================== <br />
The ``supply()`` function supplies an amount of underlying asset into the reserve, receiving in return overlying aTokens. These aTokens serves as a receipt that a user supplied assets to the pool.

Eg: User supplies 100 USDC and gets in return 100 aUSDC
```sh
function supply(
  address asset, 
  uint256 amount, 
  address onBehalfOf, 
  uint16 referralCode
) external;
```
- asset: address of the asset being supplied to the pool
- amount: amount of asset being supplied to the pool
- onBehalfOf: address that will receive the corresponding aTokens. Note: Only the onBehalfOf address will be able to withdraw asset from the pool
- referralCode: unique code for 3rd party referral program integration. Use 0 (zero) for no referral.

Inside the function we are going to find only the call to an internal function ``executeSupply()`` which is inside ``SupplyLogic.sol``:

```sh
SupplyLogic.executeSupply(
  _reserves,
  _reservesList,
  _usersConfig[onBehalfOf],
  DataTypes.ExecuteSupplyParams({
    asset: asset,
    amount: amount,
    onBehalfOf: onBehalfOf,
    referralCode: referralCode
  })
);
```

You will notice ``DataTypes.ExecuteSupplyParams()`` call inside the above function's parameter. It is simply passing a list of parameters wrapped in a struct.

Now let's look into the ``executeSupply()`` function found in the ``SupplyLogic.sol`` contract. 

We can split it in three parts:

- The updates and validation:
```sh
reserve.updateState(reserveCache);

ValidationLogic.validateSupply(reserveCache, reserve, params.amount);

reserve.updateInterestRates(reserveCache, params.asset, params.amount, 0);
```

- From the code above, the first function ``resereve.updateState(reserveCache)`` will update the reserves with the provided reserveData from the specific asset passed as argument.

- Right after it, it will pass this data to validation where the main checks will be this asset fulfils the following:

```sh
require(isActive, Errors.RESERVE_INACTIVE);
require(!isPaused, Errors.RESERVE_PAUSED);
require(!isFrozen, Errors.RESERVE_FROZEN);
```

The above checks is inside the ``validateSupply()`` function found in ``the ValidationLogic.sol``

- The last thing is to update the interest rates with the ``updateInterestRates()`` function, which receives as parameters, the reserves cached, the specific asset and the amount of the asset.

- The main action of the function is where the supply of the ERC20 token is done, by using the ``safeTransferFrom()`` function and the ``mint()`` of aTokens. See the code below:
```sh
IERC20(params.asset).safeTransferFrom(msg.sender, reserveCache.aTokenAddress, params.amount);
```

```sh
IAToken(reserveCache.aTokenAddress).mint(msg.sender, params.onBehalfOf, params.amount, reserveCache.nextLiquidityIndex);
```

- Setting the Collateral Value
This will only happen if this is the first time the sender is making a supply. That is why in the code you will see a condition ``isFirstSupply`` The validation before modifying anything and finally the setter.





### function withdraw()
========================================================== <br />
The withdraw function withdraws an amount of underlying asset from the reserve, burning the equivalent aTokens owned

Eg: User has 100 aUSDC, calls the ``withdraw()`` function and receives 100USDC, burning the 100 aUSDC.

This is the description provided in the IPool interface codebase and it is an excellent summary of what happens inside this function.

```sh
function withdraw(
  address asset, 
  uint256 amount, 
  address to
) external returns (uint256);
```

Just like the ``supply()`` function, ``withdraw()`` function is also directly calling to the internal function ``executeWithdraw()`` which is inside the ``SupplyLogic.sol``

```sh
SupplyLogic.executeWithdraw(
  _reserves,
  _reservesList,
  _eModeCategories,
  _usersConfig[msg.sender],
  DataTypes.ExecuteWithdrawParams({
    asset: asset,
    amount: amount,
    to: to,
    reservesCount: _reservesCount,
    oracle: ADDRESSES_PROVIDER.getPriceOracle(),
    userEModeCategory: _usersEModeCategory[msg.sender]
  })
);
```
Let's divide what is happening inside the ``executeWithdraw()`` function into two parts:
- Update and validate

```sh
ValidationLogic.validateWithdraw(reserveCache, amountToWithdraw, userBalance);

reserve.updateInterestRates(reserveCache, params.asset, 0, amountToWithdraw);
```

Just like in the ``executeSupply()`` it validates the provided parameters in order to update interest rates.

Then it will check one of the parameters provided which is ``isUsingAsCollateral()``. If this returns true and the amount desired to withdraw is as hight as the ``userBalance``, then it will cancel the existung collateral.

```sh
if (isCollateral && amountToWithdraw == userBalance) {
  userConfig.setUsingAsCollateral(reserve.id, false);
}
```

- Burn the aToken:
Burns aTokens from the user and sends the equivalent amount of underlying token to the address specified in the ``params.to`` parameter.

```sh
IAToken(reserveCache.aTokenAddress).burn(msg.sender, params.to, amountToWithdraw, reserveCache.nextLiquidityIndex);
```




### function borrow()
========================================================== <br />
The `borrow()` function allows users to borrow a specific amount of the reserve underlying asset, provide that the borrower already supplied enough collateral, or he was given enough allowance by a credit delegator on the corresponding debt token (StableDebtToken or VariableDebtToken)

Eg: User borrows 100USDC passing as `onBehalfOf` his own address, receiving the 100 USDC in his wallet and 100 stable/variable debt tokens, depending on the `interestRateMode`

```sh
function borrow(
    address asset,
    uint256 amount,
    uint256 interestRateMode,
    uint16 referralCode,
    address onBehalfOf
) external;
```
Inside the `borrow()` function and like in the previous functions, it will directly call an internal function `executeBorrow()` which is inside the `BorrowLogic.sol` see code below:

```sh
BorrowLogic.executeBorrow(
    _reserves,
    _reservesList,
    _eModeCategories,
    _usersConfig[onBehalfOf],
    DataTypes.ExecuteBorrowParams({
        asset: asset,
        user: msg.sender,
        onBehalfOf: onBehalfOf,
        amount: amount,
        interestRateMode: DataTypes.InterestRateMode(interestRateMode),
        referralCode: referralCode,
        releaseUnderlying: true,
        maxStableRateBorrowSizePercent: _maxStableRateBorrowSizePercent,
        reservesCount: _reservesCount,
        oracle: ADDRESSES_PROVIDER.getPriceOracle(),
        userEModeCategory: _usersEModeCategory[onBehalfOf],
        priceOracleSentinel: ADDRESSES_PROVIDER.getPriceOracleSentinel()
    })
);
```

One thing to highlight from list of parameters passed in the code above is the `interestRateMode`. It is basically an ENUM. It determines which debt token will be minted.

```sh
enum InterestRateMode {
  NONE,
  STABLE,
  VARIABLE
}
```

If the `interestRateMode` is `STABLE`, it will execute:
```
IStableDebtToken(reserveCache.stableDebtTokenAddress).mint(params.user, params.onBehalfOf, params.amount, currentStableRate);
```
And if the `interestRateMode` is `VARIABLE` it will execute:
```
IVariableDebtToken(reserveCache.variableDebtTokenAddress).mint(params.user, params.onBehalfOf, params.amount, reserveCache.nextVariableBorrowIndex);
```

Once the tokens are minted, what's left is the actual transfer of the asset/underlying to the user:
```sh
IAToken(reserveCache.aTokenAddress).transferUnderlyingTo(params.user, params.amount);
```





### function repay()
========================================================== <br />
The ``repay()`` function is called to repay a borrowed amount on a specific reserve, burning the equivalent debt tokens owned.

Eg: User repays 100USDC, burning 100 variable/stable debt tokens of the ``onBehalfOf`` address

```sh
function repay(
  address asset,
  uint256 amount,
  uint256 interestRateMode,
  address onBehalfOf
) external returns (uint256);
```

Just like in other functions, `repay()` function is also calling an internal function `executeRepay()` inside the `BorrowLogic.sol`:

```sh
BorrowLogic.executeRepay(
  _reserves,
  _reservesList,
  _usersConfig[onBehalfOf],
  DataTypes.ExecuteRepayParams({
    asset: asset,
    amount: amount,
    interestRateMode: DataTypes.InterestRateMode(interestRateMode),
    onBehalfOf: onBehalfOf,
    useATokens: false
  })
);
```

Notice from the parameters. `useATokens` which in this case is directly set to `false`. This is because there is another methode which allows user to repay with aTokens without leaving dust from interest and it's called `repayWithAtokens()`.

This means that when users want to repay their debt, it will be by either: 

- Burning their stable/variable debt token.

```sh
IStableDebtToken(reserveCache.stableDebtTokenAddress).burn(params.onBehalfOf, paybackAmount);
```

```sh
IVariableDebtToken(reserveCache.variableDebtTokenAddress).burn(params.onBehalfOf, paybackAmount, reserveCache.nextVariableBorrowIndex);
```

Or by:
- Burning the aTokens received when providing liquidity through `Supply()` function

```sh
IAToken(reserveCache.aTokenAddress).burn(msg.sender, reserveCache.aTokenAddress, paybackAmount, reserveCache.nextLiquidityIndex);
```

The difference between above two methods of repaying is that:

In the case of using aTokens to repay, the transaction is finished (there is actually lack of any transfer) because as you may remember, we mentioned that if the user wants to withdraw the asset supplied, the equivalent aTokens will be returned. Which means if a user repays with aTokens, they won't be able to withdraw the asset supplied since they don't have anymore equivalent aTokens to return.

Otherwise, there will still be the need to execute the
`IERC20().safeTransferFrom()` to the specified amount from `msg.sender`

```sh
IERC20(params.asset).safeTransferFrom(msg.sender, reserveCache.aTokenAddress, paybackAmount);

IAToken(reserveCache.aTokenAddress).handleRepayment(msg.sender, params.onBehalfOf, paybackAmount);
```