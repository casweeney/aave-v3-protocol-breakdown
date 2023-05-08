### AAVE Code Explained Part 1
Aave is a decentralized non-co=ustodial liquidity market protocol where users can participate as suppliers or borrowers.

#### Suppliers & Borrowers
Suppliers provide liquidity to the market and earn a passive income, while borrowers are able to borrow unexpected expenses, leveraging their holding.

Liquidity is at the heart of the Aave Protocol as it enables the protocol's operation and user experience.

The liquidity of the protocol is measured by the availability of assets for basic protocol operations such as borrowing of assets backed by collateral and claiming of supplied assets along with accrued yield.
- A lack of liquidity will block operations.

### Contracts Overview
The Aave Porotocol V3 contracts are divided into two repositories
1. aave-v3-core
2. aave-v3-periphery

##### aave-v3-core
The aave-v3-core host or contains the core protocol V3 contracts that contains the logic for supply, borrow, liquidation, flashloans, a/s/v tokens, portal, pool configuration, oracles and interest rate strategies.

The core protocol contracts fall in the following 4 categories:
- Configuration
- Pool Logic
- Tokenization
- Misc

##### aave-v3-periphery
In the periphery repository, you will find the contracts related towards rewards, UI data provider, wallet balance provider and WETH gateway.

The periphery contracts have two categories
- Rewards
- Misc

### What can we do with Aave Protocol
- Supplu and Earn:
By supplying you will earn passive income based on the market borrowing demand

- Borrow:
Additionally, supplying assets allows you to borrow by using your supplied assets as a collateral


#### Pool.sol Smart Contract
The Pool.sol contract is the main user facing contract of the protocol. It exposes the liquidity management methods.

The Pool.sol is owned by the PoolAddressedProvider of the specific market.

All admin functions are callable by the PoolConfigurator contract, which is defined in the PoolAddressesProvider.

![Pool.sol Contract Flow Diagram](./images/Pool_Flow.webp)

It is worth mentioning here that in the Pool.sol contract itself, it will mainly be calling to either internal functions such as:
- executeSupply() which is inside the SupplyLogic.sol or
- libraries like DataTypes which holds and stores the function's main parameters in structs.