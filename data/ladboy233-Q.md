| Index | Title |
|-------|-------|
| L1     | Magnetar#_processPermitOperation missing support for a few permit relate operation |
| L2     | Return value from ITapiocaOracle(market.oracle()).peek is not handled |
| L3     | Inconsistent implementation with airdrop epoch |
| L4     | Missing Event emission when state changes in a few contracts |
| L5     | Should validate if the merkle tree root is set before the airdrop epoch goes to 2 |

# Magnetar#_processPermitOperation missing support for a few permit relate operation

When multicall via Magnetar#burst, the user can give contract permission

```
/// @dev Permit on YB, or an SGL/BB market
if (_action.id == MagnetarAction.Permit) {
    _processPermitOperation(_action.target, _action.call, _action.allowFailure);
    continue; // skip the rest of the loop
}
```

which calls:

```
if (
    funcSig == IPermitAll.permitAll.selector || funcSig == IPermitAll.revokeAll.selector
        || funcSig == IPermit.permit.selector || funcSig == IPermit.revoke.selector
        || funcSig == IYieldBox.setApprovalForAll.selector || funcSig == IYieldBox.setApprovalForAsset.selector 
        || funcSig == IERC20.approve.selector || funcSig == IPearlmit.approve.selector
        || funcSig == IERC721.approve.selector 
) {
    _checkSender(abi.decode(_actionCalldata[4:36], (address)));
    // No need to send value on permit
    _executeCall(_target, _actionCalldata, 0, _allowFailure);
    return;
}
```

But there are a few permit related method that are not supported.

If we take a look at the IYieldBox interface,

the function 

```
   function permitToken(
        address token,
        address from,
        address to,
        uint256 amount,
        uint256 deadline,
        uint8 v,
        bytes32 r,
        bytes32 s
    ) external;
```

is missing.

It is recommendated to check if the function selector is permitToken as well in the burst multicall.

# Return value from ITapiocaOracle(market.oracle()).peek is not handled

In [MagnetarHelper.sol](https://github.com/Tapioca-DAO/tapioca-periph/blob/032396f701be935b04a7e5cf3cb40a0136259dbc/contracts/Magnetar/MagnetarHelper.sol#L305), the return value from oracel query is not validated:

```
(, info.oracleExchangeRate) = ITapiocaOracle(market.oracle()).peek(market.oracleData());
info.spotExchangeRate = ITapiocaOracle(market.oracle()).peekSpot(market.oracleData());
```

If we take a look ath the function peek returned parameter

```
    /// @notice Check the last exchange rate without any state changes.
    /// @param data Usually abi encoded, implementation specific data that contains information and arguments to & about the oracle.
    /// For example:
    /// (string memory collateralSymbol, string memory assetSymbol, uint256 division) = abi.decode(data, (string, string, uint256));
    /// @return success if no valid (recent) rate is available, return false else true.
    /// @return rate The rate of the requested asset / pair / pool.
    function peek(bytes calldata data) external view returns (bool success, uint256 rate);

```

the first returned parameter indicate if the oracle query is successful,

but the code does not validate if the query is successful and return the rate directly

It is recommended to return the boolean and let upper call handle the case in case the query failed

or handle the return value in the oracle query directly:

```
(bool success, info.oracleExchangeRate) = ITapiocaOracle(market.oracle()).peek(market.oracleData());
info.spotExchangeRate = ITapiocaOracle(market.oracle()).peekSpot(market.oracleData());
info.oracleSucceed = success
```

or

```
(bool success, info.oracleExchangeRate) = ITapiocaOracle(market.oracle()).peek(market.oracleData());
if (!success) {
    revert("oracle query failed");
}
info.spotExchangeRate = ITapiocaOracle(market.oracle()).peekSpot(market.oracleData());
```

Otherwise the service risking using outdated / failure oracle rate 

# Inconsistent implementation with airdrop epoch

In tapica docs,

https://docs.tapioca.xyz/tapioca/genesis/option-airdrop#phase-four-twtap-lockers

> The phase four and final Option Airdrop tranche will be rewarded to twTAP lockers. 

the docs says the airdrop epoch for option will span 4 epoch,

while in the implementation, [there are 8 epoch](https://github.com/Tapioca-DAO/tap-token/blob/20a83b1d2d5577653610a6c3879dff9df4968345/contracts/option-airdrop/AirdropBroker.sol#L98)

```
    uint256 public EPOCH_DURATION = 2 days; // Becomes 7 days at the start of the phase 4
    uint256 public constant LAST_EPOCH = 8; // 8 epochs, 41 days long
```

It is recommend to either update the code or the doc to make sure the code and docs match.

# Missing Event emission when state changes in a few contracts.

A lot of state change function is missing event emission,

it is recommended to add event emission for offchain service to monitor the state change

In [ERC721NFTLoader.sol](https://github.com/Tapioca-DAO/tap-token/blob/20a83b1d2d5577653610a6c3879dff9df4968345/contracts/erc721NftLoader/ERC721NftLoader.sol#L45)

```
/**
* @notice Set the NFT loader contract
*/
function setNftLoader(address _nftLoader) external onlyOwner {
    nftLoader = INFTLoader(_nftLoader);
}
```

and In [TapiocaOptionBroker.sol](https://github.com/Tapioca-DAO/tap-token/blob/20a83b1d2d5577653610a6c3879dff9df4968345/contracts/options/TapiocaOptionBroker.sol#L440)

```
    /**
     * @notice Set the `VIRTUAL_TOTAL_AMOUNT` state variable.
     * @param _virtualTotalAmount The new state variable value.
     */
    function setVirtualTotalAmount(uint256 _virtualTotalAmount) external onlyOwner {
        VIRTUAL_TOTAL_AMOUNT = _virtualTotalAmount;
    }

    /**
     * @notice Set the minimum weight factor.
     * @param _minWeightFactor The new minimum weight factor.
     */
    function setMinWeightFactor(uint256 _minWeightFactor) external onlyOwner {
        MIN_WEIGHT_FACTOR = _minWeightFactor;
    }
```

In [twTAP.sol contract](https://github.com/Tapioca-DAO/tap-token/blob/20a83b1d2d5577653610a6c3879dff9df4968345/contracts/governance/twTAP.sol#L489) as well:

```
   /**
     * @notice Set the `VIRTUAL_TOTAL_AMOUNT` state variable.
     * @param _virtualTotalAmount The new state variable value.
     */
    function setVirtualTotalAmount(uint256 _virtualTotalAmount) external onlyOwner {
        VIRTUAL_TOTAL_AMOUNT = _virtualTotalAmount;
    }

    /**
     * @notice Set the minimum weight factor.
     * @param _minWeightFactor The new minimum weight factor.
     */
    function setMinWeightFactor(uint256 _minWeightFactor) external onlyOwner {
        MIN_WEIGHT_FACTOR = _minWeightFactor;
    }
```

# Should validate if the merkle tree root is set before the airdrop epoch goes to 2

In AirdropBroker.sol in phrase 2, the airdrop option is distributed based on merkle tree root

this is the function [setPhrase2MerkleRoots](https://github.com/Tapioca-DAO/tap-token/blob/20a83b1d2d5577653610a6c3879dff9df4968345/contracts/option-airdrop/AirdropBroker.sol#L309)

```
  function setPhase2MerkleRoots(bytes32[4] calldata _merkleRoots) external onlyOwner {
        if (epoch >= 2) revert NotValid();
        phase2MerkleRoots = _merkleRoots;
        emit Phase2MerkleRootsUpdated();
    }
```

But if the epoch phrase is 2 or beyond, no one can set the merkle root,

so before the epoch advance to 2, the code should validate if the merkle root is set

```
  
   function isMerkleRooteSet() public {
     if(phase2MerkleRoots[0] == bytes32(0) || phase2MerkleRoots[1] == bytes32(0)) ||
        phase2MerkleRoots[2] == bytes32(0) || phase2MerkleRoots[3] == bytes32(0) {
            return false;
        }
     return True;
   }

   function newEpoch() external tapExists {
        if (block.timestamp < lastEpochUpdate + EPOCH_DURATION) {
            revert TooSoon();
        }

        // Update epoch info
        lastEpochUpdate = uint64(block.timestamp);
        epoch++;

        // add check here
        if (epoch == 2 && !isMerkleRooteSet()) {
            revert("phrase 2 merkle root not set");
        }

        // At epoch 4, change the epoch duration to 7 days
        if (epoch == 4) {
            EPOCH_DURATION = 7 days;
        }

        // Get epoch TAP valuation
        (bool success, uint256 _epochTAPValuation) = tapOracle.get(tapOracleData);
        if (!success) revert Failed();
        epochTAPValuation = uint128(_epochTAPValuation);
        emit NewEpoch(epoch, epochTAPValuation);
    }
```