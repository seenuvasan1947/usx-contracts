# USX Contracts - Supplementary Audit Findings
## Additional Files Review

**Audit Date:** November 21, 2025  
**Auditor:** AI Security Auditor  
**Status:** Supplementary to Main Audit Report

---

## Files Reviewed in This Supplement

1. **TreasuryStorage.sol** (182 lines) - Storage contract for Treasury
2. **ERC20Relayer.sol** (137 lines) - Bridge relayer for Scroll L2
3. **USXOFT.sol** (15 lines) - LayerZero OFT wrapper
4. **USXOFTAdaptor.sol** (14 lines) - LayerZero OFT adapter
5. **StakedUSXOFT.sol** (15 lines) - LayerZero OFT for sUSX
6. **StakedUSXOFTAdaptor.sol** (14 lines) - LayerZero OFT adapter for sUSX

---

## CRITICAL FINDINGS

### [CRITICAL-3] Bridge Relayer Can Drain All Tokens

**Contract:** `ERC20Relayer.sol`  
**Function:** `bridge()`  
**Severity:** CRITICAL  
**Status:** ⚠️ NEW FINDING

**Description:**
The `bridge()` function transfers the ENTIRE balance of tokens without any validation or limit. A compromised or malicious BRIDGE_ROLE holder can drain all tokens to an arbitrary L1 address.

**Code Location:**
```solidity
// ERC20Relayer.sol:85-96
function bridge() external onlyRole(BRIDGE_ROLE) {
    uint256 amount = IERC20(TOKEN).balanceOf(address(this));  // ❌ ENTIRE BALANCE
    IERC20(TOKEN).forceApprove(GATEWAY, amount);
    IScrollL2ERC20Gateway(GATEWAY).withdrawERC20(
        TOKEN,
        recipient,  // ❌ Can be changed by admin anytime
        amount,
        0
    );
    
    emit Bridged(TOKEN, recipient, amount);
}
```

**Impact:**
- BRIDGE_ROLE can drain all tokens in single transaction
- No amount limits or validation
- Recipient can be changed by admin immediately before bridge
- No timelock or multi-sig requirement
- Users lose all bridged funds

**Attack Scenario:**
```solidity
// Scenario 1: Malicious BRIDGE_ROLE
// 1. Users deposit 1,000,000 USX to bridge
// 2. Malicious BRIDGE_ROLE calls bridge()
// 3. All 1,000,000 USX sent to recipient on L1
// 4. Recipient is controlled by attacker
// 5. All funds stolen

// Scenario 2: Admin + BRIDGE_ROLE collusion
// 1. Users deposit 1,000,000 USX
// 2. Admin calls updateRecipient(attackerAddress)
// 3. BRIDGE_ROLE calls bridge()
// 4. All funds sent to attacker's L1 address
// 5. Both scenarios happen in same block
```

**Proof of Concept:**
```solidity
// Test case
function testBridgeDrainAttack() public {
    // Setup: 1M USX in relayer
    usxToken.mint(address(relayer), 1_000_000e18);
    
    // Attack: Change recipient and bridge
    vm.startPrank(admin);
    relayer.updateRecipient(attacker);
    vm.stopPrank();
    
    vm.startPrank(bridgeRole);
    relayer.bridge();
    vm.stopPrank();
    
    // Result: All tokens sent to attacker
    assertEq(usxToken.balanceOf(address(relayer)), 0);
    // Attacker receives 1M USX on L1
}
```

**Recommendation:**
```solidity
// Add maximum bridge amount and timelock
uint256 public constant MAX_BRIDGE_AMOUNT = 100_000e18; // 100k max per bridge
uint256 public constant BRIDGE_COOLDOWN = 1 hours;
uint256 public lastBridgeTimestamp;

mapping(bytes32 => PendingBridge) public pendingBridges;
struct PendingBridge {
    uint256 amount;
    address recipient;
    uint256 executeAfter;
}

function requestBridge(uint256 amount) external onlyRole(BRIDGE_ROLE) {
    require(amount > 0, "Zero amount");
    require(amount <= MAX_BRIDGE_AMOUNT, "Exceeds maximum");
    require(amount <= IERC20(TOKEN).balanceOf(address(this)), "Insufficient balance");
    require(block.timestamp >= lastBridgeTimestamp + BRIDGE_COOLDOWN, "Cooldown active");
    
    bytes32 bridgeId = keccak256(abi.encodePacked(amount, recipient, block.timestamp));
    pendingBridges[bridgeId] = PendingBridge({
        amount: amount,
        recipient: recipient,
        executeAfter: block.timestamp + 2 days
    });
    
    emit BridgeRequested(bridgeId, amount, recipient);
}

function executeBridge(bytes32 bridgeId) external onlyRole(BRIDGE_ROLE) {
    PendingBridge memory pending = pendingBridges[bridgeId];
    require(pending.amount > 0, "Invalid bridge");
    require(block.timestamp >= pending.executeAfter, "Timelock not passed");
    
    delete pendingBridges[bridgeId];
    lastBridgeTimestamp = block.timestamp;
    
    IERC20(TOKEN).forceApprove(GATEWAY, pending.amount);
    IScrollL2ERC20Gateway(GATEWAY).withdrawERC20(
        TOKEN,
        pending.recipient,
        pending.amount,
        0
    );
    
    emit Bridged(TOKEN, pending.recipient, pending.amount);
}

// Add emergency pause
bool public bridgingPaused;

function pauseBridging() external onlyRole(DEFAULT_ADMIN_ROLE) {
    bridgingPaused = true;
}

function unpauseBridging() external onlyRole(DEFAULT_ADMIN_ROLE) {
    bridgingPaused = false;
}
```

**References:**
- Checklist: SOL-AM-RP-1 (Rug Pull)
- Bridge security best practices
- Similar exploit: Poly Network bridge hack ($600M)

---

### [CRITICAL-4] Recipient Can Be Changed Without Timelock

**Contract:** `ERC20Relayer.sol`  
**Function:** `updateRecipient()`  
**Severity:** CRITICAL  
**Status:** ⚠️ NEW FINDING

**Description:**
The admin can change the recipient address instantly without any timelock. This can be used to redirect funds mid-flight or just before a bridge operation.

**Code Location:**
```solidity
// ERC20Relayer.sol:105-109
function updateRecipient(
    address newRecipient
) external onlyRole(DEFAULT_ADMIN_ROLE) {
    _updateRecipient(newRecipient);  // ❌ INSTANT CHANGE
}

// ERC20Relayer.sol:127-135
function _updateRecipient(address newRecipient) internal {
    if (newRecipient == address(0)) {
        revert ZeroAddress();
    }
    address oldRecipient = recipient;
    recipient = newRecipient;  // ❌ NO TIMELOCK
    
    emit UpdatedRecipient(oldRecipient, newRecipient);
}
```

**Impact:**
- Admin can front-run bridge transactions
- Funds redirected to malicious address
- No user notification or protection
- Instant execution enables atomic attacks

**Attack Scenario:**
```solidity
// 1. Legitimate recipient = treasuryL1 (0xABC...)
// 2. User initiates bridge of 100k USX
// 3. Admin sees transaction in mempool
// 4. Admin front-runs with updateRecipient(attackerL1)
// 5. User's bridge() executes with attacker as recipient
// 6. 100k USX sent to attacker on L1
// 7. Admin changes back to treasuryL1 to hide attack
```

**Recommendation:**
```solidity
address public pendingRecipient;
uint256 public recipientChangeTimestamp;
uint256 public constant RECIPIENT_CHANGE_DELAY = 7 days;

function proposeRecipientChange(address newRecipient) external onlyRole(DEFAULT_ADMIN_ROLE) {
    require(newRecipient != address(0), "Zero address");
    require(newRecipient != recipient, "Same recipient");
    
    pendingRecipient = newRecipient;
    recipientChangeTimestamp = block.timestamp + RECIPIENT_CHANGE_DELAY;
    
    emit RecipientChangeProposed(recipient, newRecipient, recipientChangeTimestamp);
}

function executeRecipientChange() external onlyRole(DEFAULT_ADMIN_ROLE) {
    require(pendingRecipient != address(0), "No pending change");
    require(block.timestamp >= recipientChangeTimestamp, "Timelock not passed");
    
    address oldRecipient = recipient;
    recipient = pendingRecipient;
    pendingRecipient = address(0);
    recipientChangeTimestamp = 0;
    
    emit UpdatedRecipient(oldRecipient, recipient);
}

function cancelRecipientChange() external onlyRole(DEFAULT_ADMIN_ROLE) {
    require(pendingRecipient != address(0), "No pending change");
    
    address cancelled = pendingRecipient;
    pendingRecipient = address(0);
    recipientChangeTimestamp = 0;
    
    emit RecipientChangeCancelled(cancelled);
}
```

---

## HIGH SEVERITY FINDINGS

### [HIGH-5] Missing Bridge Gateway Validation

**Contract:** `ERC20Relayer.sol`  
**Function:** Constructor  
**Severity:** HIGH  
**Status:** ⚠️ NEW FINDING

**Description:**
The constructor doesn't validate that the gateway address is actually a valid Scroll gateway contract. A malicious gateway could steal all approved tokens.

**Code Location:**
```solidity
// ERC20Relayer.sol:70-77
constructor(address _gateway, address _token, address _recipient) {
    GATEWAY = _gateway;  // ❌ NO VALIDATION
    TOKEN = _token;      // ❌ NO VALIDATION
    
    _grantRole(DEFAULT_ADMIN_ROLE, msg.sender);
    
    _updateRecipient(_recipient);
}
```

**Impact:**
- Malicious gateway can steal all approved tokens
- No way to verify gateway is legitimate
- Immutable variable cannot be changed
- Requires redeployment to fix

**Recommendation:**
```solidity
constructor(address _gateway, address _token, address _recipient) {
    require(_gateway != address(0), "Zero gateway");
    require(_token != address(0), "Zero token");
    
    // Validate gateway implements expected interface
    try IScrollL2ERC20Gateway(_gateway).router() returns (address) {
        // Gateway is valid
    } catch {
        revert("Invalid gateway");
    }
    
    GATEWAY = _gateway;
    TOKEN = _token;
    
    _grantRole(DEFAULT_ADMIN_ROLE, msg.sender);
    _updateRecipient(_recipient);
}
```

---

### [HIGH-6] LayerZero OFT Contracts Missing Access Controls

**Contracts:** `USXOFT.sol`, `StakedUSXOFT.sol`, `USXOFTAdaptor.sol`, `StakedUSXOFTAdaptor.sol`  
**Severity:** HIGH  
**Status:** ⚠️ NEW FINDING

**Description:**
The OFT contracts are extremely minimal wrappers around LayerZero's OFT implementation. They inherit all LayerZero functionality but don't add any protocol-specific access controls or validations.

**Code Location:**
```solidity
// USXOFT.sol:7-14
contract USXOFT is OFT {
    constructor(
        string memory _name,
        string memory _symbol,
        address _lzEndpoint,
        address _owner
    ) OFT(_name, _symbol, _lzEndpoint, _owner) Ownable(_owner) {}
    // ❌ NO ADDITIONAL LOGIC
    // ❌ NO ACCESS CONTROLS
    // ❌ NO VALIDATION
}
```

**Impact:**
- Relies entirely on LayerZero's security
- No protocol-specific protections
- No bridge limits or validations
- Owner has unlimited power
- No integration with USX protocol governance

**Concerns:**
1. **Owner Centralization:** Single owner controls all LayerZero operations
2. **No Bridge Limits:** Unlimited cross-chain transfers
3. **No Pause Mechanism:** Cannot pause bridging in emergency
4. **No Whitelist:** Anyone can bridge tokens
5. **No Integration:** Not connected to USX governance or admin

**Recommendation:**
```solidity
contract USXOFT is OFT, AccessControl {
    bytes32 public constant BRIDGE_ADMIN_ROLE = keccak256("BRIDGE_ADMIN_ROLE");
    bytes32 public constant PAUSER_ROLE = keccak256("PAUSER_ROLE");
    
    bool public bridgingPaused;
    uint256 public maxBridgeAmount;
    mapping(address => bool) public whitelistedUsers;
    
    constructor(
        string memory _name,
        string memory _symbol,
        address _lzEndpoint,
        address _owner,
        address _admin
    ) OFT(_name, _symbol, _lzEndpoint, _owner) Ownable(_owner) {
        _grantRole(DEFAULT_ADMIN_ROLE, _admin);
        _grantRole(BRIDGE_ADMIN_ROLE, _admin);
        _grantRole(PAUSER_ROLE, _admin);
        
        maxBridgeAmount = 1_000_000e18; // 1M max
    }
    
    // Override send to add validations
    function send(
        SendParam calldata _sendParam,
        MessagingFee calldata _fee,
        address _refundAddress
    ) external payable override returns (MessagingReceipt memory) {
        require(!bridgingPaused, "Bridging paused");
        require(_sendParam.amountLD <= maxBridgeAmount, "Exceeds max bridge amount");
        require(whitelistedUsers[msg.sender], "Not whitelisted");
        
        return super.send(_sendParam, _fee, _refundAddress);
    }
    
    function pauseBridging() external onlyRole(PAUSER_ROLE) {
        bridgingPaused = true;
    }
    
    function unpauseBridging() external onlyRole(PAUSER_ROLE) {
        bridgingPaused = false;
    }
    
    function setMaxBridgeAmount(uint256 _max) external onlyRole(BRIDGE_ADMIN_ROLE) {
        maxBridgeAmount = _max;
    }
    
    function whitelistUser(address user, bool status) external onlyRole(BRIDGE_ADMIN_ROLE) {
        whitelistedUsers[user] = status;
    }
}
```

---

## MEDIUM SEVERITY FINDINGS

### [MEDIUM-7] ERC20Relayer Rescue Function Can Steal Bridged Tokens

**Contract:** `ERC20Relayer.sol`  
**Function:** `rescueERC20()`  
**Severity:** MEDIUM  
**Status:** ⚠️ NEW FINDING

**Description:**
The `rescueERC20()` function allows admin to withdraw ANY token, including the main TOKEN that users deposited for bridging. This creates a rug pull vector.

**Code Location:**
```solidity
// ERC20Relayer.sol:113-119
function rescueERC20(
    IERC20 tokenContract,
    address to,
    uint256 amount
) external onlyRole(DEFAULT_ADMIN_ROLE) {
    tokenContract.safeTransfer(to, amount);  // ❌ CAN WITHDRAW MAIN TOKEN
}
```

**Impact:**
- Admin can steal user deposits
- No restriction on which tokens can be rescued
- No timelock or notification
- Users lose funds waiting to be bridged

**Attack Scenario:**
```solidity
// 1. Users deposit 500k USX to be bridged
// 2. Admin calls rescueERC20(USX, adminAddress, 500k)
// 3. All user deposits stolen
// 4. Users cannot bridge their tokens
```

**Recommendation:**
```solidity
function rescueERC20(
    IERC20 tokenContract,
    address to,
    uint256 amount
) external onlyRole(DEFAULT_ADMIN_ROLE) {
    require(address(tokenContract) != TOKEN, "Cannot rescue main token");
    require(to != address(0), "Invalid recipient");
    
    tokenContract.safeTransfer(to, amount);
}

// Or add timelock for main token rescue in emergency
mapping(bytes32 => uint256) public rescueRequests;

function requestMainTokenRescue(uint256 amount) external onlyRole(DEFAULT_ADMIN_ROLE) {
    bytes32 requestId = keccak256(abi.encodePacked(amount, block.timestamp));
    rescueRequests[requestId] = block.timestamp + 7 days;
    emit RescueRequested(requestId, amount);
}

function executeMainTokenRescue(bytes32 requestId, address to, uint256 amount) external onlyRole(DEFAULT_ADMIN_ROLE) {
    require(block.timestamp >= rescueRequests[requestId], "Timelock not passed");
    require(rescueRequests[requestId] != 0, "Invalid request");
    
    delete rescueRequests[requestId];
    IERC20(TOKEN).safeTransfer(to, amount);
    
    emit MainTokenRescued(to, amount);
}
```

---

### [MEDIUM-8] TreasuryStorage Missing Storage Gap

**Contract:** `TreasuryStorage.sol`  
**Severity:** MEDIUM  
**Status:** ⚠️ NEW FINDING

**Description:**
The TreasuryStorage contract is designed to be inherited but doesn't include a storage gap. This can cause storage collisions if the contract is upgraded and new variables are added.

**Code Location:**
```solidity
// TreasuryStorage.sol:94-110
struct TreasuryStorageStruct {
    IUSX USX;
    IStakedUSX sUSX;
    IERC20 USDC;
    address admin;
    address governance;
    address reporter;
    address allocator;
    address assetManager;
    address insuranceVault;
    address governanceWarchest;
    uint256 successFeeFraction;
    uint256 insuranceFundFraction;
    uint256 assetManagerUSDC;
    uint256 netEpochProfits;
}
// ❌ NO STORAGE GAP
```

**Impact:**
- Future upgrades may corrupt storage
- Adding new variables can overwrite existing data
- Difficult to maintain upgrade compatibility

**Recommendation:**
```solidity
struct TreasuryStorageStruct {
    IUSX USX;
    IStakedUSX sUSX;
    IERC20 USDC;
    address admin;
    address governance;
    address reporter;
    address allocator;
    address assetManager;
    address insuranceVault;
    address governanceWarchest;
    uint256 successFeeFraction;
    uint256 insuranceFundFraction;
    uint256 assetManagerUSDC;
    uint256 netEpochProfits;
    
    // Storage gap for future upgrades
    uint256[50] __gap;
}
```

---

### [MEDIUM-9] LayerZero Endpoint Not Validated

**Contracts:** All OFT contracts  
**Severity:** MEDIUM  
**Status:** ⚠️ NEW FINDING

**Description:**
The LayerZero endpoint address is not validated in the constructor. A wrong or malicious endpoint could compromise all cross-chain operations.

**Code Location:**
```solidity
// USXOFT.sol:8-13
constructor(
    string memory _name,
    string memory _symbol,
    address _lzEndpoint,  // ❌ NOT VALIDATED
    address _owner
) OFT(_name, _symbol, _lzEndpoint, _owner) Ownable(_owner) {}
```

**Impact:**
- Malicious endpoint can intercept messages
- Wrong endpoint breaks bridging
- Cannot be changed (immutable)
- Requires redeployment

**Recommendation:**
```solidity
// Maintain list of valid LayerZero endpoints per chain
mapping(uint256 => address) public validEndpoints;

constructor(
    string memory _name,
    string memory _symbol,
    address _lzEndpoint,
    address _owner
) OFT(_name, _symbol, _lzEndpoint, _owner) Ownable(_owner) {
    require(_lzEndpoint != address(0), "Zero endpoint");
    
    // Validate endpoint for current chain
    uint256 chainId = block.chainid;
    if (chainId == 1) {
        require(_lzEndpoint == 0x66A71Dcef29A0fFBDBE3c6a460a3B5BC225Cd675, "Invalid endpoint");
    } else if (chainId == 534352) { // Scroll
        require(_lzEndpoint == 0xb6319cC6c8c27A8F5dAF0dD3DF91EA35C4720dd7, "Invalid endpoint");
    }
    // Add more chains as needed
}
```

---

## LOW SEVERITY FINDINGS

### [LOW-6] Missing Events in TreasuryStorage

**Contract:** `TreasuryStorage.sol`  
**Severity:** LOW

**Description:**
TreasuryStorage defines events but they're only emitted in facets. Direct storage changes wouldn't emit events.

**Recommendation:**
Ensure all state changes in facets properly emit events defined in TreasuryStorage.

---

### [LOW-7] No Bridge Amount Validation

**Contract:** `ERC20Relayer.sol`  
**Function:** `bridge()`  
**Severity:** LOW

**Description:**
The bridge function doesn't check if amount > 0, allowing wasteful empty bridge transactions.

**Recommendation:**
```solidity
function bridge() external onlyRole(BRIDGE_ROLE) {
    uint256 amount = IERC20(TOKEN).balanceOf(address(this));
    require(amount > 0, "No tokens to bridge");
    
    // ... rest of function
}
```

---

### [LOW-8] OFT Contracts Missing NatSpec

**Contracts:** All OFT contracts  
**Severity:** LOW

**Description:**
The OFT wrapper contracts have no documentation explaining their purpose or LayerZero integration.

**Recommendation:**
Add comprehensive NatSpec:
```solidity
/// @title USXOFT
/// @notice LayerZero OFT wrapper for USX token cross-chain transfers
/// @dev Inherits from LayerZero's OFT implementation
/// @custom:security-note Owner has full control over LayerZero operations
contract USXOFT is OFT {
    /// @notice Deploys the USX OFT contract
    /// @param _name Token name
    /// @param _symbol Token symbol
    /// @param _lzEndpoint LayerZero endpoint address for this chain
    /// @param _owner Owner address with full control
    constructor(
        string memory _name,
        string memory _symbol,
        address _lzEndpoint,
        address _owner
    ) OFT(_name, _symbol, _lzEndpoint, _owner) Ownable(_owner) {}
}
```

---

## INFORMATIONAL FINDINGS

### [INFO-9] Duplicate Ownable Inheritance

**Contracts:** All OFT contracts  
**Severity:** INFORMATIONAL

**Description:**
The OFT contracts inherit from both `OFT` (which inherits `Ownable`) and explicitly inherit `Ownable` again.

**Code:**
```solidity
contract USXOFT is OFT {
    constructor(
        string memory _name,
        string memory _symbol,
        address _lzEndpoint,
        address _owner
    ) OFT(_name, _symbol, _lzEndpoint, _owner) Ownable(_owner) {}
    //                                          ^^^^^^^^^^^^^^^^ Redundant
}
```

**Recommendation:**
Remove redundant `Ownable(_owner)` call:
```solidity
) OFT(_name, _symbol, _lzEndpoint, _owner) {}
```

---

### [INFO-10] Consider Rate Limiting Bridge Operations

**Contract:** `ERC20Relayer.sol`  
**Severity:** INFORMATIONAL

**Description:**
Consider adding rate limiting to prevent rapid bridge operations that could drain liquidity.

**Recommendation:**
```solidity
uint256 public bridgeRateLimit = 100_000e18; // 100k per hour
uint256 public lastBridgeHour;
uint256 public bridgedThisHour;

function bridge() external onlyRole(BRIDGE_ROLE) {
    uint256 currentHour = block.timestamp / 1 hours;
    
    if (currentHour != lastBridgeHour) {
        lastBridgeHour = currentHour;
        bridgedThisHour = 0;
    }
    
    uint256 amount = IERC20(TOKEN).balanceOf(address(this));
    require(bridgedThisHour + amount <= bridgeRateLimit, "Rate limit exceeded");
    
    bridgedThisHour += amount;
    
    // ... rest of bridge logic
}
```

---

### [INFO-11] LayerZero Security Considerations

**Contracts:** All OFT contracts  
**Severity:** INFORMATIONAL

**Description:**
LayerZero OFT contracts should implement additional security measures:

1. **Trusted Remotes:** Validate source chains
2. **Message Validation:** Verify message integrity
3. **Replay Protection:** Prevent message replay
4. **Rate Limiting:** Limit cross-chain transfers
5. **Pause Mechanism:** Emergency stop

**Recommendation:**
Review LayerZero security best practices and implement recommended patterns.

---

## SUMMARY OF SUPPLEMENTARY FINDINGS

### New Critical Issues
- **[CRITICAL-3]** Bridge relayer can drain all tokens
- **[CRITICAL-4]** Recipient can be changed without timelock

### New High Issues
- **[HIGH-5]** Missing bridge gateway validation
- **[HIGH-6]** LayerZero OFT contracts missing access controls

### New Medium Issues
- **[MEDIUM-7]** Rescue function can steal bridged tokens
- **[MEDIUM-8]** Missing storage gap in TreasuryStorage
- **[MEDIUM-9]** LayerZero endpoint not validated

### New Low Issues
- **[LOW-6]** Missing events in TreasuryStorage
- **[LOW-7]** No bridge amount validation
- **[LOW-8]** OFT contracts missing NatSpec

### New Informational Issues
- **[INFO-9]** Duplicate Ownable inheritance
- **[INFO-10]** Consider rate limiting bridge operations
- **[INFO-11]** LayerZero security considerations

---

## COMBINED AUDIT STATISTICS

### Total Findings Across Both Reports

| Severity | Main Report | Supplementary | **Total** |
|----------|-------------|---------------|-----------|
| **CRITICAL** | 2 | 2 | **4** |
| **HIGH** | 4 | 2 | **6** |
| **MEDIUM** | 6 | 3 | **9** |
| **LOW** | 5 | 3 | **8** |
| **INFORMATIONAL** | 8 | 3 | **11** |
| **TOTAL** | **25** | **13** | **38** |

---

## PRIORITY RECOMMENDATIONS

### Immediate Actions (Critical)

1. **Add Timelock to Bridge Operations**
   - Implement 2-day timelock for bridge() function
   - Add maximum bridge amount limits
   - Require multi-sig for large bridges

2. **Protect Recipient Changes**
   - Add 7-day timelock for updateRecipient()
   - Emit clear warnings to users
   - Implement cancellation mechanism

3. **Validate Bridge Gateway**
   - Add interface validation in constructor
   - Verify gateway is legitimate Scroll contract
   - Consider using factory pattern

4. **Add Access Controls to OFT Contracts**
   - Implement role-based access control
   - Add pause mechanism
   - Implement bridge limits
   - Add whitelist for bridgers

### Short-Term Improvements (High/Medium)

1. **Restrict Rescue Function**
   - Prevent rescue of main TOKEN
   - Add timelock for emergency rescue
   - Require multi-sig approval

2. **Add Storage Gap**
   - Add `uint256[50] __gap` to TreasuryStorage
   - Document upgrade process
   - Test storage layout compatibility

3. **Validate LayerZero Endpoints**
   - Maintain whitelist of valid endpoints
   - Validate in constructor
   - Add endpoint update mechanism with timelock

---

## CONCLUSION

This supplementary audit identified **13 additional findings** including **2 CRITICAL** issues in the bridge infrastructure. The most severe issues relate to:

1. **Bridge Security:** Unlimited token drainage and instant recipient changes
2. **Access Control:** Missing validations in OFT contracts
3. **Upgrade Safety:** Missing storage gaps

Combined with the main audit, the protocol now has **38 total findings** with **4 CRITICAL** and **6 HIGH** severity issues requiring immediate attention.

### Overall Risk Assessment

**Bridge Infrastructure Risk:** 🔴 **CRITICAL**  
**Overall Protocol Risk:** 🔴 **HIGH**

The bridge components (ERC20Relayer and OFT contracts) introduce significant additional risks that must be addressed before deployment.

---

**End of Supplementary Report**

*This audit completes the comprehensive review of all Solidity files in the USX Protocol.*
