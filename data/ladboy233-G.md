| Index | Title |
|-------|-------|
| G1     | No need to create new helper address every time when custom withdraw |
| G2     | Use merkle tree when doing option airdrop |
| G3     | Use merkle tree when vesting token |
| G4     | Consider deploy market helper contract only once |
| G5     | Consider add batch distribute token in twTap.sol |
| G6     | Tap token deployment code size can be reduced |

# No need to create new helper address every time when custom withdraw

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

In the current implementation, when doing option airdrop, the admin has to [run a large for loop](https://github.com/Tapioca-DAO/tap-token/blob/20a83b1d2d5577653610a6c3879dff9df4968345/contracts/option-airdrop/AirdropBroker.sol#L321) to set the allocated token amount for each address one by one

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

# Consider add batch distribute token in twTap.sol

There maybe multiple reward token in twTap.sol, but anyone who wants to distribute the reward token has to figure out the token id and calls the function below [for every reward token id](https://github.com/Tapioca-DAO/tap-token/blob/20a83b1d2d5577653610a6c3879dff9df4968345/contracts/governance/twTAP.sol#L463).

```
   function distributeReward(uint256 _rewardTokenId, uint256 _amount) external nonReentrant {
        if (lastProcessedWeek != currentWeek()) revert AdvanceWeekFirst();
        if (_amount == 0) revert NotValid();
        if (_rewardTokenId == 0) revert NotValid(); // @dev rewardTokens[0] is 0x0

```

A more gas efficient way is to support batch reward distrubtion so anyone can distribute multiple token reward in single transaction.

# For loop for index 0 address(0) can be skipped when claiming reward

When [claiming rewards in twTAP.sol](https://github.com/Tapioca-DAO/tap-token/blob/20a83b1d2d5577653610a6c3879dff9df4968345/contracts/governance/twTAP.sol#L559), the protocol loops over all reward token

```
 function _claimRewards(uint256 _tokenId, address _to) internal returns (uint256[] memory amounts_) {
        amounts_ = claimable(_tokenId);
        uint256 len = amounts_.length;
        unchecked {
            for (uint256 i; i < len; ++i) {
                uint256 amount = amounts_[i];
                if (amount > 0) {
                    claimed[_tokenId][i] += amount;
                    rewardTokens[i].safeTransfer(_to, amount);
                }
            }
        }
    }
```

If only if when amonut > 0, we trigger transfer

```
  rewardTokens[i].safeTransfer(_to, amount);
```

but the reward token first index 0 has address(0)

the first reward index will have no reward, so it is recommended that protocol skip index 0
when computing [claimable reward](https://github.com/Tapioca-DAO/tap-token/blob/20a83b1d2d5577653610a6c3879dff9df4968345/contracts/governance/twTAP.sol#L192) and when transferring reward.

```
    function claimable(uint256 _tokenId) public view returns (uint256[] memory) {

        // add change here;
        uint256 len = rewardTokens.length - 1;
        uint256[] memory result = new uint256[](len);

        Participation memory position = participants[_tokenId];
        uint256 votes;
        unchecked {
            // Math is safe: Types fit
            votes = uint256(position.tapAmount) * uint256(position.multiplier);
        }

        if (votes == 0) {
            return result;
        }

        // If the "last processed week" is behind the actual week, rewards
        // get processed as if it were earlier.

        uint256 week = lastProcessedWeek;
        if (week <= position.lastInactive) {
            return result;
        }
        if (position.lastActive < week) {
            week = position.lastActive;
        }

        WeekTotals storage cur = weekTotals[week];
        WeekTotals storage prev = weekTotals[position.lastInactive];

        // add check here and 
        uint256 i = 1;

        for (uint256 i; i < len;) {
```

So user do not have to pay the extra gas to run the for loop when computing claimable reward and claiming reward.

# Tap token deployment code size can be reduced.

The tap token needs to be deployed in multiple network, but only when in arbitrum the [minter related function is relevant](https://github.com/Tapioca-DAO/tap-token/blob/20a83b1d2d5577653610a6c3879dff9df4968345/contracts/tokens/TapToken.sol#L352)

```
function extractTAP(address _to, uint256 _amount) external onlyMinter whenNotPaused {
function emitForWeek() external onlyMinter returns (uint256) {
```

It is recommended to not adding these function to deployment code to reduce the code size if the token is deployed in network other than arbitrum.