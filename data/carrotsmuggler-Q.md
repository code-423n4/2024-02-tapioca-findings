# Low/QA report

- [Low/QA report](#lowqa-report)
  - [L01 Counters has been deprecated](#l01-counters-has-been-deprecated)
  - [L02 OTAP contract can be hijacked by calling `brokerClaim`](#l02-otap-contract-can-be-hijacked-by-calling-brokerclaim)
  - [L03 Unbounded loops can cause OOG errors](#l03-unbounded-loops-can-cause-oog-errors)
  - [L04 TOLP contract cannot receive batch of ERC1155 tokens](#l04-tolp-contract-cannot-receive-batch-of-erc1155-tokens)
  - [L05 Different order of operations for `_getDiscountedPaymentAmount`](#l05-different-order-of-operations-for-_getdiscountedpaymentamount)
  - [L06 Missing target sanitization in some functions in `TapiocaOmnichainExtExec.sol` contract](#l06-missing-target-sanitization-in-some-functions-in-tapiocaomnichainextexecsol-contract)
  - [L07 MagnetarBaseModule contract deploys new helper contracts every time](#l07-magnetarbasemodule-contract-deploys-new-helper-contracts-every-time)
  - [L08 Airdrop broker does not check the `success` field of the oracle's `peek` function](#l08-airdrop-broker-does-not-check-the-success-field-of-the-oracles-peek-function)
  - [L09 OTAP.sol `isApprovedOrOwner` doesnt check pearlmit approvals](#l09-otapsol-isapprovedorowner-doesnt-check-pearlmit-approvals)
  - [L10 `exitPositionAndRemoveCollateral` can be denied by any user](#l10-exitpositionandremovecollateral-can-be-denied-by-any-user)
  - [QA01 Missing event](#qa01-missing-event)

## L01 Counters has been deprecated

The Counters library is used in the `ERC721Permit.sol` contract. This library has been deprecated by open zeppelin. Consider removing the usage of this library.

## L02 OTAP contract can be hijacked by calling `brokerClaim`

The OTAP.sol contract has the function `brokerClaim` which can be called by anyone. This function can only be called once, so this is only vulnerable immediately after deployment.

```solidity
function brokerClaim() external {
    if (broker != address(0)) revert OnlyOnce();
    broker = msg.sender;
}
```

As seen above, if this function is called, the `broker` address is set to the caller. This gives the caller the ability mint OTAP tokens. This situation, if it happens, will be easily identified and will at most force a redeployment of the contract.

Consider adding access control to this function, and only allow a specific `owner` to set a `broker`.

## L03 Unbounded loops can cause OOG errors

Contracts such as the TOPL contract loops over all the singularity markets.

```solidity
uint256[] memory _singularities = singularities;
uint256 sglLength = _singularities.length;

for (uint256 i; i < sglLength; i++) {
    //...
}
```

The `singularities` variable is defined as an array and can be added to by calling the `registerSingularity` function. There is no limit to which this array can grow to. The issue is that if this array is too large, there might not be enough gas in the block to accommodate such a transaction and would result in an out of gas error.

Similar problems are also faced with the array of reward tokens in the twTAP contract. The `_claimTwpTapRewardsReceiver` function in the tap token receiver module iterates over this list of tokens and sends out cross chain messages for each. If the list is too large, or if the gas dedicated to the messages is too high, this can easily lead to a out of gas error.

```solidity
for (uint256 i = 1; i < rewardTokensLength_;) {
    // Send back packet
    TapTokenSender(rewardToken_).sendPacket{
        value: claimTwTapRewardsMsg_.sendParam[sendParamIndex].fee.nativeFee
    }(claimTwTapRewardsMsg_.sendParam[sendParamIndex], bytes(""));
}
```

Consider limiting the size of these arrays.

## L04 TOLP contract cannot receive batch of ERC1155 tokens

The TOLP contract cannot receive a series of ERC1155 tokens. This is because the `onERC1155BatchReceived` function is called by the ERC1155 contract on a transfer, and the TOLP contract returns an empty bytes array, instead of the magic value.

```solidity
function onERC1155BatchReceived(
    address operator,
    address from,
    uint256[] calldata ids,
    uint256[] calldata values,
    bytes calldata data
) external returns (bytes4) {
    return bytes4(0);
}
```

## L05 Different order of operations for `_getDiscountedPaymentAmount`

The `AirdropBroker.sol` contract and the `TapiocaOptionBroker.sol` contract both implement the `_getDiscountedPaymentAmount` function to calculate the amount to be paid in order to exercise the option. The issue is that while they essentially follow the same mathematics, they have a different order of operations.

The AirdropBroker contract does a division and then a subtraction:

```solidity
uint256 rawPaymentAmount = _otcAmountInUSD / _paymentTokenValuation;
paymentAmount = rawPaymentAmount - muldiv(rawPaymentAmount, _discount, 100e4);
```

While the Option Broker does the opposite:

```solidity
uint256 discountedOTCAmountInUSD = _otcAmountInUSD - muldiv(_otcAmountInUSD, _discount, 100e4); // 1e4 is discount decimals, 100 is discount percentage
paymentAmount = discountedOTCAmountInUSD / _paymentTokenValuation;
```

This would lead to different values for the `paymentAmount` in the two contracts, everything else remaining the same due to the rounding errors. The first method is more prone to rounding errors since it does the division first, losing precision. It is advised to use the same order of operations in both contracts.

## L06 Missing target sanitization in some functions in `TapiocaOmnichainExtExec.sol` contract

The `TapiocaOmnichainExtExec.sol` contract has a number of functions which can be invoked. Some of those functions check if the target is whitelisted by using a `_sanitizeTarget` function.

```solidity
function yieldBoxPermitAll(bytes memory _data) public {
    YieldBoxApproveAllMsg memory approval = TapiocaOmnichainEngineCodec.decodeYieldBoxApproveAllMsg(_data);
    _sanitizeTarget(approval.target);
    //...
}

function _sanitizeTarget(address target) private view {
    if (!cluster.isWhitelisted(0, target)) {
        revert InvalidApprovalTarget(target);
    }
}
```

This check is however missing in the ERC20, ERC721 and pearlmit approval functions. This can allows the contract to run arbitrary code, and is thus advised to sanitize the target in all functions.

## L07 MagnetarBaseModule contract deploys new helper contracts every time

The `MagnetarBaseModule.sol` contract is used to prepare and send cross chain messages. These messages are constructed with the help of helper contracts. The issue is that these helper contracts are deployed everytime this function is called.

```solidity
function _lzCustomWithdraw(
    //...
) private {
    PrepareLzCallReturn memory prepareLzCallReturn = _prepareLzSend(_asset, _lzSendParam, _lzSendGas, _lzSendVal);

    //@audit creates new helper each time
    TapiocaOmnichainEngineHelper _toeHelper = new TapiocaOmnichainEngineHelper();
```

```solidity
function _prepareLzSend(address _asset, LZSendParam memory _lzSendParam, uint128 _lzSendGas, uint128 _lzSendVal)
    private
    returns (PrepareLzCallReturn memory prepareLzCallReturn)
{
    TapiocaOmnichainEngineHelper _toeHelper = new TapiocaOmnichainEngineHelper();
```

This is unnecessary since the helper contracts are stateless contracts with only view functions and can be re-used. This is very gas inefficient to deploy them multiple times over.

## L08 Airdrop broker does not check the `success` field of the oracle's `peek` function

The Airdrop broker contract, option broker contract, and the Magnetar helper contracts all call the `peek` function of the oracle contract.

```solidity
(, uint256 paymentTokenValuation) = paymentTokenOracle.oracle.peek(paymentTokenOracle.oracleData);
```

However, in all three cases, the first return value is not checked. The `peek` function returns a `success` variable as the first value, which according to the docs of the oracle contract, should be checked.

```bash
/// @return success if no valid (recent) rate is available, return false else true.
```

So incase of a downed oracle, this should be checked since that would invalidate the oracle results. However this check is not performed. Consider checking the `success` value of the oracle `peek` function.

## L09 OTAP.sol `isApprovedOrOwner` doesnt check pearlmit approvals

Most of the NFTs in the system implement the `isApprovedOrOwner` and checks not only the approval in the ERC721 contract itself, but also in the pearlmit contract in case the user has given approval via permitC. For example, here is the same function for the TOLP contract:

```solidity
function _isApprovedOrOwner(address _spender, uint256 _tokenId) internal view override returns (bool) {
    return super._isApprovedOrOwner(_spender, _tokenId)
        || isERC721Approved(_ownerOf(_tokenId), _spender, address(this), _tokenId);
}
```

en
The `isERC721Approved` checks for the approval in the permitC conrtact. This is also the case for the aoTAP token. However, for the OTAP token, this check is missing. The function calls the default ERC721 `_isApprovedOrOwner` function, which does not do pearlmit approval checks

```solidity
function isApprovedOrOwner(address _spender, uint256 _tokenId) external view returns (bool) {
    return _isApprovedOrOwner(_spender, _tokenId);
}
```

Consider changing it so they all behave in the same way.

## L10 `exitPositionAndRemoveCollateral` can be denied by any user

The function `exitPositionAndRemoveCollateral` in the MagnetarOptionModule.sol contract can be denied by anyone if the user is also trying to exit their position from the TOB contract.

The `data.removeAndRepayData.exitData.exit` parameter of the function above decides if the user/caller wants to exit their position from the TOB contract as well. If so, the Magnetar contract transfers out the OTAP token from the user, and then calls `exitPosition` on the TOB contract.

```solidity
if (ownerOfTapTokenId == data.user) {
    // IERC721(oTapAddress).safeTransferFrom(
    //     data.user, address(this), data.removeAndRepayData.exitData.oTAPTokenID, "0x"
    // );
    bool isErr = pearlmit.transferFromERC721(
        data.user, address(this), oTapAddress, data.removeAndRepayData.exitData.oTAPTokenID
    );
    if (isErr) revert Magnetar_ExtractTokenFail();
}
ITapiocaOptionBroker(data.removeAndRepayData.exitData.target).exitPosition(
    data.removeAndRepayData.exitData.oTAPTokenID
);
```

The issue is that the `exitPosition` function in TOB contract does not need the token or any access control. Anyone can call that function with any token ID, and as long as it is expired, they will get kicked out. The TOB contract is designed this way such that anyone who doesnt have a locked position anymore can be kicked out of the system, and their votes and twAML weights will be taken out of the calculations.

So if a user calls a long operation on the Magnetar contract with the `data.removeAndRepayData.exitData.exit` parameter set to true, anyone can see that message on the base chain and call `exitPosition` on that tokenID in the target chain themselves. This will result in the actual user's lzcompose failing, since the token is already burnt and the user is already kicked out of the system. This means all the subsequent operations that the user was trying to do through Magnetar will not happen as well.

While this does not cause harm to the system, this can still affect user UX and make user transactions revert, creating a temporary DOS. Consider putting the `exitPosition` function in a try-catch statement so it cannot be denied by others.

## QA01 Missing event

The function `setNftLoader` in the `ERC721NftLoader.sol` contract does a state cange but does not emit an event. Consider adding an event.
