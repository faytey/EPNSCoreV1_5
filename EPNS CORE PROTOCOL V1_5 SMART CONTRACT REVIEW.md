![](https://pbs.twimg.com/amplify_video_thumb/1574757092522864641/img/JvTLiQXi4HypQ7EW?format=jpg&name=large)

# EPNS CORE PROTOCOL V1_5 SMART CONTRACT REVIEW

By: **Faith M. Roberts**

**APRIL 2023**

## INTRODUCTION

This is a project given as an assignment by Web3Bridge, a personal review of the Push Protocol, formally EPNS, this project reviews the code base of EPNSCoreV1_5. The EPNSCoreV1_5 is an upgrade to the EPNSCoreV1_0 with more code optimization, security and removal of redundant code, also state variables where all declared in a separate file called EPNSCoreStorageV1_5 resulting in a cleaner, clearer and shorter code base.

**Source:**
[Github](https://github.com/ethereum-push-notification-service/push-smart-contracts/blob/master/contracts/EPNSCore/EPNSCoreV1_5.sol)

---

**_DISCLAIMER_**
_The following review refrains from making any claims or guarantees regarding the effectiveness of the code, appropriateness of the business model, regulatory compliance of the business model, or any other declarations about the suitability of the contracts for a particular purpose, or their status regarding potential acquisition costs._

## OVERVIEW

> The Ethereum Push Notification Service (EPNS) founded by Harsh Rajat and Richa Joshi, is a middleware protocol with a decentralized design. It allows smart contracts, dApps, and conventional services to send notifications to selected wallet addresses that have opted-in for these notifications. By utilizing EPNS, users can earn passive income and incentives for receiving decentralized notifications.

## CONTRACT REVIEW

The EPNSCoreV1_5 contract is the core for the performance of channel creations on the push network and creation of channel Admin, storing and handling all other core channel functionalities.

### Solidity Version

```
pragma solidity >=0.6.0 <0.7.0;
```

The Solidity Version used in the preparation of this contract is from version **0.6.0** to **0.7.0**

### Imports

```
import "./EPNSCoreStorageV1_5.sol";
import "./EPNSCoreStorageV2.sol";
import "../interfaces/IPUSH.sol";
import "../interfaces/IUniswapV2Router.sol";
import "../interfaces/IEPNSCommV1.sol";

import "@openzeppelin/contracts/utils/Strings.sol";
import "@openzeppelin/contracts/math/SafeMath.sol";
import "@openzeppelin/contracts/token/ERC20/IERC20.sol";
import "@openzeppelin/contracts/token/ERC20/SafeERC20.sol";
import "@openzeppelin/contracts-upgradeable/utils/PausableUpgradeable.sol";
```

The dependencies needed for the contract are first imported from the various files where there are constructed.

- The EPNSCoreStorageV1_5.sol declares all the state variables including the user defined types Enum (ChannelType and ChannelAction), Struct (Channel) and Mappings (channels, channelById, channelNotifSettings).
- The EPNSCoreStorageV2.sol stores the state variables that entails hashing functions for getting signatures and verifying addresses.
- The IPUSH.sol is the interface for the PUSH contract, to ensure channel creators can pay the minimum channel creation fees using PUSH tokens as opposed to DAI in V1_0.
- The IUniswapV2Router.sol is the interface that helps handle the swapExactTokensForTokens function.
- The IEPNSCommV1.sol is the interface of the communications contract version 1 that handles the flow of communications directly to users.
- The Openzeppelin Strings.sol imports the safe contract that handles strings.
- The Openzeppelin SafeMath.sol imports the safe math contract that handles mathematical computations.
- The Openzeppelin IERC20.sol imports the ERC20 interface that handles tokens.
- The Openzeppelin SafeERC20.sol imports the safeERC20 that handles tokens.
- The Openzeppelin PausableUpgradeable.sol imports and uses the pause and unpause functions to handle the pausing and unpausing of the contract.

### Events

```
event UpdateChannel(address indexed channel, bytes identity);
    event RewardsClaimed(address indexed user, uint256 rewardAmount);
    event ChannelVerified(address indexed channel, address indexed verifier);
    event ChannelVerificationRevoked(
        address indexed channel,
        address indexed revoker
    );

    event DeactivateChannel(
        address indexed channel,
        uint256 indexed amountRefunded
    );
    event ReactivateChannel(
        address indexed channel,
        uint256 indexed amountDeposited
    );
    event ChannelBlocked(address indexed channel);
    event AddChannel(
        address indexed channel,
        ChannelType indexed channelType,
        bytes identity
    );
    event ChannelNotifcationSettingsAdded(
        address _channel,
        uint256 totalNotifOptions,
        string _notifSettings,
        string _notifDescription
    );
    event AddSubGraph(address indexed channel, bytes _subGraphData);
    event TimeBoundChannelDestroyed(
        address indexed channel,
        uint256 indexed amountRefunded
    );
    event ChannelOwnershipTransfer(
        address indexed channel,
        address indexed newOwner
    );
```

The events to be emitted when various functions are called are defined here with the parameters holding the pieces of information to be displayed.

- The UpdateChannel event emits when an update has been done on a particular channel, updates where not possible till recently in order to prevent fraud and scam accounts, but strict regulations are in place for any channel that gets updated to prevent fraudulent activity.
- The RewardsClaimed event emits when a user has received a reward.
- The ChannelVerified event emits when a channel has been successfully verified.
- The ChannelVerificationRevoked event emits when a channel's verification has been revoked due to channel deactivation or inappropriateness.
- The DeactivateChannel event emits when a channel has been deactivated successfully.
- The ReactivateChannel event emits when a deactivated channel has been reactivated successfully.
- The ChannelBlocked event emits when a channel gets blocked due to fraudulent or inappropriate behaviour by channel creator.
- The ChannelNotifcationSettingsAdded event emits when a user in a channel has selected the type of notifications they want to get.
- The AddSubGraph event emits when the subgraph has been added to the channel to enable users to read notifications from the graph.
- The TimeBoundChannelDestroyed event emits when a time has been set for the expiry of a channel.
- The ChannelOwnershipTransfer event emits when a channel's owner has been updated.

### Modifiers

The modifiers to be used in the contract are declared

```
modifier onlyPushChannelAdmin() {
        require(
            msg.sender == pushChannelAdmin,
            "EPNSCoreV1_5::onlyPushChannelAdmin: Caller not pushChannelAdmin"
        );
        _;
    }
```

The onlyPushChannelAdmin modifier checks and requires that the caller of a function with the modifier mark is the push channel admin.

```
 modifier onlyGovernance() {
        require(
            msg.sender == governance,
            "EPNSCoreV1_5::onlyGovernance: Caller not Governance"
        );
        _;
    }
```

The onlyGovernance modifier checks and requires that the caller of a function with the modifier mark is the governance address.

```
 modifier onlyInactiveChannels(address _channel) {
        require(
            channels[_channel].channelState == 0,
            "EPNSCoreV1_5::onlyInactiveChannels: Channel already Activated"
        );
        _;
    }
```

The onlyInactiveChannels modifier checks and requires that the caller of a function with the modifier mark are channels which are not yet activated.

```
    modifier onlyActivatedChannels(address _channel) {
        require(
            channels[_channel].channelState == 1,
            "EPNSCoreV1_5::onlyActivatedChannels: Channel Deactivated, Blocked or Does Not Exist"
        );
        _;
    }
```

The onlyActivatedChannels modifier checks and requires that the caller of a function with the modifier mark are channels which are currently activated.

```
modifier onlyDeactivatedChannels(address _channel) {
        require(
            channels[_channel].channelState == 2,
            "EPNSCoreV1_5::onlyDeactivatedChannels: Channel is not Deactivated Yet"
        );
        _;
    }
```

The onlyDeactivatedChannels modifier checks and requires that the caller of a function with the modifier mark are channels which are currently deactivated.

```
   modifier onlyUnblockedChannels(address _channel) {
        require(
            ((channels[_channel].channelState != 3) &&
                (channels[_channel].channelState != 0)),
            "EPNSCoreV1_5::onlyUnblockedChannels: Channel is BLOCKED Already or Not Activated Yet"
        );
        _;
    }
```

The onlyUnblockedChannels modifier checks and requires that the caller of a function with the modifier mark are channels which are currently not blocked.

```
    modifier onlyChannelOwner(address _channel) {
        require(
            ((channels[_channel].channelState == 1 && msg.sender == _channel) ||
                (msg.sender == pushChannelAdmin && _channel == address(0x0))),
            "EPNSCoreV1_5::onlyChannelOwner: Channel not Exists or Invalid Channel Owner"
        );
        _;
    }
```

The onlyChannelOwner modifier checks and requires that the caller of a function with the modifier mark is the channel owner.

```
 modifier onlyUserAllowedChannelType(ChannelType _channelType) {
        require(
            (_channelType == ChannelType.InterestBearingOpen ||
                _channelType == ChannelType.InterestBearingMutual ||
                _channelType == ChannelType.TimeBound ||
                _channelType == ChannelType.TokenGaited),
            "EPNSCoreV1_5::onlyUserAllowedChannelType: Channel Type Invalid"
        );

        _;
    }
```

The onlyUserAllowedChannelType modifier checks and requires that the caller of a function with the modifier mark are channels which are designed for only users.

### Functions

The functions that sets and handles the contract logic are defined.

#### 1. Initializer

```
    function initialize(
        address _pushChannelAdmin,
        address _pushTokenAddress,
        address _wethAddress,
        address _uniswapRouterAddress,
        address _lendingPoolProviderAddress,
        address _daiAddress,
        address _aDaiAddress,
        uint256 _referralCode
    ) public initializer returns (bool success) {
        // setup addresses
        pushChannelAdmin = _pushChannelAdmin;
        governance = _pushChannelAdmin; // Will be changed on-Chain governance Address later
        daiAddress = _daiAddress;
        aDaiAddress = _aDaiAddress;
        WETH_ADDRESS = _wethAddress;
        REFERRAL_CODE = _referralCode;
        PUSH_TOKEN_ADDRESS = _pushTokenAddress;
        UNISWAP_V2_ROUTER = _uniswapRouterAddress;
        lendingPoolProviderAddress = _lendingPoolProviderAddress;

        FEE_AMOUNT = 10 ether; // PUSH Amount that will be charged as Protocol Pool Fees
        MIN_POOL_CONTRIBUTION = 50 ether; // Channel's poolContribution should never go below MIN_POOL_CONTRIBUTION
        ADD_CHANNEL_MIN_FEES = 50 ether; // can never be below MIN_POOL_CONTRIBUTION

        ADJUST_FOR_FLOAT = 10**7;
        groupLastUpdate = block.number;
        groupNormalizedWeight = ADJUST_FOR_FLOAT; // Always Starts with 1 * ADJUST FOR FLOAT

        // Create Channel
        success = true;
    }
```

The Initialize function acts as the constructor and handles all the logic for the creation of the contract as it takes in the following parameters;

- Address of the Channel Admin
- Address of the PUSH token for channel creation fees
- Address of Wrapped ether
- Address of UniswapV2Router
- Address of lending pool provider
- Address of DAI token
- Address of aDai token
- Referral code

After which the channel is successfully created.

#### 2. Setter & Helper Functions

```
function addSubGraph(bytes calldata _subGraphData)
    external
    onlyActivatedChannels(msg.sender)
{
    emit AddSubGraph(msg.sender, _subGraphData);
}
```

The function addSubGraph helps to read notifications from the graph protocol

```
function updateWETHAddress(address _newAddress)
    external
    onlyPushChannelAdmin
{
    WETH_ADDRESS = _newAddress;
}
```

The function updateWETHAddress helps to set the address of Wrapped ether the contract will be calling.

```
function updateUniswapRouterAddress(address _newAddress)
    external
    onlyPushChannelAdmin
{
    UNISWAP_V2_ROUTER = _newAddress;
}
```

The function updateUniswapRouterAddress helps to set the address of uniswap router the contract will be calling.

```
function setEpnsCommunicatorAddress(address _commAddress)
    external
    onlyPushChannelAdmin
{
    epnsCommunicator = _commAddress;
}
```

The function setEPNSCommunicatorAddress helps to set the address of EPNSComms the contract will be calling.

```
function setGovernanceAddress(address _governanceAddress)
    external
    onlyPushChannelAdmin
{
    governance = _governanceAddress;
}
```

The function setGovernanceAddress contains the logic to set the governance address of a channel handled by only the push channel admin.

```
function setMigrationComplete() external onlyPushChannelAdmin {
    isMigrationComplete = true;
}

```

The function setMigrationComplete contains the logic that sets the migration of a channel to true, when the channel has moved from one network to another.

```
function setFeeAmount(uint256 _newFees) external onlyGovernance {
    require(
        _newFees > 0 && _newFees < ADD_CHANNEL_MIN_FEES,
        "EPNSCoreV1_5::setFeeAmount: Fee amount must be greater than ZERO"
    );
    FEE_AMOUNT = _newFees;
}

```

The function setFeeAmount sets the minimum amount of push tokens to deposit, the fees cannot be less than the ADD_CHANNEL_MIN_FEES.

```
function setMinPoolContribution(uint256 _newAmount) external onlyGovernance {
    require(
        _newAmount > 0,
        "EPNSCoreV1_5::setMinPoolContribution: New Pool Contribution amount must be greater than ZERO"
    );
    MIN_POOL_CONTRIBUTION = _newAmount;
}

```

The function setMinPoolContribution sets the minimum amount of tokens to deposit into the pool as channel pool contribution.

```
function pauseContract() external onlyGovernance {
    _pause();
}
```

The function pauseContract contains the openzeppelin helper function \_pause, that pauses the channel.

```
function unPauseContract() external onlyGovernance {
    _unpause();
}
```

The function unPauseContract contains the openzeppelin helper function \_unpause, that unpauses the channel.

```
function setMinChannelCreationFees(uint256 _newFees)
external
onlyGovernance
{
require(
    _newFees >= MIN_POOL_CONTRIBUTION,
    "EPNSCoreV1_5::setMinChannelCreationFees: Fees should be greater than MIN_POOL_CONTRIBUTION"
);
ADD_CHANNEL_MIN_FEES = _newFees;
}
```

The function setMinChannelCreationFees sets the minimum amount of push tokens to deposit while creating a channel, the fees cannot be less than the MIN_POOL_CONTRIBUTION.

```
function transferPushChannelAdminControl(address _newAdmin)
    external
    onlyPushChannelAdmin
{
    require(
        _newAdmin != address(0),
        "EPNSCoreV1_5::transferPushChannelAdminControl: Invalid Address"
    );
    require(
        _newAdmin != pushChannelAdmin,
        "EPNSCoreV1_5::transferPushChannelAdminControl: Admin address is same"
    );
    pushChannelAdmin = _newAdmin;
}
```

The function transferPushChannelAdminControl sets the push channel admin of a particular channel to another specified by only the push channel owner.

#### 3. Channel Functionalities

##### 1. getChannelState

```
function getChannelState(address _channel)
        external
        view
        returns (uint256 state)
    {
        state = channels[_channel].channelState;
    }
```

The function getChannelState returns the state of the channel if active, not active or deactivated.

---

##### 2. updateChannelMeta

```
function updateChannelMeta(
        address _channel,
        bytes calldata _newIdentity,
        uint256 _amount
    ) external whenNotPaused onlyChannelOwner(_channel) {
        uint256 updateCounter = channelUpdateCounter[_channel].add(1);
        uint256 requiredFees = ADD_CHANNEL_MIN_FEES.mul(updateCounter);

        require(
            _amount >= requiredFees,
            "EPNSCoreV1_5::updateChannelMeta: Insufficient Deposit Amount"
        );
        PROTOCOL_POOL_FEES = PROTOCOL_POOL_FEES.add(_amount);
        channelUpdateCounter[_channel] = updateCounter;
        channels[_channel].channelUpdateBlock = block.number;

        IERC20(PUSH_TOKEN_ADDRESS).safeTransferFrom(
            _channel,
            address(this),
            _amount
        );
        emit UpdateChannel(_channel, _newIdentity);
    }
```

The function updateChannelMeta sets the logic that handles the update of channel information by the admin e.g. name of channel, description, logo etc.

---

##### 3. createChannelWithPUSH

```
function createChannelWithPUSH(
        ChannelType _channelType,
        bytes calldata _identity,
        uint256 _amount,
        uint256 _channelExpiryTime
    )
        external
        whenNotPaused
        onlyInactiveChannels(msg.sender)
        onlyUserAllowedChannelType(_channelType)
    {
        require(
            _amount >= ADD_CHANNEL_MIN_FEES,
            "EPNSCoreV1_5::_createChannelWithPUSH: Insufficient Deposit Amount"
        );
        emit AddChannel(msg.sender, _channelType, _identity);

        IERC20(PUSH_TOKEN_ADDRESS).safeTransferFrom(
            msg.sender,
            address(this),
            _amount
        );
        _createChannel(msg.sender, _channelType, _amount, _channelExpiryTime);
    }
```

The function createChannelWithPUSH handles the logic of channel creation to ensure that subscribers create channel with atleast the minimum fees specified.

---

##### 4. migrateChannelData

```
function migrateChannelData(
        uint256 _startIndex,
        uint256 _endIndex,
        address[] calldata _channelAddresses,
        ChannelType[] calldata _channelTypeList,
        bytes[] calldata _identityList,
        uint256[] calldata _amountList,
        uint256[] calldata _channelExpiryTime
    ) external onlyPushChannelAdmin returns (bool) {
        require(
            !isMigrationComplete,
            "EPNSCoreV1_5::migrateChannelData: Migration is already done"
        );

        require(
            (_channelAddresses.length == _channelTypeList.length) &&
                (_channelAddresses.length == _identityList.length) &&
                (_channelAddresses.length == _amountList.length) &&
                (_channelAddresses.length == _channelExpiryTime.length),
            "EPNSCoreV1_5::migrateChannelData: Unequal Arrays passed as Argument"
        );

        for (uint256 i = _startIndex; i < _endIndex; i++) {
            if (channels[_channelAddresses[i]].channelState != 0) {
                continue;
            } else {
                IERC20(PUSH_TOKEN_ADDRESS).safeTransferFrom(
                    msg.sender,
                    address(this),
                    _amountList[i]
                );
                emit AddChannel(
                    _channelAddresses[i],
                    _channelTypeList[i],
                    _identityList[i]
                );
                _createChannel(
                    _channelAddresses[i],
                    _channelTypeList[i],
                    _amountList[i],
                    _channelExpiryTime[i]
                );
            }
        }
        return true;
    }
```

The function migrateChannelData migrates a channel from one network to another with the same channel information present.

---

##### 5. \_createChannel

```
function _createChannel(
        address _channel,
        ChannelType _channelType,
        uint256 _amountDeposited,
        uint256 _channelExpiryTime
    ) private {
        uint256 poolFeeAmount = FEE_AMOUNT;
        uint256 poolFundAmount = _amountDeposited.sub(poolFeeAmount);
        //store funds in pool_funds & pool_fees
        CHANNEL_POOL_FUNDS = CHANNEL_POOL_FUNDS.add(poolFundAmount);
        PROTOCOL_POOL_FEES = PROTOCOL_POOL_FEES.add(poolFeeAmount);

        // Calculate channel weight
        uint256 _channelWeight = poolFundAmount.mul(ADJUST_FOR_FLOAT).div(
            MIN_POOL_CONTRIBUTION
        );
        // Next create the channel and mark user as channellized
        channels[_channel].channelState = 1;
        channels[_channel].poolContribution = poolFundAmount;
        channels[_channel].channelType = _channelType;
        channels[_channel].channelStartBlock = block.number;
        channels[_channel].channelUpdateBlock = block.number;
        channels[_channel].channelWeight = _channelWeight;
        // Add to map of addresses and increment channel count
        uint256 _channelsCount = channelsCount;
        channelById[_channelsCount] = _channel;
        channelsCount = _channelsCount.add(1);

        if (_channelType == ChannelType.TimeBound) {
            require(
                _channelExpiryTime > block.timestamp,
                "EPNSCoreV1_5::createChannel: Invalid channelExpiryTime"
            );
            channels[_channel].expiryTime = _channelExpiryTime;
        }

        // Subscribe them to their own channel as well
        address _epnsCommunicator = epnsCommunicator;
        if (_channel != pushChannelAdmin) {
            IEPNSCommV1(_epnsCommunicator).subscribeViaCore(_channel, _channel);
        }

        // All Channels are subscribed to EPNS Alerter as well, unless it's the EPNS Alerter channel iteself
        if (_channel != address(0x0)) {
            IEPNSCommV1(_epnsCommunicator).subscribeViaCore(
                address(0x0),
                _channel
            );
            IEPNSCommV1(_epnsCommunicator).subscribeViaCore(
                _channel,
                pushChannelAdmin
            );
        }
    }
```

The function \_createChannel is a private function which handles the logic of channel creation by users.

---

##### 6. destroyTimeBoundChannel

```
function destroyTimeBoundChannel(address _channelAddress)
        external
        whenNotPaused
        onlyActivatedChannels(_channelAddress)
    {
        Channel memory channelData = channels[_channelAddress];

        require(
            channelData.channelType == ChannelType.TimeBound,
            "EPNSCoreV1_5::destroyTimeBoundChannel: Channel is not TIME BOUND"
        );
        require(
            (msg.sender == _channelAddress &&
                channelData.expiryTime < block.timestamp) ||
                (msg.sender == pushChannelAdmin &&
                    channelData.expiryTime.add(14 days) < block.timestamp),
            "EPNSCoreV1_5::destroyTimeBoundChannel: Invalid Caller or Channel has not Expired Yet"
        );
        uint256 totalRefundableAmount = channelData.poolContribution;

        if (msg.sender != pushChannelAdmin) {
            CHANNEL_POOL_FUNDS = CHANNEL_POOL_FUNDS.sub(totalRefundableAmount);
            IERC20(PUSH_TOKEN_ADDRESS).safeTransfer(
                msg.sender,
                totalRefundableAmount
            );
        } else {
            CHANNEL_POOL_FUNDS = CHANNEL_POOL_FUNDS.sub(totalRefundableAmount);
            PROTOCOL_POOL_FEES = PROTOCOL_POOL_FEES.add(totalRefundableAmount);
        }
        // Unsubscribing from imperative Channels
        address _epnsCommunicator = epnsCommunicator;
        IEPNSCommV1(_epnsCommunicator).unSubscribeViaCore(
            address(0x0),
            _channelAddress
        );
        IEPNSCommV1(_epnsCommunicator).unSubscribeViaCore(
            _channelAddress,
            _channelAddress
        );
        IEPNSCommV1(_epnsCommunicator).unSubscribeViaCore(
            _channelAddress,
            pushChannelAdmin
        );
        // Decrement Channel Count and Delete Channel Completely
        channelsCount = channelsCount.sub(1);
        delete channels[_channelAddress];

        emit TimeBoundChannelDestroyed(msg.sender, totalRefundableAmount);
    }
```

The function destroyTimeBoundChannel is called upon to destroy the channel when it's reached it's expiry time

---

##### 7. createChannelSettings

```
function createChannelSettings(
        uint256 _notifOptions,
        string calldata _notifSettings,
        string calldata _notifDescription,
        uint256 _amountDeposited
    ) external onlyActivatedChannels(msg.sender) {
        require(
            _amountDeposited >= ADD_CHANNEL_MIN_FEES,
            "EPNSCoreV1_5::createChannelSettings: Insufficient Funds Passed"
        );

        string memory notifSetting = string(
            abi.encodePacked(
                Strings.toString(_notifOptions),
                "+",
                _notifSettings
            )
        );
        channelNotifSettings[msg.sender] = notifSetting;

        PROTOCOL_POOL_FEES = PROTOCOL_POOL_FEES.add(_amountDeposited);
        IERC20(PUSH_TOKEN_ADDRESS).safeTransferFrom(
            msg.sender,
            address(this),
            _amountDeposited
        );
        emit ChannelNotifcationSettingsAdded(
            msg.sender,
            _notifOptions,
            notifSetting,
            _notifDescription
        );
    }
```

The function createChannelSettings creates the channel settings that would be available to users of the particular channel.

---

##### 8. deactivateChannel

```
function deactivateChannel()
        external
        whenNotPaused
        onlyActivatedChannels(msg.sender)
    {
        Channel storage channelData = channels[msg.sender];

        uint256 minPoolContribution = MIN_POOL_CONTRIBUTION;
        uint256 totalRefundableAmount = channelData.poolContribution.sub(
            minPoolContribution
        );

        uint256 _newChannelWeight = minPoolContribution
            .mul(ADJUST_FOR_FLOAT)
            .div(minPoolContribution);

        channelData.channelState = 2;
        CHANNEL_POOL_FUNDS = CHANNEL_POOL_FUNDS.sub(totalRefundableAmount);
        channelData.channelWeight = _newChannelWeight;
        channelData.poolContribution = minPoolContribution;

        IERC20(PUSH_TOKEN_ADDRESS).safeTransfer(
            msg.sender,
            totalRefundableAmount
        );

        emit DeactivateChannel(msg.sender, totalRefundableAmount);
    }
```

The deactivateChannel function contains the logic that deactivates a channel after which transfer out the tokens held within the contract to the admin.

---

##### 9. reactivateChannel

```
function reactivateChannel(uint256 _amount)
        external
        whenNotPaused
        onlyDeactivatedChannels(msg.sender)
    {
        require(
            _amount >= ADD_CHANNEL_MIN_FEES,
            "EPNSCoreV1_5::reactivateChannel: Insufficient Funds Passed for Channel Reactivation"
        );

        IERC20(PUSH_TOKEN_ADDRESS).safeTransferFrom(
            msg.sender,
            address(this),
            _amount
        );
        uint256 poolFeeAmount = FEE_AMOUNT;
        uint256 poolFundAmount = _amount.sub(poolFeeAmount);
        //store funds in pool_funds & pool_fees
        CHANNEL_POOL_FUNDS = CHANNEL_POOL_FUNDS.add(poolFundAmount);
        PROTOCOL_POOL_FEES = PROTOCOL_POOL_FEES.add(poolFeeAmount);

        Channel storage channelData = channels[msg.sender];

        uint256 _newPoolContribution = channelData.poolContribution.add(
            poolFundAmount
        );
        uint256 _newChannelWeight = _newPoolContribution
            .mul(ADJUST_FOR_FLOAT)
            .div(MIN_POOL_CONTRIBUTION);

        channelData.channelState = 1;
        channelData.poolContribution = _newPoolContribution;
        channelData.channelWeight = _newChannelWeight;

        emit ReactivateChannel(msg.sender, _amount);
    }
```

The function reactivateChannel is called upon when a deactivated account needs reactivation and has it's reactivation fees present

---

##### 10. blockChannel

```
function blockChannel(address _channelAddress)
        external
        whenNotPaused
        onlyPushChannelAdmin
        onlyUnblockedChannels(_channelAddress)
    {
        uint256 minPoolContribution = MIN_POOL_CONTRIBUTION;
        Channel storage channelData = channels[_channelAddress];
        // add channel's currentPoolContribution to PoolFees - (no refunds if Channel is blocked)
        // Decrease CHANNEL_POOL_FUNDS by currentPoolContribution
        uint256 currentPoolContribution = channelData.poolContribution.sub(
            minPoolContribution
        );
        CHANNEL_POOL_FUNDS = CHANNEL_POOL_FUNDS.sub(currentPoolContribution);
        PROTOCOL_POOL_FEES = PROTOCOL_POOL_FEES.add(currentPoolContribution);

        uint256 _newChannelWeight = minPoolContribution
            .mul(ADJUST_FOR_FLOAT)
            .div(minPoolContribution);

        channelsCount = channelsCount.sub(1);
        channelData.channelState = 3;
        channelData.channelWeight = _newChannelWeight;
        channelData.channelUpdateBlock = block.number;
        channelData.poolContribution = minPoolContribution;

        emit ChannelBlocked(_channelAddress);
    }
```

The blockChannel function handles the logic to block channels who act maliciously.

---

##### 11. transferChannelOwnership

```
function transferChannelOwnership(
        address _channelAddress,
        address _newChannelAddress,
        uint256 _amountDeposited
    ) external whenNotPaused onlyActivatedChannels(msg.sender) returns (bool) {
        require(
            _newChannelAddress != address(0) &&
                channels[_newChannelAddress].channelState == 0,
            "EPNSCoreV1_5::transferChannelOwnership: Invalid address for new channel owner"
        );
        require(
            _amountDeposited >= ADD_CHANNEL_MIN_FEES,
            "EPNSCoreV1_5::transferChannelOwnership: Insufficient Funds Passed for Ownership Transfer Reactivation"
        );
        IERC20(PUSH_TOKEN_ADDRESS).safeTransferFrom(
            _channelAddress,
            address(this),
            _amountDeposited
        );

        PROTOCOL_POOL_FEES = PROTOCOL_POOL_FEES.add(_amountDeposited);
        Channel memory channelData = channels[_channelAddress];
        channels[_newChannelAddress] = channelData;

        // Subscribe newChannelOwner address to important channels
        address _epnsCommunicator = epnsCommunicator;
        IEPNSCommV1(_epnsCommunicator).subscribeViaCore(
            _newChannelAddress,
            _newChannelAddress
        );

        IEPNSCommV1(_epnsCommunicator).subscribeViaCore(
            address(0x0),
            _newChannelAddress
        );
        IEPNSCommV1(_epnsCommunicator).subscribeViaCore(
            _newChannelAddress,
            pushChannelAdmin
        );

        // Unsubscribing pushChannelAdmin from old Channel
        IEPNSCommV1(_epnsCommunicator).unSubscribeViaCore(
            _channelAddress,
            pushChannelAdmin
        );

        delete channels[_channelAddress];
        emit ChannelOwnershipTransfer(_channelAddress, _newChannelAddress);
        return true;
    }
```

This function transferChannelOwnership contains the logic for transferring the ownership of a particular channel to another address callable by only admin

---

##### 12. getChannelVerification

```
  function getChannelVerfication(address _channel)
        public
        view
        returns (uint8 verificationStatus)
    {
        address verifiedBy = channels[_channel].verifiedBy;
        bool logicComplete = false;

        // Check if it's primary verification
        if (
            verifiedBy == pushChannelAdmin ||
            _channel == address(0x0) ||
            _channel == pushChannelAdmin
        ) {
            // primary verification, mark and exit
            verificationStatus = 1;
        } else {
            // can be secondary verification or not verified, dig deeper
            while (!logicComplete) {
                if (verifiedBy == address(0x0)) {
                    verificationStatus = 0;
                    logicComplete = true;
                } else if (verifiedBy == pushChannelAdmin) {
                    verificationStatus = 2;
                    logicComplete = true;
                } else {
                    // Upper drill exists, go up
                    verifiedBy = channels[verifiedBy].verifiedBy;
                }
            }
        }
    }
```

The function getChannelVerification handles the logic that returns the verification status of the channel.

---

##### 13. batchVerification

```
   function batchVerification(
        uint256 _startIndex,
        uint256 _endIndex,
        address[] calldata _channelList
    ) external onlyPushChannelAdmin returns (bool) {
        for (uint256 i = _startIndex; i < _endIndex; i++) {
            verifyChannel(_channelList[i]);
        }
        return true;
    }
```

The function batchVerification handles the logic that returns the verification status of multiple channels.

---

##### 14. batchRevokeVerification

```
  function batchRevokeVerification(
        uint256 _startIndex,
        uint256 _endIndex,
        address[] calldata _channelList
    ) external onlyPushChannelAdmin returns (bool) {
        for (uint256 i = _startIndex; i < _endIndex; i++) {
            unverifyChannel(_channelList[i]);
        }
        return true;
    }
```

The function batchRevokeVerification handles the logic that revokes the verified status of multiple channels and sets their statuses to unverified.

---

##### 15. verifyChannel

```
 function verifyChannel(address _channel)
        public
        onlyActivatedChannels(_channel)
    {
        // Check if caller is verified first
        uint8 callerVerified = getChannelVerfication(msg.sender);
        require(
            callerVerified > 0,
            "EPNSCoreV1_5::verifyChannel: Caller is not verified"
        );

        // Check if channel is verified
        uint8 channelVerified = getChannelVerfication(_channel);
        require(
            channelVerified == 0 || msg.sender == pushChannelAdmin,
            "EPNSCoreV1_5::verifyChannel: Channel already verified"
        );

        // Verify channel
        channels[_channel].verifiedBy = msg.sender;

        // Emit event
        emit ChannelVerified(_channel, msg.sender);
    }
```

The function verifyChannel handles the logic that verifies a channel.

---

##### 16. unverifyChannel

```
    function unverifyChannel(address _channel) public {
        require(
            channels[_channel].verifiedBy == msg.sender ||
                msg.sender == pushChannelAdmin,
            "EPNSCoreV1_5::unverifyChannel: Only channel who verified this or Push Channel Admin can revoke"
        );

        // Unverify channel
        channels[_channel].verifiedBy = address(0x0);

        // Emit Event
        emit ChannelVerificationRevoked(_channel, msg.sender);
    }
```

The function unverifyChannel handles the logic that revokes the verified status of a channel.

---

##### 17. getChainId

```

    function getChainId() internal pure returns (uint256) {
        uint256 chainId;
        assembly {
            chainId := chainid()
        }
        return chainId;
    }
```

The function getChainId returns the chainId of the current blockchain network.

## OBSERVATIONS

Based of on the review I performed on this contract, i came across some minor informational fixes that would be great if fixed.

1. EPNSCoreStorageV1_5 line 17 "tokengaited" could be replaced with "tokengated".
2. EPNSCoreV1_5 line 22 IERC20 import could be removed.
3. EPNSCoreV1_5 line 23 SafeERC20 import could be removed.
4. EPNSCoreV1_5 line 33 IERC20 could be removed and replaced by IPUSH.
5. EPNSCoreV1_5 line 154 WETHAddress import could be removed as it is not used and the current accepted payment token is PUSH and also line 77 of EPNSCoreStorageV1_5 should get updated.
6. EPNSCoreV1_5 line 157 DaiAddress import could be removed as it is not used and the current accepted payment token is PUSH and also line 75 of EPNSCoreStorageV1_5 it should get updated.
7. EPNSCoreV1_5 line 158 aDaiAddress import could be removed as it is not used and the current accepted payment token is PUSH and also line 76 of EPNSCoreStorageV1_5 it should get updated.
8. EPNSCoreStorageV1_5 line 33 EPNS_CORE_V2 could be replaced with EPNS_CORE_V2

## RECOMMENDATIONS

I hereby recommend that the above observations be look into and considered. Also be implemented.
