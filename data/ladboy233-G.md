| Index | Title |
|-------|-------|
| G1     | No need to create new helper address every time when custom withdraw |
| G2     | Use merkle tree when doing option airdrop |
| G3     | Use merkle tree when vesting token |
| G4     | Consider deploy market helper contract only once |

# no need to create new helper address every time when custom withdraw

When perform custom withdraw,

a new TapiocaOmnichainEngineHelper contract is [created every time](https://github.com/Tapioca-DAO/tapioca-periph/blob/032396f701be935b04a7e5cf3cb40a0136259dbc/contracts/Magnetar/modules/MagnetarBaseModule.sol#L142)

```
PrepareLzCallReturn memory prepareLzCallReturn = _prepareLzSend(_asset, _lzSendParam, _lzSendGas, _lzSendVal);
TapiocaOmnichainEngineHelper _toeHelper = new TapiocaOmnichainEngineHelper();
```

A much more gas efficient approach is only deploy the TapiocaOmnichainEngineHelper contract once and query the helper function from the contract each time user calls custom withdraw.

So the protocol only needs to pay for the gas fee for TapiocaOmnichainEngineHelper one time deployment.

User does not need to pay the gas to create TapiocaOmnichainEngineHelper contract every time + this approach reduce the deployment size for the MagnetarBaseModule contract.

# Use merkle tree when doing option airdrop

In the current implementation, when doing option airdrop, the admin has to run a large for loop to set the allocated token amount for each address one by one

```
  function registerUsersForPhase(uint256 _phase, address[] calldata _users, uint256[] calldata _amounts)
        external
        onlyOwner
    {
        if (_users.length != _amounts.length) revert NotValid();

        if (_phase == 1) {
            if (epoch >= 1) revert NotValid();
            for (uint256 i; i < _users.length; i++) {
                phase1Users[_users[i]] = _amounts[i];
            }
        }
        /// @dev We want to be able to set phase 4 users in the future on subsequent epochs
        else if (_phase == 4) {
            for (uint256 i; i < _users.length; i++) {
                phase4Users[_users[i]] = _amounts[i];
            }
        }
    }
```

This is a very gas inefficient way to distribute airdrop.

A more efficient way is set a merkle tree root and let user claim themselve, the merkle tree proof can be generated offchain.

https://github.com/Anish-Agnihotri/merkle-airdrop-starter

There is already very mature way to integrate with merkle tree airdrop contract.

# Use merkle tree when vesting token

In vesting.sol contract,

admin has to [register user address paired with amount](https://github.com/Tapioca-DAO/tap-token/blob/20a83b1d2d5577653610a6c3879dff9df4968345/contracts/tokens/Vesting.sol#L180) one by one.

```
  function registerUser(address _user, uint256 _amount) external onlyOwner {
        if (start > 0) revert Initialized();
        if (_user == address(0)) revert AddressNotValid();
        if (_amount == 0) revert AmountNotValid();
        if (users[_user].amount > 0) revert AlreadyRegistered();

        UserData memory data;
        data.amount = _amount;
        users[_user] = data;

        totalRegisteredAmount += _amount;

        emit UserRegistered(_user, _amount);
    }

```

This is a very gas inefficient way to distribute token because the for loop consume large amount of gas.

A more efficient way is set a merkle tree root and let user claim themselve, the merkle tree proof can be generated offchain.

https://github.com/Anish-Agnihotri/merkle-airdrop-starter

There is already very mature way to integrate with merkle tree token distribution contract.

# Consider deploy market helper contract only once

Across module implementation,

the code has to check if the market helper address is whitelisted every time.

```
if (!cluster.isWhitelisted(0, address(data.marketHelper))) {
    revert Magnetar_TargetNotWhitelisted(address(data.marketHelper));
}
```

There is a gas efficient way to reduce the code and optimize this check.

The protocol can deploy the marketHelper address only once behind a upgradeable proxy contract and hard coe the marketHelper address

and remove all check 

```
if (!cluster.isWhitelisted(0, address(data.marketHelper))) {
    revert Magnetar_TargetNotWhitelisted(address(data.marketHelper));
}
```

Because marketHelper is hardcoded, we know it is a trusted address that encode call data and because it is upgradeable, 

the protocol can extend and add new function to encode more calldata in the future.