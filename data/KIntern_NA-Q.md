# [2024-02-tapioca] QA report 
###### tags: `c4`, `2024-02-tapioca`, `QA`

##  Low Risk Issues

| Number | Issue |
|--------|--------|
|[L-01]| Missing a safe check to ensure the correct asset is being withdrawn in the `depositAddCollateralAndBorrowFromMarket` function |
|[L-02]| Should use lower mixedCase for a variable instead of UPPER_CASE_WITH_UNDERSCORES   |
|[L-03]| Redundant check `if (_timestampToWeek(block.timestamp) > currentWeek()) revert AdvanceEpochFirst();` |
|[L-04]| Incorrect parameter used in error message `NotApproved` in function `_requireClaimPermission()` |
|[L-05]| Redundant function `_existInArray()` in contract twTAP.sol   |
|[L-06]| Different `MIN_WEIGHT_FACTOR` was used in the contract `twTAP`   |
|[L-07]| Redundant event `UnregisterSingularity` emission    |
|[L-08]| Redundant check `if (lock.lockDuration < EPOCH_DURATION) revert DurationTooShort();`   |
|[L-09]| Incorrect description comment in function `decodeArrayOfYieldBoxPermitAssetMsg()`   |
|[L-10]| Users are able to buy TAPs without paying any paymentToken   |

### [L-01] Missing a safe check to ensure the correct asset is being withdrawn in the `depositAddCollateralAndBorrowFromMarket` function
The `depositAddCollateralAndBorrowFromMarket` function doesn't check to ensure that `data.withdrawParams.assetId` is the same as the asset ID of `market_`. Therefore, the received token from borrowing and the withdrawn token later may be different.
```solidity=
if (data.borrowAmount > 0) {
    address borrowReceiver = data.withdrawParams.withdraw ? address(this) : data.user;

    (Module[] memory modules, bytes[] memory calls) =
        IMarketHelper(data.marketHelper).borrow(data.user, borrowReceiver, data.borrowAmount);
    market_.execute(modules, calls, true);

    if (data.withdrawParams.withdraw) {
        _withdrawToChain(data.withdrawParams);
    }
}
```
https://github.com/Tapioca-DAO/tapioca-periph/blob/032396f701be935b04a7e5cf3cb40a0136259dbc/contracts/Magnetar/modules/MagnetarCollateralModule.sol#L106



### [L-02] Should use lower mixedCase for a variable instead of UPPER_CASE_WITH_UNDERSCORES 
Variable `VIRTUAL_TOTAL_AMOUNT` and `MIN_WEIGHT_FACTOR` in contract `twTAP` is not a constant variable but it uses the UPPER_CASE_WITH_UNDERSCORES for their name which is doesn't follow the solidity convention 
https://github.com/Tapioca-DAO/tap-token/blob/20a83b1d2d5577653610a6c3879dff9df4968345/contracts/governance/twTAP.sol#L80-L82



### [L-03] Redundant check `if (_timestampToWeek(block.timestamp) > currentWeek()) revert AdvanceEpochFirst();`
In contract `twTAP`, the function `_timestampToWeek()` is implemented as follows: 
```solidity=
function _timestampToWeek(uint256 timestamp) internal view returns (uint256) {
    return ((timestamp - creation) / EPOCH_DURATION);
}
```

Hence if the `timestamp` equal to the `block.timestamp`, we have the value of `_timestampToWeek(block.timestamp)` equal to the value of `currentWeek()`.

```solidity=
function currentWeek() public view returns (uint256) {
    return (block.timestamp - creation) / EPOCH_DURATION;
}
```

So the check `if (_timestampToWeek(block.timestamp) > currentWeek()) revert AdvanceEpochFirst();` in line 301 of contract twTAP is redundant. 
* https://github.com/Tapioca-DAO/tap-token/blob/20a83b1d2d5577653610a6c3879dff9df4968345/contracts/governance/twTAP.sol#L301


### [L-04] Incorrect parameter used in error message `NotApproved` in function `_requireClaimPermission()`

https://github.com/Tapioca-DAO/tap-token/blob/20a83b1d2d5577653610a6c3879dff9df4968345/contracts/governance/twTAP.sol#L549
Function `twTAP._requireClaimPermission()` is used to check if the address `_to` is approved by the `_tokenId`'s owner. It will revert the transaction if the `_to` address isn't approved. However if this case happens, the error is emitted using the `spender` with the value `msg.sender` instead of `_to`.


### [L-05] Redundant function `_existInArray()` in contract twTAP.sol 

Function `_existInArray()` is implemented but isn't used anywhere in the protocol 
https://github.com/Tapioca-DAO/tap-token/blob/20a83b1d2d5577653610a6c3879dff9df4968345/contracts/governance/twTAP.sol#L626-L636


### [L-06] Different `MIN_WEIGHT_FACTOR` was used in the contract `twTAP`

According to the [Tapioca docs](https://docs.tapioca.xyz/tapioca/core-technologies/twaml#implementation-details)
> User's locks which account for less than 0.1% of total platform liquidity do not effect the AML. This variable is modifiable by governance instruction.

the `MIN_WEIGHT_FACTOR` should be 0.1% which is the value 10 but in the current implementation, the default value of the `MIN_WEIGHT_FACTOR` is 1000 which is 10% 

https://github.com/Tapioca-DAO/tap-token/blob/20a83b1d2d5577653610a6c3879dff9df4968345/contracts/governance/twTAP.sol#L82

### [L-07] Redundant event `UnregisterSingularity` emission 

Function `TapiocaOptionLiquidityProvision.unregisterSingularity()` is used to unregister a singularity. This function loops through all the singularities in the contract, and check if which element is matched with the requested singularity. When the element was found, the event `UnregisterSingularity` will be emitted within the for-loop. 

```solidity=
function unregisterSingularity(IERC20 singularity) external onlyOwner updateTotalSGLPoolWeights {
    ...
    
    for (uint256 i; i < sglLength; i++) {
        if (_singularities[i] == sglAssetID) {
            ...
            emit UnregisterSingularity(address(singularity), sglAssetID);
            break;
        }
        emit UnregisterSingularity(address(singularity), sglAssetID);
    }
}
```

However after the loop is "break", the function still emit the event `UnregisterSingularity` again which is redundant. 

https://github.com/Tapioca-DAO/tap-token/blob/20a83b1d2d5577653610a6c3879dff9df4968345/contracts/options/TapiocaOptionLiquidityProvision.sol#L360

### [L-08] Redundant check `if (lock.lockDuration < EPOCH_DURATION) revert DurationTooShort();`

Function `TapiocaOptionBroker.participate()` is used to lock the tOLP position to the contract. In the contract there is a check to make sure the tOLP position's lock duration is bigger than 1 week 

https://github.com/Tapioca-DAO/tap-token/blob/20a83b1d2d5577653610a6c3879dff9df4968345/contracts/options/TapiocaOptionBroker.sol#L239
```solidity=
function participate(uint256 _tOLPTokenID) external whenNotPaused nonReentrant returns (uint256 oTAPTokenID) {
    ... 
    
    if (lock.lockDuration < EPOCH_DURATION) revert DurationTooShort();
    
    ...
}
```

However this check is already happend when the tOLP position is minted in function `TapiocaOptionLiquidityProvision.lock()` 

https://github.com/Tapioca-DAO/tap-token/blob/20a83b1d2d5577653610a6c3879dff9df4968345/contracts/options/TapiocaOptionLiquidityProvision.sol#L192
```solidity=
function lock(address _to, IERC20 _singularity, uint128 _lockDuration, uint128 _ybShares)
    external
    nonReentrant
    returns (uint256 tokenId)
{
    if (_lockDuration < EPOCH_DURATION) revert DurationTooShort();
    
    ...
}
```

Hence the check in the function `TapiocaOptionBroker.participate()` is redundant 



### [L-09] Incorrect description comment in function `decodeArrayOfYieldBoxPermitAssetMsg()`

In function `decodeArrayOfYieldBoxPermitAssetMsg()`, the `_msg`'s length should be 190 instead of 189. However in the comment about the function, it said 

https://github.com/Tapioca-DAO/tapioca-periph/blob/032396f701be935b04a7e5cf3cb40a0136259dbc/contracts/tapiocaOmnichainEngine/TapiocaOmnichainEngineCodec.sol#L330
```
* @dev The message length must be a multiple of 189.
```

which is incorrect 

### [L-10] Users are able to buy TAPs without paying any paymentToken

The function `TapiocaOptionBroker._getDiscountedPaymentAmount()` is used to compute the payment amount for a given OTC in USD. This function first calculates the `rawPaymentAmount` by dividing the `_otcAmountInUSD` by `_paymentTokenValuation`. Then, it deducts the discount value to determine the amount of payment tokens needed to complete the trade.

https://github.com/Tapioca-DAO/tap-token/blob/20a83b1d2d5577653610a6c3879dff9df4968345/contracts/options/TapiocaOptionBroker.sol#L541-L571

```solidity=
uint256 rawPaymentAmount = _otcAmountInUSD / _paymentTokenValuation;
paymentAmount =
    rawPaymentAmount -
    muldiv(rawPaymentAmount, _discount, 100e4); // 1e4 is discount decimals, 100 is discount percentage
```

The issue here is that the `rawPaymentAmount` is determined using round-down calculation, which can be manipulated by an attacker. The attacker can exploit this by selecting a paymentToken with a high value and each time they call `exerciseOption()`, they can choose an appropriate amount of TAP tokens to exercise such that `tapAmount * epochTAPValuation < _paymentTokenValuation`. By doing this, the attacker can effectively pay nothing to receive the TAP tokens.