# QA Report

## Low Risk Issues

### [L-01] toggleCampaignOperator() emits incorrect event state

The `toggleCampaignOperator()` function emits the previous state instead of the new state in the `CampaignOperatorToggled` event. After toggling the operator status with `campaignOperators[user][operator] = 1 - currentStatus`, the event emits `currentStatus == 0` which represents the old state, not the new toggled state.

**Code Location**: `contracts/DistributionCreator.sol:329-335`

```solidity
function toggleCampaignOperator(address user, address operator) external onlyUserOrGovernor(user) {
    uint256 currentStatus = campaignOperators[user][operator];
    campaignOperators[user][operator] = 1 - currentStatus;
    // BUG: Emitting the previous state, not the new state
    emit CampaignOperatorToggled(user, operator, currentStatus == 0);
}
```

When `currentStatus` is 1 (operator was enabled):
- New state becomes 0 (operator disabled)
- Event emits `currentStatus == 0` which is `false`
- But the operator was actually **disabled**, so event should emit `false`

When `currentStatus` is 0 (operator was disabled):
- New state becomes 1 (operator enabled)
- Event emits `currentStatus == 0` which is `true`
- Operator was **enabled**, so event correctly shows `true`

The logic accidentally works correctly when enabling (0→1) but fails when disabling (1→0).

**Impact**

Off-chain monitoring tools and indexers will incorrectly track operator status changes. When an operator is disabled, the event will show `isWhitelisted = false` (correct by coincidence), but when re-enabled from a disabled state, the event tracking becomes permanently misaligned.

**Recommended mitigation steps**

Emit the new state after the toggle:

```solidity
function toggleCampaignOperator(address user, address operator) external onlyUserOrGovernor(user) {
    uint256 currentStatus = campaignOperators[user][operator];
    uint256 newStatus = 1 - currentStatus;
    campaignOperators[user][operator] = newStatus;
    emit CampaignOperatorToggled(user, operator, newStatus == 1);
}
```

---

### [L-02] campaign() getter returns original parameters when overrides exist, causing confusion

The `campaign()` function explicitly returns original campaign parameters even when `campaignOverrides[_campaignId]` has been set, as documented on line 354: "Returns original parameters even if the campaign has been overridden."

**Code Location**: `contracts/DistributionCreator.sol:356-358`

```solidity
function campaign(bytes32 _campaignId) public view returns (CampaignParameters memory) {
    return campaignList[campaignLookup(_campaignId)];
}
```

While `campaignOverrides` mapping stores modified parameters, there is no getter function that returns the effective (overridden) parameters. External integrators calling `campaign()` will receive stale parameters that don't reflect the actual active campaign configuration.

**Impact**

Off-chain systems, APIs, and user interfaces that call `campaign()` to display campaign information will show incorrect data when overrides are active. Users may make decisions based on outdated campaign parameters (start time, duration, campaign data), leading to confusion about campaign status and reward distribution timing.

**Recommended mitigation steps**

Add a getter function that returns the effective campaign parameters:

```solidity
/// @notice Returns the effective campaign parameters (with overrides applied)
function getEffectiveCampaign(bytes32 _campaignId) public view returns (CampaignParameters memory) {
    CampaignParameters memory effectiveCampaign = campaignList[campaignLookup(_campaignId)];

    // Check if override exists
    if (campaignOverrides[_campaignId].campaignId != bytes32(0)) {
        // Override preserves: rewardToken, amount, creator, campaignId
        // Override can change: campaignType, startTimestamp, duration, campaignData
        CampaignParameters memory override = campaignOverrides[_campaignId];
        effectiveCampaign.campaignType = override.campaignType;
        effectiveCampaign.startTimestamp = override.startTimestamp;
        effectiveCampaign.duration = override.duration;
        effectiveCampaign.campaignData = override.campaignData;
    }

    return effectiveCampaign;
}
```

Alternatively, rename the existing `campaign()` to `getOriginalCampaign()` and make the override-aware version the default `campaign()`.

---

### [L-03] overrideCampaign() documentation incomplete regarding mutable parameters

The `overrideCampaign()` function documentation states "Can only update startTimestamp if the campaign has not yet started" (line 232), but the implementation actually allows modifying multiple parameters: `campaignType`, `startTimestamp`, `duration`, and `campaignData`.

**Code Location**: `contracts/DistributionCreator.sol:237-257`

```solidity
function overrideCampaign(bytes32 _campaignId, CampaignParameters memory newCampaign) external {
    CampaignParameters memory _campaign = campaign(_campaignId);
    _isValidOperator(_campaign.creator);
    if (
        newCampaign.rewardToken != _campaign.rewardToken ||
        newCampaign.amount != _campaign.amount ||
        (newCampaign.startTimestamp != _campaign.startTimestamp && block.timestamp > _campaign.startTimestamp) ||
        newCampaign.duration + _campaign.startTimestamp <= block.timestamp
    ) revert Errors.InvalidOverride();

    newCampaign.campaignId = _campaignId;
    newCampaign.creator = _campaign.creator;
    // The entire newCampaign is stored, including campaignType and campaignData
    campaignOverrides[_campaignId] = newCampaign;
    campaignOverridesTimestamp[_campaignId].push(block.timestamp);
    emit CampaignOverride(_campaignId, newCampaign);
}
```

The function only validates:
- `rewardToken` cannot change
- `amount` cannot change
- `startTimestamp` can only change if campaign hasn't started
- New end time must be in the future

But allows changes to:
- `campaignType`
- `duration`
- `campaignData`

This creates ambiguity about which fields are actually mutable.

**Impact**

Campaign creators may be surprised that they can change `campaignType` and `campaignData` after creation, potentially allowing them to redirect campaigns to different pools or change reward distribution rules. While this may be intentional flexibility, the incomplete documentation obscures the actual behavior and security boundaries.

**Recommended mitigation steps**

Update the natspec documentation to explicitly list all mutable parameters:

```solidity
/// @notice Updates parameters of an existing campaign while preserving core immutable fields
/// @param _campaignId ID of the campaign to override
/// @param newCampaign New campaign parameters (some fields will be ignored or validated)
/// @dev Immutable fields: rewardToken, amount, creator
/// @dev Mutable fields: campaignType, startTimestamp (only before start), duration, campaignData
/// @dev Validation: startTimestamp can only change before campaign starts, end time must be in future
/// @dev The Merkl engine validates override correctness; invalid overrides are ignored
function overrideCampaign(bytes32 _campaignId, CampaignParameters memory newCampaign) external {
    // ... existing implementation
}
```

---

### [L-04] toggleOperator() in Distributor emits incorrect event state

Similar to the issue in `DistributionCreator`, the `toggleOperator()` function in `Distributor.sol` emits the previous state instead of the new state in the `OperatorToggled` event.

**Code Location**: `contracts/Distributor.sol:255-267`

```solidity
function toggleOperator(address user, address operator) external {
    if (user != msg.sender && !accessControlManager.isGovernorOrGuardian(msg.sender)) revert Errors.NotTrusted();
    uint256 oldValue = operators[user][operator];
    operators[user][operator] = 1 - oldValue;

    // BUG: Emitting the previous state, not the new state
    emit OperatorToggled(user, operator, oldValue == 0);
}
```

When `oldValue` is 1 (operator was enabled):
- New state becomes 0 (operator disabled)
- Event emits `oldValue == 0` which is `false`
- But the operator was actually **disabled**, so the event accidentally shows the correct final state

When `oldValue` is 0 (operator was disabled):
- New state becomes 1 (operator enabled)
- Event emits `oldValue == 0` which is `true`
- Operator was **enabled**, so event correctly shows `true`

The logic works by coincidence but is confusing and fragile.

**Impact**

Off-chain monitoring tools and indexers tracking operator authorization status will be misled. While the emitted value happens to match the final state in this case, the implementation is misleading and could cause maintenance issues if the toggle logic is ever modified.

**Recommended mitigation steps**

Emit the new state explicitly to make the logic clear:

```solidity
function toggleOperator(address user, address operator) external {
    if (user != msg.sender && !accessControlManager.isGovernorOrGuardian(msg.sender)) revert Errors.NotTrusted();
    uint256 oldValue = operators[user][operator];
    uint256 newValue = 1 - oldValue;
    operators[user][operator] = newValue;
    emit OperatorToggled(user, operator, newValue == 1);
}
```

---

### [L-05] ClaimRecipientUpdated event emits parameters in wrong order

The `ClaimRecipientUpdated` event is defined with parameters `(user, token, recipient)` but is emitted with the order `(user, recipient, token)`, causing indexed event parameters to be swapped.

**Code Location**:
- Event definition: `contracts/Distributor.sol:120`
- Event emission: `contracts/Distributor.sol:555`

```solidity
// Event definition (line 120)
event ClaimRecipientUpdated(address indexed user, address indexed token, address indexed recipient);

// Event emission in _setClaimRecipient() (line 555)
function _setClaimRecipient(address user, address recipient, address token) internal {
    claimRecipient[user][token] = recipient;
    // BUG: Wrong parameter order - should be (user, token, recipient)
    emit ClaimRecipientUpdated(user, recipient, token);
}
```

The event expects `(user, token, recipient)` but is called with `(user, recipient, token)`.

**Impact**

Off-chain indexers and monitoring systems that listen for `ClaimRecipientUpdated` events will record incorrect data:
- The `token` indexed field will contain the recipient address
- The `recipient` indexed field will contain the token address

This breaks event-based queries like "find all recipients for token X" or "find all tokens sent to recipient Y", corrupting historical event data and breaking integrations that rely on these indexed parameters.

**Recommended mitigation steps**

Correct the event emission order to match the event definition:

```solidity
function _setClaimRecipient(address user, address recipient, address token) internal {
    claimRecipient[user][token] = recipient;
    emit ClaimRecipientUpdated(user, token, recipient);
}
```

---

### [L-06] setFeeRecipient() allows zero address, breaking fee collection

The `setFeeRecipient()` function does not validate that the new fee recipient is not `address(0)`, which could accidentally disable fee collection if the governor mistakenly sets it to zero address.

**Code Location**: `contracts/DistributionCreator.sol:443-446`

```solidity
function setFeeRecipient(address _feeRecipient) external onlyGovernor {
    feeRecipient = _feeRecipient;
    emit FeeRecipientUpdated(_feeRecipient);
}
```

When `feeRecipient` is `address(0)`, the `_pullTokens()` function at line 614 will send fees to `address(this)` instead:

```solidity
if (fees > 0 && _feeRecipient != address(this)) IERC20(rewardToken).safeTransfer(_feeRecipient, fees);
```

**Impact**

If the governor accidentally sets `feeRecipient` to `address(0)`, all campaign fees will be sent to the `DistributionCreator` contract itself instead of the intended fee recipient. These fees would become stuck in the contract with no direct way to recover them without governance intervention.

**Recommended mitigation steps**

Add zero-address validation:

```solidity
function setFeeRecipient(address _feeRecipient) external onlyGovernor {
    if (_feeRecipient == address(0)) revert Errors.ZeroAddress();
    feeRecipient = _feeRecipient;
    emit FeeRecipientUpdated(_feeRecipient);
}
```

---

### [L-07] setUserFeeRebate() does not validate rebate cannot exceed BASE_9

The `setUserFeeRebate()` function allows setting a fee rebate to any value without checking if it exceeds `BASE_9` (100%), which could cause arithmetic underflow when computing fees.

**Code Location**: `contracts/DistributionCreator.sol:486-489`

```solidity
function setUserFeeRebate(address user, uint256 userFeeRebate) external onlyGovernorOrGuardian {
    feeRebate[user] = userFeeRebate;
    emit FeeRebateUpdated(user, userFeeRebate);
}
```

If `feeRebate[msg.sender]` exceeds `BASE_9`, the computation at line 636 would underflow:

```solidity
uint256 _fees = (baseFeesValue * (BASE_9 - feeRebate[msg.sender])) / BASE_9;
```

**Impact**

Setting a fee rebate larger than `BASE_9` would cause the fee calculation to revert with an arithmetic underflow, preventing any campaign creation by that user. While this requires governance/guardian error, the lack of validation makes it easy to misconfigure.

**Recommended mitigation steps**

Add bounds validation similar to `setFees()`:

```solidity
function setUserFeeRebate(address user, uint256 userFeeRebate) external onlyGovernorOrGuardian {
    if (userFeeRebate >= BASE_9) revert Errors.InvalidParam();
    feeRebate[user] = userFeeRebate;
    emit FeeRebateUpdated(user, userFeeRebate);
}
```

---

### [L-08] acceptConditions() missing event emission for off-chain tracking

The `acceptConditions()` function updates the `userSignatures` mapping but does not emit an event, making it difficult for off-chain systems to track when users accept new terms.

**Code Location**: `contracts/DistributionCreator.sol:224-226`

```solidity
function acceptConditions() external {
    userSignatures[msg.sender] = messageHash;
}
```

**Impact**

Off-chain monitoring systems cannot easily track which users have accepted the current terms and conditions. This makes it harder to maintain compliance records and verify that users have agreed to the latest terms before interacting with the protocol.

**Recommended mitigation steps**

Add an event emission:

```solidity
event ConditionsAccepted(address indexed user, bytes32 indexed messageHash);

function acceptConditions() external {
    userSignatures[msg.sender] = messageHash;
    emit ConditionsAccepted(msg.sender, messageHash);
}
```

---

### [L-09] setDisputePeriod() lacks validation for reasonable bounds

The `setDisputePeriod()` function allows setting the dispute period to any value including 0, which could either make disputes ineffective or create excessively long dispute windows.

**Code Location**: `contracts/Distributor.sol:395-398`

```solidity
function setDisputePeriod(uint48 _disputePeriod) external onlyGovernor {
    disputePeriod = uint48(_disputePeriod);
    emit DisputePeriodUpdated(_disputePeriod);
}
```

**Impact**

- If set to 0, the dispute mechanism becomes ineffective as Merkle roots would be immediately finalized without any dispute window
- If set to an extremely large value, rewards would be locked for an unreasonably long time before they can be claimed

**Recommended mitigation steps**

Add reasonable bounds validation:

```solidity
function setDisputePeriod(uint48 _disputePeriod) external onlyGovernor {
    // Minimum 1 epoch, maximum 100 epochs (adjust based on protocol needs)
    if (_disputePeriod == 0 || _disputePeriod > 100) revert Errors.InvalidParam();
    disputePeriod = uint48(_disputePeriod);
    emit DisputePeriodUpdated(_disputePeriod);
}
```

---

### [L-10] setRewardTokenMinAmounts() does not validate for zero addresses

The `setRewardTokenMinAmounts()` function does not check if any token address in the input array is `address(0)`, which could accidentally whitelist the zero address as a reward token.

**Code Location**: `contracts/DistributionCreator.sol:507-521`

```solidity
function setRewardTokenMinAmounts(address[] calldata tokens, uint256[] calldata amounts) external onlyGovernorOrGuardian {
    uint256 tokensLength = tokens.length;
    if (tokensLength != amounts.length) revert Errors.InvalidLengths();
    for (uint256 i; i < tokensLength; ) {
        uint256 amount = amounts[i];
        if (amount != 0 && rewardTokenMinAmounts[tokens[i]] == 0) rewardTokens.push(tokens[i]);
        rewardTokenMinAmounts[tokens[i]] = amount;
        emit RewardTokenMinimumAmountUpdated(tokens[i], amount);
        unchecked {
            ++i;
        }
    }
}
```

**Impact**

If `address(0)` is accidentally added to the reward tokens whitelist with a non-zero minimum amount, campaigns could be created with `address(0)` as the reward token, causing token transfers to fail and making those campaigns non-functional. This would waste gas and potentially lock campaign fees.

**Recommended mitigation steps**

Add zero-address validation in the loop:

```solidity
function setRewardTokenMinAmounts(address[] calldata tokens, uint256[] calldata amounts) external onlyGovernorOrGuardian {
    uint256 tokensLength = tokens.length;
    if (tokensLength != amounts.length) revert Errors.InvalidLengths();
    for (uint256 i; i < tokensLength; ) {
        address token = tokens[i];
        if (token == address(0)) revert Errors.ZeroAddress();
        uint256 amount = amounts[i];
        if (amount != 0 && rewardTokenMinAmounts[token] == 0) rewardTokens.push(token);
        rewardTokenMinAmounts[token] = amount;
        emit RewardTokenMinimumAmountUpdated(token, amount);
        unchecked {
            ++i;
        }
    }
}
```

---

### [L-11] createCampaigns() lacks error handling for batch operations

The `createCampaigns()` function creates multiple campaigns in a single transaction but does not handle partial failures. If any single campaign creation fails, the entire batch transaction reverts, causing all campaigns to fail even if others are valid.

**Code Location**: `contracts/DistributionCreator.sol:206-219`

```solidity
function createCampaigns(CampaignParameters[] memory campaigns) external nonReentrant hasSigned returns (bytes32[] memory) {
    uint256 campaignsLength = campaigns.length;
    bytes32[] memory campaignIds = new bytes32[](campaignsLength);
    for (uint256 i; i < campaignsLength; ) {
        campaignIds[i] = _createCampaign(campaigns[i]);
        unchecked {
            ++i;
        }
    }
    return campaignIds;
}
```

Campaign creation can fail for various reasons:
- Insufficient token balance for one campaign
- Reward token not whitelisted
- Amount below minimum threshold
- Invalid duration or parameters
- Campaign already exists

When any failure occurs, all campaigns in the batch fail.

**Impact**

Users attempting to create multiple campaigns must ensure all campaigns are perfectly valid, or the entire transaction reverts. This creates poor user experience and wastes gas when one invalid campaign prevents the creation of valid ones. Users must debug which campaign is failing and resubmit the entire batch.

**Recommended mitigation steps**

Add try-catch error handling to allow partial success and track which campaigns failed:

```solidity
function createCampaigns(CampaignParameters[] memory campaigns)
    external
    nonReentrant
    hasSigned
    returns (bytes32[] memory campaignIds, bool[] memory success)
{
    uint256 campaignsLength = campaigns.length;
    campaignIds = new bytes32[](campaignsLength);
    success = new bool[](campaignsLength);

    for (uint256 i; i < campaignsLength; ) {
        try this.createCampaignExternal(campaigns[i]) returns (bytes32 id) {
            campaignIds[i] = id;
            success[i] = true;
        } catch {
            // Campaign creation failed, continue with next one
            campaignIds[i] = bytes32(0);
            success[i] = false;
        }
        unchecked {
            ++i;
        }
    }
    return (campaignIds, success);
}

// External wrapper needed for try-catch
function createCampaignExternal(CampaignParameters memory campaign)
    external
    returns (bytes32)
{
    require(msg.sender == address(this), "Internal only");
    return _createCampaign(campaign);
}
```

Alternatively, add an event to log failed campaigns:

```solidity
event CampaignCreationFailed(uint256 indexed index, CampaignParameters campaign, string reason);

function createCampaigns(CampaignParameters[] memory campaigns)
    external
    nonReentrant
    hasSigned
    returns (bytes32[] memory)
{
    uint256 campaignsLength = campaigns.length;
    bytes32[] memory campaignIds = new bytes32[](campaignsLength);

    for (uint256 i; i < campaignsLength; ) {
        try this.createCampaignExternal(campaigns[i]) returns (bytes32 id) {
            campaignIds[i] = id;
        } catch Error(string memory reason) {
            emit CampaignCreationFailed(i, campaigns[i], reason);
            campaignIds[i] = bytes32(0);
        } catch {
            emit CampaignCreationFailed(i, campaigns[i], "Unknown error");
            campaignIds[i] = bytes32(0);
        }
        unchecked {
            ++i;
        }
    }
    return campaignIds;
}
```

---

## Non-Critical Issues

### [NC-01] Misleading validation message in overrideCampaign() about end timestamp calculation

In `overrideCampaign()`, the validation on line 248 checks `newCampaign.duration + _campaign.startTimestamp <= block.timestamp` with a comment "End timestamp should be in the future", but the calculation uses the **original** campaign's `startTimestamp`, not the potentially modified `newCampaign.startTimestamp`.

**Code Location**: `contracts/DistributionCreator.sol:246-248`

```solidity
(newCampaign.startTimestamp != _campaign.startTimestamp && block.timestamp > _campaign.startTimestamp) ||
// End timestamp should be in the future
newCampaign.duration + _campaign.startTimestamp <= block.timestamp
```

If a creator is updating both `startTimestamp` and `duration`, the validation uses `_campaign.startTimestamp` (old value) to validate the new duration, which could allow setting an end time in the past if:
- Old start time: 1000
- Current time: 2000
- New start time: 1500
- New duration: 400
- Validation checks: `400 + 1000 = 1400 <= 2000` ✓ (passes)
- Actual new end time: `1500 + 400 = 1900` (in the past)

**Impact**

The validation logic is inconsistent and could theoretically allow setting campaign end times in the past when modifying start time, though in practice the Merkl engine would likely ignore such campaigns.

**Recommended mitigation steps**

Use the new `startTimestamp` in the end time validation:

```solidity
function overrideCampaign(bytes32 _campaignId, CampaignParameters memory newCampaign) external {
    CampaignParameters memory _campaign = campaign(_campaignId);
    _isValidOperator(_campaign.creator);
    if (
        newCampaign.rewardToken != _campaign.rewardToken ||
        newCampaign.amount != _campaign.amount ||
        (newCampaign.startTimestamp != _campaign.startTimestamp && block.timestamp > _campaign.startTimestamp) ||
        // Use newCampaign.startTimestamp for end time validation
        newCampaign.startTimestamp + newCampaign.duration <= block.timestamp
    ) revert Errors.InvalidOverride();

    newCampaign.campaignId = _campaignId;
    newCampaign.creator = _campaign.creator;
    campaignOverrides[_campaignId] = newCampaign;
    campaignOverridesTimestamp[_campaignId].push(block.timestamp);
    emit CampaignOverride(_campaignId, newCampaign);
}
```

---

## Governance/Centralization Risk Issues

### [G-01] Epoch duration can be changed mid-operation without user notice

The `setEpochDuration()` function allows the governor to change the epoch duration at any time without a timelock, affecting claim timing and dispute periods.

**Code Location**: `contracts/Distributor.sol:350-353`

```solidity
function setEpochDuration(uint32 epochDuration) external onlyGovernor {
    emit EpochDurationSet(epochDuration);
    _setEpochDuration(epochDuration);
}
```

**Impact**

Changing epoch duration mid-operation affects:
- When users can claim rewards (epochs determine tree update timing)
- Dispute period lengths (measured in epochs)
- Tree validity periods

Users planning claims based on current epoch timing may find their assumptions invalidated. A malicious governor could extend epochs to delay claims or shorten them to rush through tree updates.

**Recommended mitigation steps**

Add a timelock and only allow changes at epoch boundaries:

```solidity
uint32 public pendingEpochDuration;
uint256 public epochDurationChangeTime;
uint256 public constant EPOCH_CHANGE_DELAY = 7 days;

function setPendingEpochDuration(uint32 newDuration) external onlyGovernor {
    if (newDuration < 1 hours || newDuration > 30 days) revert Errors.InvalidEpochDuration();
    pendingEpochDuration = newDuration;
    epochDurationChangeTime = block.timestamp + EPOCH_CHANGE_DELAY;
    emit PendingEpochDurationSet(newDuration, epochDurationChangeTime);
}

function applyEpochDurationChange() external onlyGovernor {
    if (block.timestamp < epochDurationChangeTime) revert Errors.TimelockNotExpired();
    if (pendingEpochDuration == 0) revert Errors.NoPendingChange();

    // Only apply at epoch boundary
    uint32 currentEpoch = uint32(block.timestamp) / getEpochDuration();
    uint32 nextEpochStart = (currentEpoch + 1) * getEpochDuration();
    if (block.timestamp < nextEpochStart) revert Errors.MustWaitForEpochBoundary();

    _setEpochDuration(pendingEpochDuration);
    pendingEpochDuration = 0;
}
```