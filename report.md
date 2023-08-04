<table>
    <tr><th></th><th></th></tr>
    <tr>
        <td><img src="[https://media-exp1.licdn.com/dms/image/C4E0BAQGsUEaZCJtT-w/company-logo_200_200/0/1625861473301?e=2147483647&v=beta&t=y8nFNaFXa7nkrgNV8-8giEb0m5s9cYOEwHSMJCC4a3s](https://www.grappa.finance/static/media/logo.a1e152c4e188ac3f75ad.png)" width="250" height="250" /></td>
        <td>
            <h1>Grappa Finance - Full Collateral Engine report</h1>
            <h2>Minting options engine</h2>
            <h3>Status: not deployed</h3>
            <p>Author of Smart Contracts: @antoncoding</p>
            <p>Prepared by: Delvir0, Independent Security Researcher</p>
            <p>Date of completion: 08-03-2023</p>
        </td>
    </tr>
</table>

# About **Grappa Finance - Full Collateral Engine**

Grappa Finance in it's core is an exchange and settlement platform for cash-settled options and physical assets. It's smart contracts enable sellers and buyers a single point of minting and settlement of options.
Handling of these options (for sellers) is done via Engines which connect with the Grappa Finance system. 
The Full Collateral Engine is one of those engine which ensures that sellers deposit up to the full collateral in order to mint an option which then can be sold. 

# Summary & Scope

[`grappafinance/full-collat-engine/src`](https://github.com/grappafinance/full-collat-engine/tree/42d52af7d802521f88ca0cb89e36afcffda1182b/src), including contracts inherited, under commit [42d52](https://github.com/grappafinance/full-collat-engine/tree/42d52af7d802521f88ca0cb89e36afcffda1182b)

The following contracts were in scope:
- [BaseEngine.sol@8e67b0b](https://github.com/grappafinance/core-cash/blob/8e67b0bedf53679c18de76ae70345d829737df56/src/core/engines/BaseEngine.sol)
- [DebitSpread.sol@8e67b0b](https://github.com/grappafinance/core-cash/blob/8e67b0bedf53679c18de76ae70345d829737df56/src/core/engines/mixins/DebitSpread.sol)

- [FullMarginEngine.sol@42d52af](https://github.com/grappafinance/full-collat-engine/blob/42d52af7d802521f88ca0cb89e36afcffda1182b/src/FullMarginEngine.sol)
- [FullMarginLib.sol@42d52af](https://github.com/grappafinance/full-collat-engine/blob/42d52af7d802521f88ca0cb89e36afcffda1182b/src/FullMarginLib.sol)
- [FullMarginMath.sol@42d52af](https://github.com/grappafinance/full-collat-engine/blob/42d52af7d802521f88ca0cb89e36afcffda1182b/src/FullMarginMath.sol)
- [errors.sol@42d52af](https://github.com/grappafinance/full-collat-engine/blob/42d52af7d802521f88ca0cb89e36afcffda1182b/src/errors.sol)
- [types.sol@42d52af](https://github.com/grappafinance/full-collat-engine/blob/42d52af7d802521f88ca0cb89e36afcffda1182b/src/types.sol)

# Summary of Findings

|  Identifier  | Title                        | Severity      | Fixed |
| ------ | ---------------------------- | ------------- | ----- |
| [C-01] | Minter can mint options with incorrect collateral asset, leading the protocol to be drained | Critical |  |
| [M-02] | Other entities interacting with the mint function can be DOSSed | Medium |  |
| [M-03] | Changing amount of account access is open to a sandwich attack | Medium |  |
| [L-04] | Assumption on resetting collateral will fail in cases where tokens do not support 0 transfers | Low |  |
| [I-05] | Naming convention of longStrike and shortStrike is subjected to perspective | Informational |  |

# Detailed Findings

## [C-01] Minter can mint options with incorrect collateral asset, leading the protocol to be drained
### FullMarginEngine.sol

The Engine has a single point of entry for all the possible actions provided:
```solidity
function execute(address _subAccount, ActionArgs[] calldata actions) public override nonReentrant {
        _assertCallerHasAccess(_subAccount);
        // update the account state and do external calls on the flight
        for (uint256 i; i < actions.length;) { 
            if (actions[i].action == ActionType.AddCollateral) _addCollateral(_subAccount, actions[i].data);
            else if (actions[i].action == ActionType.RemoveCollateral) _removeCollateral(_subAccount, actions[i].data);
            else if (actions[i].action == ActionType.MintShort) _mintOption(_subAccount, actions[i].data);
            else if (actions[i].action == ActionType.BurnShort) _burnOption(_subAccount, actions[i].data);
            else if (actions[i].action == ActionType.MergeOptionToken) _merge(_subAccount, actions[i].data);
            else if (actions[i].action == ActionType.SplitOptionToken) _split(_subAccount, actions[i].data);
            else if (actions[i].action == ActionType.SettleAccount) _settle(_subAccount);
            else revert FM_UnsupportedAction();

            unchecked {
                ++i;
            }
        }
        if (!_isAccountAboveWater(_subAccount)) revert BM_AccountUnderwater();
```
When minting or creating an option, each type (CALL, PUT, CALL_SPREAD, PUT_SPREAD) has each own requirement of underlying collateral. This could be either tokens like WETH, WBTC, ARB, UNI or stables.
The collateral is used in order to pay the option holder (which has been sold to by the minter) if the option is worth anything at time of expiry.
Collateral health is checked at the end of the `execute()` function: `if (!_isAccountAboveWater(_subAccount)) revert BM_AccountUnderwater();` which eventually leads to:

```solidity
function getCollateralRequirement(FullMarginDetail memory _account) internal pure returns (uint256 minCollat) {
        // don't need collateral
        if (_account.shortAmount == 0) return 0;

        // amount with UNIT decimals
        uint256 unitAmount;

        if (_account.tokenType == TokenType.CALL) {
            // call option must be collateralized with underlying
            unitAmount = _account.shortAmount;
        } else if (_account.tokenType == TokenType.CALL_SPREAD) { 
            // if long strike <= short strike, all loss is covered, amount = 0
            // only consider when long strike > short strike 
            if (_account.longStrike > _account.shortStrike) { 
            //note FullMarginEngine._getAccountPayout:  if (tokenType == TokenType.CALL_SPREAD && shortStrike > longStrike) 
                // only call spread can be collateralized by both strike or underlying
                if (_account.collateralizedWithStrike) { 
                    // ex: 2000-4000 call spread with usdc collateral
                    // return (longStrike - shortStrike) * amount / unit

                    unchecked {
                        unitAmount = (_account.longStrike - _account.shortStrike);
                    }
                    unitAmount = unitAmount * _account.shortAmount;
                    unchecked {
                        unitAmount = unitAmount / UNIT;
                    }
                } else {
                    // ex: 2000-4000 call spread with eth collateral
                    unchecked {
                        unitAmount =
                            (_account.longStrike - _account.shortStrike).mulDivUp(_account.shortAmount, _account.longStrike);
                    }
                }
            }
        } else if (_account.tokenType == TokenType.PUT) {
            // put option must be collateralized with strike (USDC) asset
            // unitAmount = shortStrike * amount / UNIT
            unitAmount = _account.shortStrike * _account.shortAmount; 
            unchecked {
                unitAmount = unitAmount / UNIT;
            }
        } else if (_account.tokenType == TokenType.PUT_SPREAD) {
            // put spread must be collateralized with strike (USDC) asset
            // if long strike >= short strike, all loss is covered, amount = 0
            // only consider when long strike < short strike
            if (_account.longStrike < _account.shortStrike) {
                // unitAmount = (shortStrike - longStrike) * amount / UNIT

                unchecked {
                    unitAmount = (_account.shortStrike - _account.longStrike);
                }
                unitAmount = unitAmount * _account.shortAmount;
                unchecked {
                    unitAmount = unitAmount / UNIT;
                }
            }
        } else {
            revert FM_UnsupportedTokenType();
        }

        return NumberUtil.convertDecimals(unitAmount, UNIT_DECIMALS, _account.collateralDecimals); 
    }
```
The danger of single point of entries where we're allowed any combination of actions and where health is only checked at the end is it provides re-entrancy like vulnerabilities.
An CALL options requires the underlying collateral to be the same asset representing the option. This is due to the fact that the price could increase Infinitely. The opposite is true when it's a PUT option, the max loss would be the price of the asset. Since the asset could drop to 0 and leave it worthless, the underlying needs to be (underlyingPrice * 1USDC).


collateralId and amount is stored in the following struct:
```solidity
struct FullMarginAccount {
    uint256 tokenId;
    uint64 shortAmount;
    uint8 collateralId;
    uint80 collateralAmount;
}
```
The accounting of adding and removing collateral is done in the following way: 
```solidity
    function addCollateral(FullMarginAccount storage account, uint8 collateralId, uint80 amount) internal {
        uint80 cacheId = account.collateralId;
        if (cacheId == 0) {
            account.collateralId = collateralId;
        } else {
            if (cacheId != collateralId) revert FM_WrongCollateralId();
        }
        account.collateralAmount += amount;
    }


    function removeCollateral(FullMarginAccount storage account, uint8 collateralId, uint80 amount) internal {
        if (account.collateralId != collateralId) revert FM_WrongCollateralId();

        uint80 newAmount = account.collateralAmount - amount;
        account.collateralAmount = newAmount;
        if (newAmount == 0) {
            account.collateralId = 0;
        }
    }
```
When settlement of the option token(s) is performed, the profit of the option holder is deducted from `FullMarginAccount.collateralAmount` of the seller (minter).

The problem is that there is no check in place that links the required collateral type of an option token that has been minted with the current collateral of the seller.
Above would be a problem if we would reach a scenario where the seller has an option which requires collateralA of high value per unit while depositing collateralB of low value per unit.

Here is one of the flows of how this can be achieved:
- A seller uses the `execute()` function to perform 4 actions (given ETH == 2000USDC):
    - Add USDC as collateral (1e6)
    - Mint a PUT option (with very high strike price like 1 million)
    - Remove USDC collateral
    - Add 1e-6 ETH as collateral. Collateral required calculation will be calculated in USDC term: 1 million * 1e6 = 1e12
    (this passes the collateral check. The option now enables to withdraw "1 million - 2000" USDC from the protocol)

The damage occures when the option holder's option is worth x amount of ETH and calls the settlement function. 
This will trigger the Engine to send a x amount of ETH from the contract to the holder and deduct that from `FullMarginAccount.collateralAmount`.
This essentially leaves the contract in a debt position (collateralETH of other sellers is send to the holder while deducting the amount from collateralUSDC of the seller).
Mainly, making this issue a C-level, the seller could also act as a holder, essentially trading a fraction of ETH for USDC, which leaves a way to drain the contract. 

### Recommendation
There are different ways to prevent this. I would suggest to add a check in `_isAccountAboveWater()` where it is checked if `optionCollateralRequirement == account.collateralId` which is also the most gass efficient way.

## [M-02] Other entities interacting with the mint function can be DOSSed
### FullMarginEngine.sol

Each account can have up to 255 sub accounts, where sub-account(0) is the main account, in order to be able to handle different option tokens with each their collateral requirement (inspired by Euler's sub-Accounts).
The property's of these are held in the struct `FullMarginAccount` and can be moved to any (sub)account:

```solidity
     * @notice transfer an account to someone else
     * @dev expected to be call by account owner
     * @param _subAccount the id of subaccount to transfer
     * @param _newSubAccount the id of receiving account
     */
    function transferAccount(address _subAccount, address _newSubAccount) external { 
        if (!_isPrimaryAccountFor(msg.sender, _subAccount)) revert FM_NoAccess(); 

        if (!marginAccounts[_newSubAccount].isEmpty()) revert FM_AccountIsNotEmpty();
        marginAccounts[_newSubAccount] = marginAccounts[_subAccount];

        delete marginAccounts[_subAccount];
    }
```
At point of minting an option token, it is required to either have `FullMarginAccount.collateralId == 0` or to the correct underlying collateralId. If both are incorrect, the transactions reverts.
Since it's possible to send the `FullMarginAccount` to anyone and it's possible to calculate the 255 addresses of each account, we could prevent any entity from minting by transfering an account with the incorrect collateralId.

Here is a simple flow of how this can be achieved:
- Attacker monitors mempool and sees mint (CALL) transactions for sub-account(1) of a competing seller 
- Attacker sees this and transfers an account with `collateralId == USDC` to that sub account
- Since an CALL option requires e.g. ETH, the transaction will fail

The result is a possebility to damage an entity by making their service fail (mint and sell options) during crucial times. 

### Recommendation
I would like to recommend rethinking the need for the transfer of sub-accounts.
Incase it's needed you could work with whitelists where a user is not able to send the details of a sub-account to sub-accounts of an other head-account unless it's whitelisted to do so.

## [M-03] Changing amount of account access is open to a sandwich attack
### BaseEngine.sol

The BaseEngine contract has the option to allow any other entity to perform actions on behave. This is recorded with an amount of actions which is inputted:
```solidity
    function setAccountAccess(
        address _account,
        uint256 _allowedExecutions
    ) external {
        uint160 maskedId = uint160(msg.sender) | 0xFF;
        allowedExecutionLeft[maskedId][_account] = _allowedExecutions;

        emit AccountAuthorizationUpdate(maskedId, _account, _allowedExecutions);
    }
```
This leads to the typical allowance vulnerability where allowance is set to x amount and increased to a new amount.

Simple example where this could lead to an issue: 
- UserA has set UserB to 50 allowedExecutions
- UserA decides to add another 50 allowedExecutions and calls `setAccountAccess( ,100)`
- UserB now had the opportunity to perform 50 allowedExecutions before above call gets mined and another 100 after. This results in a total allowedExecutions of 150.

### Recommendation
Reconstruct the function to an increase/ decrease flow instead of setting a x amount.
This can be achieved in the same as the decreaseAllowance/ increaseAllowance flow of OZ:
https://docs.openzeppelin.com/contracts/2.x/api/token/erc20#ERC20-decreaseAllowance-address-uint256-


## [L-04] Assumption on resetting collateral will fail in cases where tokens do not support 0 transfers
### FullMarginLib.sol

At settlement, theres a possebility of the amount of debt being equal to collateralAmount.
If this happens, it is assumed that `account.collateralId` can be changed to an other id in order to be able to mint other options by calling `removeCollateral(0)`

The problem is that calling `removeCollateral(0)` triggers a 0 amount transfer while not all tokens support that.

```solidity
    function settleAtExpiry(FullMarginAccount storage account, int80 payout) internal { 
        // clear all debt
        account.tokenId = 0;
        account.shortAmount = 0; 

        int256 collateral = int256(uint256(account.collateralAmount));

        // this line should not underflow because collateral should always be enough
        // but keeping the underflow check to make sure
        account.collateralAmount = (collateral - payout).toUint256().toUint80();

        // do not check ending collateral amount (and reset collateral id) because it is very
        // unlikely the payout is the exact amount in the account
        // if that is the case (collateralAmount = 0), use can use removeCollateral(0) // @audit-issue incorrect since some tokens do not support 0 transfers
        // to reset the collateral id 
    }
```
In this case the specific account would be stuck to the current `collateralId`.

### Recommendation
Given the scenario, the following are to have in mind when considering a fix  1. low probablilty of above happening 2. there are 255 remaining accounts to be used.
This can be fixed by checking the remaining collateral and resetting the collateralId if it's 0

## [I-05] Naming convention of longStrike and shortStrike is subjected to perspective
### Multiple contracts

When dealing with options, the terms short and long strike are in it's core and is adapted in the contracts.
When a minter has an CALL option with a long strike at x and sells it, it would be considered an CALL option of **short strike** at x at the perspective of the minter (seller).

The contracts are written in such a way that `longStrike` and `shortStrike` are written in the perspective of the minter _and in the scenario of the option already sold_.
Having the logic of the strikes inverted causes a lot of confusion when reading through the logic of the codebase and does not align with Grappa.sol.

### Recommendation
Invert the current naming of the strikes so that they represent the values of what's actually being minted instead of how they would be seen in the perspective of the seller.
`longStrike -> shortStrike`
`shortStrike -> longStrike`
