# [L-1] Attacker can steal ETH from TapiocaOmnichainReceiver contract when its decoding multiple composed calls.
https://github.com/Tapioca-DAO/tapioca-periph/blob/032396f701be935b04a7e5cf3cb40a0136259dbc/contracts/tapiocaOmnichainEngine/TapiocaOmnichainReceiver.sol#L181-L183
https://github.com/Tapioca-DAO/tapioca-periph/blob/032396f701be935b04a7e5cf3cb40a0136259dbc/contracts/tapiocaOmnichainEngine/TapiocaOmnichainReceiver.sol#L153-L155

The BaseTapiocaOmnichainEngine override the `_payNative` function to check for msg.value < native fee values.

The issue is that, the `TapiocaOmnichainReceiver.lzCompose` function \_lzCompose can recursively traverse through multiple msgs in oftComposeMsg\_ one after another.
```
        if (nextMsg_.length > 0) {
            _lzCompose(srcChainSender_, _guid, nextMsg_);
        }
```
The internal function doesn't have its own msg.value, the original msg.value of the external lzCompose function input is exactly the same in each msg execution.
if `msgType_ == MSG_REMOTE_TRANSFER`, the `_internalRemoteTransferSendPacket` function will be called. `_internalRemoteTransferSendPacket` function will call `lzEndpoint.send{value: _nativeFee}`. And the LZ send accepts a `adapterParams` param to receive airdropped native gas from the relayer on destination. It can be used to steal all the eth in the TapToken and USDO contract.


# [N-1] Meaningless checks
https://github.com/Tapioca-DAO/tap-token/blob/20a83b1d2d5577653610a6c3879dff9df4968345/contracts/governance/twTAP.sol#L301

The `currentWeek()` is completely equivalent to `_timestampToWeek(block.timestamp)`, therefore this inequality is always false.

# [N-2] index traversal for rewardTokens of twTap should start from 1 
https://github.com/Tapioca-DAO/tap-token/blob/20a83b1d2d5577653610a6c3879dff9df4968345/contracts/governance/twTAP.sol#L220-L264
https://github.com/Tapioca-DAO/tap-token/blob/20a83b1d2d5577653610a6c3879dff9df4968345/contracts/governance/twTAP.sol#L447-L452
https://github.com/Tapioca-DAO/tap-token/blob/20a83b1d2d5577653610a6c3879dff9df4968345/contracts/governance/twTAP.sol#L561-L568

Because the index 0 is is reserved with 0x0 address, so the above traversal should start from 1.
```
mapping(IERC20 => uint256) public rewardTokenIndex; // Index 0 is reserved with 0x0 address
        rewardTokens.push(IERC20(address(0x0))); // 0 index is reserved
```

# [N-3] MagnetarStorage._getRevertMsg doesn't check the max len of the error string
https://github.com/Tapioca-DAO/tapioca-periph/blob/032396f701be935b04a7e5cf3cb40a0136259dbc/contracts/Magnetar/MagnetarStorage.sol#L102-L105

There is a max len check in the `ModuleManager._getRevertMsg`:
```
        if (_returnData.length > 1000) return "Module: reason too long";
```