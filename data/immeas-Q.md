# QA Report


## Summary

| id | title |
| --- | --- |
| [L-01](#l-01-a-lot-of-calls-in-magnetaractionpermit-enables-anyone-to-steal-whitelisted-tokens-held-by-magnetar) | a lot of calls in `MagnetarAction.Permit` enables anyone to steal whitelisted tokens held by `Magnetar` |
| [L-02](#l-02-unnecessary-approval-in-magnetarmintmodule) | Unnecessary approval in `MagnetarMintModule` |
| [L-03](#l-03-fixed-min-amount-in-exerciseoption) | fixed min amount in `exerciseOption` |
| [L-04](#l-04-twtap-position-not-burnt-on-exit) | `TwTAP` position not burnt on exit |
| [NC-01](#nc-01-use-modifier-onlyhostchain) | use modifier `onlyHostChain` |
| [NC-02](#nc-02-unnecessary-check-for-0-amount-in-airdropbroker_processotcdeal) | unnecessary check for 0 amount in `AirdropBroker::_processOTCDeal` |
| [NC-03](#nc-03-pausing-tapiocaoptionliquidityprovision-doesnt-do-anything) | Pausing `TapiocaOptionLiquidityProvision` doesn't do anything |
| [NC-04](#nc-04-unused-field) | Unused field |
| [NC-05](#nc-05-link-broken) | Link broken |
| [NC-06](#nc-06-openzeppelin-draft-erc20permit-is-deprecated) | OpenZeppelin `draft-ERC20Permit` is deprecated |


## Low


### L-01 a lot of calls in `MagnetarAction.Permit` enables anyone to steal whitelisted tokens held by `Magnetar`
In `Magnetar` a user can batch a lot of calls to the Tapioca ecosystem. One of these is the `MagnetarAction.Permit`. This performs various permit operations, [`Magnetar::_processPermitOperation`](https://github.com/Tapioca-DAO/tapioca-periph/blob/032396f701be935b04a7e5cf3cb40a0136259dbc/contracts/Magnetar/Magnetar.sol#L199-L212):
```solidity
File: tapioca-periph/contracts/Magnetar/Magnetar.sol

199:        bytes4 funcSig = bytes4(_actionCalldata[:4]);
200:        if (
201:            funcSig == IPermitAll.permitAll.selector || funcSig == IPermitAll.revokeAll.selector
202:                || funcSig == IPermit.permit.selector || funcSig == IPermit.revoke.selector
203:                || funcSig == IYieldBox.setApprovalForAll.selector || funcSig == IYieldBox.setApprovalForAsset.selector 
204:                || funcSig == IERC20.approve.selector || funcSig == IPearlmit.approve.selector
205:                || funcSig == IERC721.approve.selector 
206:        ) {
207:            /// @dev Owner param check. See Warning above.
208:            _checkSender(abi.decode(_actionCalldata[4:36], (address)));
209:            // No need to send value on permit
210:            _executeCall(_target, _actionCalldata, 0, _allowFailure);
211:            return;
212:        }
```

Let's also have a look at [`MagnetarStorage::_checkSender`](https://github.com/Tapioca-DAO/tapioca-periph/blob/032396f701be935b04a7e5cf3cb40a0136259dbc/contracts/Magnetar/MagnetarStorage.sol#L93-L97):
```solidity
File: tapioca-periph/Magnetar/MagnetarStorage.sol

93:    function _checkSender(address _from) internal view {
94:        if (_from != msg.sender && !cluster.isWhitelisted(0, msg.sender)) {
95:            revert Magnetar_NotAuthorized(msg.sender, _from);
96:        }
97:    }
```
Here, the first argument to whatever call is done in `_processPermitOperation` is checked. The important thing here is that it always passes if the first argument is `msg.sender`

This is troublesome for the calls: `IYieldBox::setApprovalForAll`, `IYieldBox::setApprovalForAsset`, `IERC20::approve`, and `IERC721::approve`. As they all have the operator/approvee as the first argument and use `msg.sender` as the "owner".

Hence any of these calls would pass with `msg.sender`, granting allowance from `Magnetar` to `msg.sender` which can be any user.

#### Impact
Any user can steal any whitelisted tokens held by `Magnetar`. However, as `Magnetar` is just a helper contract it is not supposed to hold any tokens.

#### PoC
Test in `tap-token/test/MagnetarApproval.t.sol`:
```solidity
    function testTakeERC721() public {
        erc721.mint(address(magnetar),1);

        MagnetarCall memory approve = MagnetarCall({
            id: MagnetarAction.Permit,
            target: address(erc721),
            value: 0,
            allowFailure: false,
            call: abi.encodeWithSelector(IERC721.setApprovalForAll.selector, address(this), true)
        });

        MagnetarCall[] memory calls = new MagnetarCall[](1);
        calls[0] = approve;

        magnetar.burst(calls);

        erc721.transferFrom(address(magnetar), address(this), 1);
        assertEq(erc721.ownerOf(1),address(this));
    }

    function testTakeERC20() public {
        erc20.mint(address(magnetar),1e18);

        MagnetarCall memory approve = MagnetarCall({
            id: MagnetarAction.Permit,
            target: address(erc20),
            value: 0,
            allowFailure: false,
            call: abi.encodeWithSelector(IERC20.approve.selector, address(this), 1e18)
        });

        MagnetarCall[] memory calls = new MagnetarCall[](1);
        calls[0] = approve;

        magnetar.burst(calls);

        erc20.transferFrom(address(magnetar),address(this),1e18);
        assertEq(erc20.balanceOf(address(this)),1e18);
    }
```

Full test code can be found [here](https://gist.github.com/0ximmeas/470717ac554ea9ab350f5bf4d89fa730)

#### Recommendation
Consider rethinking how `MagnetarAction.Permit` should work. As there is support in modules for a lot of the calls it is used for perhaps it is unnecessary to have.


### L-02 Unnecessary approval in `MagnetarMintModule`

[`MagnetarMintCommonModule::_participateOnTOLP`](https://github.com/Tapioca-DAO/tapioca-periph/blob/032396f701be935b04a7e5cf3cb40a0136259dbc/contracts/Magnetar/modules/MagnetarMintCommonModule.sol#L88-L89):
```solidity
File: tapioca-periph/contracts/Magnetar/modules/MagnetarMintCommonModule.sol

88:        IERC721(lockDataTarget).approve(participateData.target, tOLPTokenId);
89:        uint256 oTAPTokenId = ITapiocaOptionBroker(participateData.target).participate(tOLPTokenId);
```

This approval is unnecessary as `TapiocaOptionBroker::participate` uses `Pearlmit` to transfer the token. Hence the approval is needed against `Pearlmit`, not `TapiocaOptionBroker`.

#### Recommendation
Consider approving `Pearlmit` instead of `participateData.target`.


### L-03 fixed min amount in `exerciseOption`
In [`TapiocaOptionBroker::exerciseOption`](https://github.com/Tapioca-DAO/tap-token/blob/20a83b1d2d5577653610a6c3879dff9df4968345/contracts/options/TapiocaOptionBroker.sol#L396) there is a minimum amount you can claim:
```solidity
File: tap-token/blob/options/TapiocaOptionBroker.sol

396:        if (chosenAmount < 1e18) revert TooLow();
```

This amount is fixed. Hence if `TAP` would become valuable this can be very limiting in how much a large an option a user can exercise.

#### Recommendation
Consider making this amount mutable and a call to change it.


### L-04 `TwTAP` position not burnt on exit
When exiting a position in `TapiocaOptionBroker` the users `oTAP` position [is burned](https://github.com/Tapioca-DAO/tap-token/blob/20a83b1d2d5577653610a6c3879dff9df4968345/contracts/options/TapiocaOptionBroker.sol#L353).

This is not done when exiting a `TwTAP` position however. This will leave a worthless NFT in the users account.

#### Recommendation
Consider burning the `TwTAP` position when exiting.


## Non-critical / Informational


### NC-01 use modifier `onlyHostChain`
In `TapToken` there is a modifier `onlyHostChain` to check that the operation is done one the `governanceEid`-chain. This is used in the call `setTwTAP`.

It is however, not used in the call [`TapToken::emitForWeek`](https://github.com/Tapioca-DAO/tap-token/blob/20a83b1d2d5577653610a6c3879dff9df4968345/contracts/tokens/TapToken.sol#L385):
```solidity
File: tap-token/contracts/tokens/TapToken.sol

385:        if (_getChainId() != governanceEid) revert NotValid();
```

Consider using the modifier `onlyHostChain` there as well.


### NC-02 unnecessary check for 0 amount in `AirdropBroker::_processOTCDeal`
In [`AirdropBroker::_processOTCDeal`](https://github.com/Tapioca-DAO/tap-token/blob/20a83b1d2d5577653610a6c3879dff9df4968345/contracts/option-airdrop/AirdropBroker.sol#L492) there is a check for `0` amount:
```solidity
File: tap-token/contracts/option-airdrop/AirdropBroker.sol

492:        if (tapAmount == 0) revert TapAmountNotValid();
```

However, `tapAmount` can never be `0` as it is already checked to be `>1e18` before calling in [`exerciseOption`](https://github.com/Tapioca-DAO/tap-token/blob/20a83b1d2d5577653610a6c3879dff9df4968345/contracts/option-airdrop/AirdropBroker.sol#L254):
```solidity
File: tap-token/contracts/option-airdrop/AirdropBroker.sol

254:        if (chosenAmount < 1e18) revert TooLow();
```

Consider removing the `tapAmount == 0` check in `_processOTCDeal`.


### NC-03 Pausing `TapiocaOptionLiquidityProvision` doesn't do anything
[`TapiocaOptionLiquidityProvision::setPause`](https://github.com/Tapioca-DAO/tap-token/blob/20a83b1d2d5577653610a6c3879dff9df4968345/contracts/options/TapiocaOptionLiquidityProvision.sol#L363-L372):
```solidity
File: tap-token/contracts/options/TapiocaOptionLiquidityProvision.sol

363:    /**
364:     * @notice Un/Pauses this contract.
365:     */
366:    function setPause(bool _pauseState) external onlyOwner {
367:        if (_pauseState) {
368:            _pause();
369:        } else {
370:            _unpause();
371:        }
372:    }
```

However, no other call checks if the contract is paused or not, hence pausing will have no effect.

Consider removing the pause functionality to avoid confusion or consider what should be stopped if the contract is paused.


### NC-04 Unused field
[`Vesting::UserData`](https://github.com/Tapioca-DAO/tap-token/blob/20a83b1d2d5577653610a6c3879dff9df4968345/contracts/tokens/Vesting.sol#L47-L52):
```solidity
File: tap-token/contracts/tokens/Vesting.sol

47:    struct UserData {
48:        uint256 amount;
49:        uint256 claimed;
50:        uint256 latestClaimTimestamp;
           // @audit-issue not used 
51:        bool revoked;
52:    }
```

the field `revoked` in `UserData` is not used, consider removing it.


### NC-05 Link broken
[`TWAML::sqrt`](https://github.com/Tapioca-DAO/tap-token/blob/20a83b1d2d5577653610a6c3879dff9df4968345/contracts/options/twAML.sol#L144):
```solidity
File: tap-token/contracts/options/twAML.sol

144:    // babylonian method (https://en.wikipedia.org/wiki/Methods_of_computing_square_roots#Babylonian_method)
```

It's renamed to https://en.wikipedia.org/wiki/Methods_of_computing_square_roots#Heron's_method


### NC-06 OpenZeppelin `draft-ERC20Permit` is deprecated
[`TapToken`](https://github.com/Tapioca-DAO/tap-token/blob/20a83b1d2d5577653610a6c3879dff9df4968345/contracts/tokens/TapToken.sol#L11) uses OpenZeppelin `draft-ERC20Permit.sol`, this is the contents:
```solidity
// EIP-2612 is Final as of 2022-11-01. This file is deprecated.

import "./ERC20Permit.sol";
```

Consider importing `ERC20Permit.sol` directly instead.