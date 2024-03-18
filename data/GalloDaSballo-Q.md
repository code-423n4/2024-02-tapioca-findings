# Preamble

The following are observations I made in reviewing the codebase

Some could have been sent as med, but overall they are best categorized as small gotchas that are best sent as QA.

# Recommendations for future steps

My main recommendation is to further shrink down the scope of the codebase to:
- SGL
- BB
- TapToken
- TwTAP
- TOB

I don't have strong confidence in any periphery contract being sage and believe the current `cluster` based setup has reached it's limit and is showing an increase in surface area.

Specifically, `isWhitelisted` being a Yes or No opens up to factorial complexity in possible combination of `whitelisted` called

A refactoring to:

```solidity
mapping(address caller => mapping(address receiver => mapping(bytes4 funSig => bool enabled)) isWhitelistedOnThisChain;
```

Will at least enforce all flows in a consistent way

That said, the additional surface area of these macro is in my opinion not fully explored.

Below are my QA findings

- [Preamble](#preamble)
- [Recommendations for future steps](#recommendations-for-future-steps)
- [twTAP](#twtap)
  - [QA / MED - You cannot pop rewardTokens, which may make claimers go OOG over time](#qa--med---you-cannot-pop-rewardtokens-which-may-make-claimers-go-oog-over-time)
  - [QA - You can get rewards for next epoch, by specifying duration of second + 1](#qa---you-can-get-rewards-for-next-epoch-by-specifying-duration-of-second--1)
  - [Year is 365.25 days](#year-is-36525-days)
  - [Unused code](#unused-code)
    - [Old comment (prob was uint b4)](#old-comment-prob-was-uint-b4)
    - [Incorrect data](#incorrect-data)
  - [PermitC allows u256 as input but breaks at u200](#permitc-allows-u256-as-input-but-breaks-at-u200)
  - [Permit C doesn't work with infinite allowance!](#permit-c-doesnt-work-with-infinite-allowance)
  - [Check is always true](#check-is-always-true)
    - [Mitigation](#mitigation)
- [TapiocaOptionBroker](#tapiocaoptionbroker)
  - [QA - `VIRTUAL_TOTAL_AMOUNT` can cause issues for below 18 decimal tokens](#qa---virtual_total_amount-can-cause-issues-for-below-18-decimal-tokens)
  - [QA - is there a MED? - You can get a bit more rewards](#qa---is-there-a-med---you-can-get-a-bit-more-rewards)
  - [M/QA - Cannot be used with stETH / sharelike tokens](#mqa---cannot-be-used-with-steth--sharelike-tokens)
  - [M / QA - tokens with hooks will cause issues](#m--qa---tokens-with-hooks-will-cause-issues)
- [LTap](#ltap)
  - [LTap is still using ERC20 Permit](#ltap-is-still-using-erc20-permit)
  - [Minting to LBP may not be ideal (you're prob minting to a multi I guess) | Or just mint to msg.sender](#minting-to-lbp-may-not-be-ideal-youre-prob-minting-to-a-multi-i-guess--or-just-mint-to-msgsender)
  - [Setting token multiple times is not ideal and is scary to end users](#setting-token-multiple-times-is-not-ideal-and-is-scary-to-end-users)
  - [You may want to open redemptions once the balance of tap matches the total supply as to prevent race conditions](#you-may-want-to-open-redemptions-once-the-balance-of-tap-matches-the-total-supply-as-to-prevent-race-conditions)
- [aoTAP](#aotap)
  - [QA: Forgot Pearlmit](#qa-forgot-pearlmit)
- [ModuleManager](#modulemanager)
  - [Memory expansion has already happened, check for length is ineffective against DOS](#memory-expansion-has-already-happened-check-for-length-is-ineffective-against-dos)
- [TapToken](#taptoken)
  - [QA - If everyone donates then the total supply will be lower than 100 MLN](#qa---if-everyone-donates-then-the-total-supply-will-be-lower-than-100-mln)
  - [QA - Pausable but all xChain stuff doesn't seem pausable](#qa---pausable-but-all-xchain-stuff-doesnt-seem-pausable)
- [QA - Storage is ok but could break](#qa---storage-is-ok-but-could-break)
  - [Silent fallback on the receiver means that TapToken has a ghost function that never reverts](#silent-fallback-on-the-receiver-means-that-taptoken-has-a-ghost-function-that-never-reverts)
- [TapTokenCodec](#taptokencodec)
  - [QA - Ambigous decoding for `decodeUnlockTwTapPositionMsg` allows passing multiple extra bytes while returning the same value](#qa---ambigous-decoding-for-decodeunlocktwtappositionmsg-allows-passing-multiple-extra-bytes-while-returning-the-same-value)
- [TapTokenReceiver](#taptokenreceiver)
  - [QA - Good old Safe Transfer](#qa---good-old-safe-transfer)
  - [`_internalTransferWithAllowance` can present issues for contracts that have different addresses crossChain](#_internaltransferwithallowance-can-present-issues-for-contracts-that-have-different-addresses-crosschain)
  - [QA - Overflow possible](#qa---overflow-possible)
- [Vesting](#vesting)
- [QA](#qa)
  - [Could pass duplicate users and lose those values](#could-pass-duplicate-users-and-lose-those-values)
  - [Not exact amt | Need SWEEP](#not-exact-amt--need-sweep)
  - [No longer needed](#no-longer-needed)
  - [BPS / Constants](#bps--constants)
- [Tapioca Periphery](#tapioca-periphery)
- [Magnetar](#magnetar)
  - [LOW / MED - Multiple ways to reenter Magnetar while having approvals](#low--med---multiple-ways-to-reenter-magnetar-while-having-approvals)
  - [`data.yieldBox` is not sanitized](#datayieldbox-is-not-sanitized)
  - [LOW / M - Magnetar allows to approve any approved target, this could be used to sweep if any contract that is whitelisted allows to have someone else pay on behalf](#low--m---magnetar-allows-to-approve-any-approved-target-this-could-be-used-to-sweep-if-any-contract-that-is-whitelisted-allows-to-have-someone-else-pay-on-behalf)
  - [Unwrap doesn't have a `from` field, but rather a `to` field](#unwrap-doesnt-have-a-from-field-but-rather-a-to-field)
    - [Mitigation](#mitigation-1)
  - [The function `_wrapSglReceipt` is declared twice](#the-function-_wrapsglreceipt-is-declared-twice)
- [Magnetar Helper](#magnetar-helper)
  - [QA - Risky Rounding down for total debt](#qa---risky-rounding-down-for-total-debt)
  - [Another round down](#another-round-down)
- [MagnetarAssetModule](#magnetarassetmodule)
  - [QA - Good old `safeApprove`](#qa---good-old-safeapprove)
  - [QA - Partial Deposit will require 2 calls](#qa---partial-deposit-will-require-2-calls)
- [MagnetarBaseModule](#magnetarbasemodule)
  - [Needlessly re-deploys a helper on each call](#needlessly-re-deploys-a-helper-on-each-call)
- [MagnetarOptionModule](#magnetaroptionmodule)
  - [QA - This is not amount, these are shares](#qa---this-is-not-amount-these-are-shares)
  - [This rounding could make repay 100% fail there](#this-rounding-could-make-repay-100-fail-there)
- [TapiocaOmnichainEngineHelper](#tapiocaomnichainenginehelper)
  - [QA: Not empty bytes](#qa-not-empty-bytes)
- [Extra](#extra)
  - [Pearlmit no longer supports infinite allowance](#pearlmit-no-longer-supports-infinite-allowance)
  - [Approve assets the following](#approve-assets-the-following)
  - [Run this test](#run-this-test)
  - [Pearlmit effectively caps all values to uint200](#pearlmit-effectively-caps-all-values-to-uint200)


# twTAP

## QA / MED - You cannot pop rewardTokens, which may make claimers go OOG over time

https://github.com/Tapioca-DAO/tap-token/blob/050e666142e61018dbfcba64d295f9c458c69700/contracts/governance/twTAP.sol#L513-L514

```solidity
        if (rewardTokens.length + 1 > maxRewardTokens) { /// @audit No POP

```

## QA - You can get rewards for next epoch, by specifying duration of second + 1

This means you get an extra epoch worth of rewards (can claim even after withdrawing)
While not needing to be locked

## Year is 365.25 days

https://github.com/Tapioca-DAO/tap-token/blob/050e666142e61018dbfcba64d295f9c458c69700/contracts/governance/twTAP.sol#L86-L87

```solidity
    uint256 public constant MAX_LOCK_DURATION = 100 * 365 days; // 100 years /// @audit QA: 365.25

```

## Unused code

https://github.com/Tapioca-DAO/tap-token/blob/050e666142e61018dbfcba64d295f9c458c69700/contracts/governance/twTAP.sol#L626-L627

```solidity
    function _existInArray(address _check, address[] memory _array) internal pure returns (bool) {

```


### Old comment (prob was uint b4)
https://github.com/Tapioca-DAO/tap-token/blob/050e666142e61018dbfcba64d295f9c458c69700/contracts/governance/twTAP.sol#L43-L44

```solidity
    bool divergenceForce; // 0 negative, 1 positive

```

### Incorrect data

Should be both `msg.sender` and `to`

https://github.com/Tapioca-DAO/tap-token/blob/050e666142e61018dbfcba64d295f9c458c69700/contracts/governance/twTAP.sol#L549

```solidity
 revert NotApproved(_tokenId, msg.sender); /// @audit This is incorrect
```

## PermitC allows u256 as input but breaks at u200

https://github.com/limitbreakinc/PermitC/blob/c431dc5e80690c1d8c3727f5992d519df3d38254/src/PermitC.sol#L1115-L1129

```solidity
    function _checkAndUpdateApproval(
        address owner,
        address token,
        uint256 id,
        uint256 amount,
        bool zeroOutApproval
    ) internal {
        PackedApproval storage approval = _getPackedApprovalPtr(owner, token, id, ZERO_BYTES32, msg.sender);
        
        if (approval.expiration < block.timestamp) {
            revert PermitC__ApprovalTransferPermitExpiredOrUnset();
        }
        if (approval.amount < amount) {
            revert PermitC__ApprovalTransferExceededPermittedAmount();
        }
```

Allows to use higher amount but will break at u200

## Permit C doesn't work with infinite allowance!

It seems like the developers didn't implement a mechanism for infinite allowance, see the POC I sent to them

https://gist.github.com/GalloDaSballo/1fc0f210830f4fb59700233037d83cb9


## Check is always true

https://github.com/Tapioca-DAO/tap-token/blob/050e666142e61018dbfcba64d295f9c458c69700/contracts/governance/twTAP.sol#L301-L302

```solidity
        if (_timestampToWeek(block.timestamp) > currentWeek()) revert AdvanceEpochFirst(); /// @audit QA for now

```


### Mitigation

https://github.com/Tapioca-DAO/tap-token/blob/050e666142e61018dbfcba64d295f9c458c69700/contracts/governance/twTAP.sol#L464-L465

```solidity
        if (lastProcessedWeek != currentWeek()) revert AdvanceWeekFirst();

```

# TapiocaOptionBroker

## QA - `VIRTUAL_TOTAL_AMOUNT` can cause issues for below 18 decimal tokens

However, TOB is meant to work with USDO so it will always have 18 decimals in the in-scope codebase


## QA - is there a MED? - You can get a bit more rewards

- You can be out of a lock and receive rewards in a bunch of ways
- But basically you get one epoch free out of every 2 epochs
- Cause you can lock -> Unlock at E + 1 -> Unlocked -> Relock at En - 1 -> Etc...

https://github.com/Tapioca-DAO/tap-token/blob/050e666142e61018dbfcba64d295f9c458c69700/contracts/options/TapiocaOptionBroker.sol#L317-L321

```solidity
        if (!isSGLInRescueMode) {
            if (block.timestamp < lock.lockTime + lock.lockDuration) {
                revert LockNotExpired();
            }
        }
```

## M/QA - Cannot be used with stETH / sharelike tokens

https://github.com/Tapioca-DAO/tap-token/blob/050e666142e61018dbfcba64d295f9c458c69700/contracts/options/TapiocaOptionBroker.sol#L560-L568

```solidity
 {
            bool isErr =
                pearlmit.transferFromERC20(msg.sender, address(this), address(_paymentToken), discountedPaymentAmount);
            if (isErr) revert TransferFailed();
        }
        uint256 balAfter = _paymentToken.balanceOf(address(this));
        if (balAfter - balBefore != discountedPaymentAmount) {
            revert TransferFailed();
        }
```

Due to possible rounding errors, valid transfers can fail when using stETH or similar share like tokens

e.g. blast ETH, stETH

This will not apply to stETH that is bridged as it won't have any accounting mechanism


## M / QA - tokens with hooks will cause issues

transferFromERC20

```solidity
 {
            bool isErr =
                pearlmit.transferFromERC20(msg.sender, address(this), address(_paymentToken), discountedPaymentAmount);
            if (isErr) revert TransferFailed();
        }
        uint256 balAfter = _paymentToken.balanceOf(address(this));
        if (balAfter - balBefore != discountedPaymentAmount) {
            revert TransferFailed();
        }
```

Can send dust amount and grief that



# LTap

## LTap is still using ERC20 Permit
## Minting to LBP may not be ideal (you're prob minting to a multi I guess) | Or just mint to msg.sender
## Setting token multiple times is not ideal and is scary to end users
## You may want to open redemptions once the balance of tap matches the total supply as to prevent race conditions

# aoTAP

## QA: Forgot Pearlmit

# ModuleManager

## Memory expansion has already happened, check for length is ineffective against DOS


https://github.com/Tapioca-DAO/tap-token/blob/050e666142e61018dbfcba64d295f9c458c69700/contracts/tokens/module/ModuleManager.sol#L73-L75

```solidity

    function _getRevertMsg(bytes memory _returnData) internal pure returns (string memory) {
        if (_returnData.length > 1000) return "Module: reason too long"; /// @audit QA: Still DOSSABLE
```



https://github.com/Tapioca-DAO/tap-token/blob/050e666142e61018dbfcba64d295f9c458c69700/contracts/tokens/module/ModuleManager.sol#L78-L79

```solidity
        if (_returnData.length < 68) return "Module: data"; /// @audit QA: This will not catch Custom Errors with 0 or 1 params

```



# TapToken

## QA - If everyone donates then the total supply will be lower than 100 MLN

I think

https://github.com/Tapioca-DAO/tap-token/blob/050e666142e61018dbfcba64d295f9c458c69700/contracts/tokens/TapToken.sol#L394-L395

```solidity
            dso_supply -= mintedInWeek[week - 1]; /// @audit Can Minted + `boostedTAP` mess up the math in the future?

```

## QA - Pausable but all xChain stuff doesn't seem pausable

If you're adding that, may as well allow to toggle those as well

# QA - Storage is ok but could break

is ok, see Gist
https://gist.github.com/GalloDaSballo/caaf22c136a716f010152199ed9167d8

https://github.com/Tapioca-DAO/tapioca-periph/blob/2ddbcb1cde03b548e13421b2dba66435d2ac8eb5/contracts/tapiocaOmnichainEngine/BaseTapiocaOmnichainEngine.sol#L32-L41

https://github.com/Tapioca-DAO/tapioca-periph/blob/2ddbcb1cde03b548e13421b2dba66435d2ac8eb5/contracts/tapiocaOmnichainEngine/BaseTapiocaOmnichainEngine.sol#L32-L41

```solidity
abstract contract BaseTapiocaOmnichainEngine is OFT, PearlmitHandler, BaseToeMsgType {
    using BytesLib for bytes;
    using SafeERC20 for IERC20;
    using OFTMsgCodec for bytes;
    using OFTMsgCodec for bytes32;

    /// @dev Used to execute certain extern calls from the TapToken contract, such as ERC20Permit approvals.
    TapiocaOmnichainExtExec public toeExtExec;
    /// @dev For future use, to extend the receive() operation.
    ITapiocaOmnichainReceiveExtender public tapiocaOmnichainReceiveExtender;
```


## Silent fallback on the receiver means that TapToken has a ghost function that never reverts

TapTokenReceiver silent failure could cause issues as they are unchecked, meaning they will not cause reverts

https://github.com/Tapioca-DAO/tap-token/blob/050e666142e61018dbfcba64d295f9c458c69700/contracts/tokens/TapToken.sol#L182-L186

```solidity
    fallback() external payable {
        /// @dev Call the receiver module on fallback, assume it's gonna be called by endpoint.
        _executeModule(uint8(ITapToken.Module.TapTokenReceiver), msg.data, false);
    } /// @audit This fails Silently | Even when it does call the module | It does fail if module doesn't exist

```


# TapTokenCodec

## QA - Ambigous decoding for `decodeUnlockTwTapPositionMsg` allows passing multiple extra bytes while returning the same value

This finding has no particular impact beside the risk of minting nonces / attempting to replay a signature with the same values

Due to a lack of checks in the decode function, it's possible to pass in ambigous values

```solidity
function test_malformedtokenUnlockTwTapPositionMsg_codec() public {
    malformedtokenUnlockTwTapPositionMsg(ab7686604127cbee3894e0b61bbdafcfa442ad6e228feccdc0dc2defaa68e57034b9aaffd9b4f660a3708517a479ce85501f48fb3e978acfae7a47dc9c957385fd9cf4bc5badcac685233942d686e959c58d990f55d76b3fd33b12a7d6040c);
}
```	

```solidity
  [25996] CryticToFoundry::test_malformedtokenUnlockTwTapPositionMsg_12334()
    ├─ emit DebugBytes(: 0xab7686604127cbee3894e0b61bbdafcfa442ad6e228feccdc0dc2defaa68e57034b9aaffd9b4f660a3708517a479ce85501f48fb3e978acfae7a47dc9c957385fd9cf4bc5badcac685233942d686e959c58d990f55d76b3fd33b12a7d6040c)
    ├─ emit DebugBytes(: 0xab7686604127cbee3894e0b61bbdafcfa442ad6e228feccdc0dc2defaa68e57034b9aaffd9b4f660a3708517a479ce85501f48fb)
    ├─ emit DebugAddress(: 0xAb7686604127CbEE3894e0b61bbdafCfa442AD6e)
    ├─ emit DebugUint(: 15632930341331812070391813023724101186232420026371244801021592211491894806779 [1.563e76])
    ├─ emit DebugAddress(: 0xAb7686604127CbEE3894e0b61bbdafCfa442AD6e)
    ├─ emit DebugUint(: 15632930341331812070391813023724101186232420026371244801021592211491894806779 [1.563e76])
    ├─ emit log_named_string(key: "Error", val: "tokenUnlockTwTapPositionMsg")
    ├─ emit log(val: "Error: Assertion Failed")
    ├─ [0] VM::store(VM: [0x7109709ECfa91a80626fF3989D68f67F5b1DD12D], 0x6661696c65640000000000000000000000000000000000000000000000000000, 0x0000000000000000000000000000000000000000000000000000000000000001)
    │   └─ ← ()
    └─ ← ()

```

As you can see:

```
0xab7686604127cbee3894e0b61bbdafcfa442ad6e228feccdc0dc2defaa68e57034b9aaffd9b4f660a3708517a479ce85501f48fb3e978acfae7a47dc9c957385fd9cf4bc5badcac685233942d686e959c58d990f55d76b3fd33b12a7d6040c
```

Results in the same encoding as:

```
0xab7686604127cbee3894e0b61bbdafcfa442ad6e228feccdc0dc2defaa68e57034b9aaffd9b4f660a3708517a479ce85501f48fb
```


# TapTokenReceiver

## QA - Good old Safe Transfer

https://github.com/Tapioca-DAO/tap-token/blob/050e666142e61018dbfcba64d295f9c458c69700/contracts/tokens/TapTokenReceiver.sol#L186-L187

```solidity
                IERC20(rewardToken_).transfer(sendTo_, dust);

```



## `_internalTransferWithAllowance` can present issues for contracts that have different addresses crossChain

xChain owner could be different and could perform operation on behalf

https://github.com/Tapioca-DAO/tapioca-periph/blob/2ddbcb1cde03b548e13421b2dba66435d2ac8eb5/contracts/tapiocaOmnichainEngine/TapiocaOmnichainReceiver.sol#L286-L292

```solidity
    function _internalTransferWithAllowance(address _owner, address srcChainSender, uint256 _amount) internal {
        if (_owner != srcChainSender) {
            _spendAllowance(_owner, srcChainSender, _amount);
        }

        _transfer(_owner, address(this), _amount);
    }
```


This means that this logic will only work with EOAs

And that SC Wallets will have to setup allowance manually, an extra burden for them to use the contracts



## QA - Overflow possible

`lockTwTapPositionMsg_.amount` is a `u256` pearlmit only uses up to `u200` so those calls will revert

```
        _approve(address(this), address(pearlmit), lockTwTapPositionMsg_.amount);
```


# Vesting


# QA

## Could pass duplicate users and lose those values

https://github.com/Tapioca-DAO/tap-token/blob/050e666142e61018dbfcba64d295f9c458c69700/contracts/tokens/Vesting.sol#L180-L183

```solidity
    function registerUsers(address[] calldata _users, uint256[] calldata _amounts) external onlyOwner {
        if (start > 0) revert Initialized();
        if (_users.length != _amounts.length) revert("Lengths not equal");

```

## Not exact amt | Need SWEEP

https://github.com/Tapioca-DAO/tap-token/blob/050e666142e61018dbfcba64d295f9c458c69700/contracts/tokens/Vesting.sol#L228

```solidity
seeded = _seededAmount; /// @audit Some is going to be stuck here as unclaimable
```


## No longer needed

https://github.com/Tapioca-DAO/tap-token/blob/050e666142e61018dbfcba64d295f9c458c69700/contracts/tokens/Vesting.sol#L210-L211

```solidity
        if (_cachedTotalAmount > _totalAmount) revert Overflow();

```



## BPS / Constants

https://github.com/Tapioca-DAO/tap-token/blob/050e666142e61018dbfcba64d295f9c458c69700/contracts/tokens/Vesting.sol#L230-L233

```solidity

        if (_initialUnlock > 10_000) revert AmountNotValid();
        if (_initialUnlock > 0) {
            uint256 initialUnlockAmount = (_seededAmount * _initialUnlock) / 10_000;
```

# Tapioca Periphery

# Magnetar

## LOW / MED - Multiple ways to reenter Magnetar while having approvals

- Token with hooks can reenter while having permissions from Magnetar
- Yieldbox address in data.yieldbox is not validated

Allows us to regain control while having unintended approvals

https://github.com/Tapioca-DAO/tapioca-periph/blob/2ddbcb1cde03b548e13421b2dba66435d2ac8eb5/contracts/Magnetar/modules/MagnetarAssetCommonModule.sol#L91-L101

```solidity
    // if `lendAmount` > 0:
    //      - add asset to SGL
    fraction = 0;
    if (lendAmount == 0 && depositData.deposit) {
        lendAmount = depositData.amount;
    }
    if (lendAmount > 0) {
        uint256 lendShare = yieldBox_.toShare(sglAssetId, lendAmount, false);
        fraction = ISingularity(singularityAddress).addAsset(user, user, false, lendShare);
    } /// @audit Would use the allowance here, it should be granting it there
```

## `data.yieldBox` is not sanitized

https://github.com/Tapioca-DAO/tapioca-periph/blob/2ddbcb1cde03b548e13421b2dba66435d2ac8eb5/contracts/Magnetar/modules/MagnetarBaseModule.sol#L55-L56

```solidity
        IYieldBox _yieldBox = IYieldBox(data.yieldBox);

```


https://github.com/Tapioca-DAO/tapioca-periph/blob/2ddbcb1cde03b548e13421b2dba66435d2ac8eb5/contracts/Magnetar/modules/MagnetarBaseModule.sol#L131-L162


## LOW / M - Magnetar allows to approve any approved target, this could be used to sweep if any contract that is whitelisted allows to have someone else pay on behalf

https://github.com/Tapioca-DAO/tapioca-periph/blob/2ddbcb1cde03b548e13421b2dba66435d2ac8eb5/contracts/Magnetar/Magnetar.sol#L188-L200

```solidity
    function _processPermitOperation(address _target, bytes calldata _actionCalldata, bool _allowFailure) private {
        if (!cluster.isWhitelisted(0, _target)) revert Magnetar_NotAuthorized(_target, _target);
        /// @dev owner address should always be first param.
        // permitAction(bytes,uint16)
        // permit(address owner...)
        // revoke(address owner...)
        // permitAll(address from,..)
        // permit(address from,...)
        // setApprovalForAll(address from,...)
        // setApprovalForAsset(address from,...)
        bytes4 funcSig = bytes4(_actionCalldata[:4]);
        if (
```

This allows adding and leaving open allowances from Magnetar to many targets, and could be a pre-cursor to a higher impact exploit

## Unwrap doesn't have a `from` field, but rather a `to` field

The implementation and the check for calling `unwrap` is incorrect

`unwrap`

https://github.com/Tapioca-DAO/tapioca-periph/blob/2ddbcb1cde03b548e13421b2dba66435d2ac8eb5/contracts/Magnetar/Magnetar.sol#L236-L237

```solidity
        // unwrap(address from,...) /// @audit Wrong https://github.com/Tapioca-DAO/TapiocaZ/blob/57750b7e997e5a1654651af9b413bbd5ea508f59/contracts/tOFT/TOFT.sol#L282-L284
        bytes4 funcSig = bytes4(_actionCalldata[:4]);
```


As with other `calldata`s the check on the `from` will be:

https://github.com/Tapioca-DAO/tapioca-periph/blob/2ddbcb1cde03b548e13421b2dba66435d2ac8eb5/contracts/Magnetar/Magnetar.sol#L241-L242

```solidity
            _checkSender(abi.decode(_actionCalldata[4:36], (address)));

```

However, for `unwrap` the first parameter is not `from` but rather `to`

Meaning that those tokens will be unwrapped to `to`

This allows:
- Claiming of any token that is in magnetar to arbitrary `to`
- Claiming and consequent loss of unwrapped assets by claiming the underlying token to any `approved` address

### Mitigation

`unwrap` probably doesn't need sanitization as it expects the token to be in Magnetar

## The function `_wrapSglReceipt` is declared twice


# Magnetar Helper 
## QA - Risky Rounding down for total debt

https://github.com/Tapioca-DAO/tapioca-periph/blob/2ddbcb1cde03b548e13421b2dba66435d2ac8eb5/contracts/Magnetar/MagnetarHelper.sol#L157-L164

```solidity
    function getAmountForBorrowPart(IMarket market, uint256 borrowPart) external view returns (uint256 amount) {
        Rebase memory _totalBorrowed;
        (uint128 totalBorrowElastic, uint128 totalBorrowBase) = market.totalBorrow();
        _totalBorrowed = Rebase(totalBorrowElastic, totalBorrowBase);

        return _totalBorrowed.toElastic(borrowPart, false); /// @audit Looks wrong?
    }

```


## Another round down

https://github.com/Tapioca-DAO/tapioca-periph/blob/2ddbcb1cde03b548e13421b2dba66435d2ac8eb5/contracts/Magnetar/MagnetarHelper.sol#L176-L177

```solidity
        return _totalBorrowed.toBase(amount, false);

```

# MagnetarAssetModule

## QA - Good old `safeApprove`

The finding is here just to make sure it get's douped, it should be made OOS by the 4nalyz3r report

https://github.com/Tapioca-DAO/tapioca-periph/blob/2ddbcb1cde03b548e13421b2dba66435d2ac8eb5/contracts/Magnetar/modules/MagnetarAssetModule.sol#L93-L94

```solidity
            IERC20(assetAddress).approve(address(_yieldBox), 0); /// @audit safe approve pls

```

## QA - Partial Deposit will require 2 calls

If a user has some tokens, and some tokens are in Magnetar, then they will have to separate the `repay` call in 2 parts

https://github.com/Tapioca-DAO/tapioca-periph/blob/2ddbcb1cde03b548e13421b2dba66435d2ac8eb5/contracts/Magnetar/modules/MagnetarAssetModule.sol#L99-L106

```solidity
        if (data.repayAmount > 0) {
            _setApprovalForYieldBox(data.market, _yieldBox);
            (Module[] memory modules, bytes[] memory calls) = IMarketHelper(data.marketHelper).repay(
                data.depositAmount > 0 ? address(this) : data.user, data.user, false, data.repayAmount
            ); /// @audit Unclear why this has approval when user has to pay
            _market.execute(modules, calls, true);
            _revertYieldBoxApproval(data.market, _yieldBox);
        }
```

# MagnetarBaseModule

## Needlessly re-deploys a helper on each call

https://github.com/Tapioca-DAO/tapioca-periph/blob/2ddbcb1cde03b548e13421b2dba66435d2ac8eb5/contracts/Magnetar/modules/MagnetarBaseModule.sol#L142

```solidity
TapiocaOmnichainEngineHelper _toeHelper = new TapiocaOmnichainEngineHelper(); /// @audit Why new contract?
```


# MagnetarOptionModule

## QA - This is not amount, these are shares

https://github.com/Tapioca-DAO/tapioca-periph/blob/2ddbcb1cde03b548e13421b2dba66435d2ac8eb5/contracts/Magnetar/modules/MagnetarOptionModule.sol#L173-L175

```solidity
            (Module[] memory modules, bytes[] memory calls) = IMarketHelper(data.externalData.marketHelper).repay(
                address(this), data.user, false, data.removeAndRepayData.repayAmount  /// TODO @audit shares as amount
            );
```

## This rounding could make repay 100% fail there

https://github.com/Tapioca-DAO/tapioca-periph/blob/2ddbcb1cde03b548e13421b2dba66435d2ac8eb5/contracts/Magnetar/MagnetarHelper.sol#L171-L177

```solidity
    function getBorrowPartForAmount(IMarket market, uint256 amount) external view returns (uint256 part) {
        Rebase memory _totalBorrowed;
        (uint128 totalBorrowElastic, uint128 totalBorrowBase) = market.totalBorrow();
        _totalBorrowed = Rebase(totalBorrowElastic, totalBorrowBase);

        return _totalBorrowed.toBase(amount, false); /// @audit Pretty sure this one makes it so they don't repay 100% of tokens
    }
```

UsdoMarketReceiverModule.sol

```solidity
        if (msg_.lendParams.repay) {
            if (msg_.lendParams.repayAmount == 0) {
                msg_.lendParams.repayAmount = IMagnetarHelper(IMagnetar(payable(msg_.lendParams.magnetar)).helper())
                    .getBorrowPartForAmount(msg_.lendParams.market, msg_.lendParams.depositAmount);
            }
```


# TapiocaOmnichainEngineHelper

## QA: Not empty bytes
https://github.com/Tapioca-DAO/tapioca-periph/blob/2ddbcb1cde03b548e13421b2dba66435d2ac8eb5/contracts/tapiocaOmnichainEngine/extension/TapiocaOmnichainEngineHelper.sol#L127-L128

```solidity
            composeMsg: "0x",

```


# Extra

## Pearlmit no longer supports infinite allowance

Due to a programming mistake, the infinite allowance functionality, which is part of `PermitC` Comments, was never implemented

## Approve assets the following

```solidity
    /**
     * @notice Approve an operator to spend a specific token / ID combination
     * @notice This function is compatible with ERC20, ERC721 and ERC1155
     * @notice To give unlimited approval for ERC20 and ERC1155, set amount to type(uint200).max
     * @notice When approving an ERC721, you MUST set amount to `1`
     * @notice When approving an ERC20, you MUST set id to `0`
     *
     * @dev    <h4>Postconditions:</h4>
     * @dev    1. Updates the approval for an operator to use an amount of a specific token / ID combination
     * @dev    2. If the expiration is 0, the approval is valid only in the context of the current block
     * @dev    3. If the expiration is not 0, the approval is valid until the expiration timestamp
     * @dev    4. If the provided amount is type(uint208).max, the approval is unlimited
     *
     * @param  token      The address of the token contract
     * @param  id         The token ID
     * @param  operator   The address of the operator
     * @param  amount     The amount of tokens to approve
     * @param  expiration The expiration timestamp of the approval
     */
    function approve(
        address token, 
        uint256 id, 
        address operator, 
        uint200 amount, 
        uint48 expiration) external {
        _storeApproval(token, id, amount, expiration, msg.sender, operator);
    }
```

Stating that type(uint200).max is unlimited (should not be decremented)


That's not the case


## Run this test

```solidity
function testERC20MaxIsNotImplemented(uint48 expiration)
     whenExpirationIsInTheFuture(expiration)
     whenTokenIsERC20()
     public {
        _mint20(testData.token, testData.owner, 1);

        assertEq(ERC20(testData.token).balanceOf(testData.owner), 1);

        changePrank(testData.owner);
        ERC20(testData.token).approve(address(permitC), 1);
        permitC.approve(testData.token, 0, testData.spender, type(uint200).max, uint48(block.timestamp));

        (uint256 allowanceAmount, uint256 allowanceExpiration) = permitC.allowance(testData.owner, testData.spender, testData.token, 0);
        assertEq(allowanceAmount, type(uint200).max);
        assertEq(allowanceExpiration, uint48(block.timestamp));

        changePrank(testData.spender);
        permitC.transferFromERC20(testData.owner, testData.spender, testData.token, 1);

        (allowanceAmount,) = permitC.allowance(testData.owner, testData.spender, testData.token, 0);

        assertEq(allowanceAmount, type(uint200).max);
    }
```

I sent the Limit Break Security an email but I have never head back

## Pearlmit effectively caps all values to uint200

This may not be an issue, but it's worth keeping in mind that you can no longer support u256 by using `PermitC`