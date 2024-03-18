The following are very simply and effective refactorings to save gas


# TapToken

## Make the variable immutable  - 2.1k
https://github.com/Tapioca-DAO/tap-token/blob/050e666142e61018dbfcba64d295f9c458c69700/contracts/tokens/TapToken.sol#L65-L66

```solidity
    uint256 public governanceEid; /// IMMUTABLE

```

## Use Constant instead of recomputing hashes - 200+ gas

https://github.com/Tapioca-DAO/tap-token/blob/050e666142e61018dbfcba64d295f9c458c69700/contracts/tokens/TapToken.sol#L327

```solidity
keccak256("Permit(address owner,address spender,uint256 value,uint256 nonce,uint256 deadline)");
```


https://github.com/OpenZeppelin/openzeppelin-contracts/blob/7417c5946f8a213a8e61eca8d3c5247bf3854249/contracts/token/ERC20/extensions/ERC20Permit.sol#L21C1-L22C105


## Make the variable immutable  - 2.1k

https://github.com/Tapioca-DAO/tap-token/blob/050e666142e61018dbfcba64d295f9c458c69700/contracts/governance/twTAP.sol#L111-L112

```solidity
    uint256 public creation; // Week 0 starts here /// @audit Make Immutable

```

## Cache to mem - 100

https://github.com/Tapioca-DAO/tap-token/blob/050e666142e61018dbfcba64d295f9c458c69700/contracts/governance/twTAP.sol#L468-L469

```solidity
        WeekTotals storage totals = weekTotals[lastProcessedWeek];

```

# TOLP


### Do not copy `sgl` to `memory`, instead use a pointer - 2.1k

https://github.com/Tapioca-DAO/tap-token/blob/050e666142e61018dbfcba64d295f9c458c69700/contracts/options/TapiocaOptionLiquidityProvision.sol#L197-L201

```solidity
        SingularityPool memory sgl = activeSingularities[_singularity]; /// GAS DO NOT COPY
        if (sgl.rescue) revert SingularityInRescueMode();

        uint256 sglAssetID = sgl.sglAssetID;
        if (sglAssetID == 0) revert SingularityNotActive();
```

-> Change to storage to cache the pointer, not copy the data

SingularityPool storage sgl = activeSingularities[_singularity]; /// GAS DO NOT COPY

# Use the memory value instead of storage - 100

https://github.com/Tapioca-DAO/tap-token/blob/050e666142e61018dbfcba64d295f9c458c69700/contracts/options/TapiocaOptionBroker.sol#L421-L424

```solidity
        bool success;
        (success, epochTAPValuation) = tapOracle.get(tapOracleData);
        if (!success) revert Failed();
        emit NewEpoch(epoch, epochTAP, epochTAPValuation); /// @audit GAS
```