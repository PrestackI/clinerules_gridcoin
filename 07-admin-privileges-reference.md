# Administrative Privileges Reference

## Overview

This document provides a comprehensive reference for administrative privileges and features currently implemented in the Gridcoin codebase. These systems use cryptographic keys to authorize privileged operations, ensuring network security and governance.

---

## Table of Contents

1. [Alert System](#1-alert-system)
2. [Master Keys for Administrative Contracts](#2-master-keys-for-administrative-contracts)
3. [Contract Authorization Requirements](#3-contract-authorization-requirements)
4. [Signature Verification Process](#4-signature-verification-process)
5. [Key Management and Rotation](#5-key-management-and-rotation)
6. [Security Considerations](#6-security-considerations)

---

## 1. Alert System

### Purpose
The alert system allows authorized administrators to broadcast network-wide alerts to notify users of critical issues, security vulnerabilities, or required upgrades.

### Implementation

**Files:**
- Header: `src/alert.h`
- Implementation: `src/alert.cpp`
- Key Definition: `src/chainparams.cpp`

**Key Structure:**
```cpp
// In CMainParams (mainnet):
vAlertPubKey = ParseHex("0352063cf6cf0317cc848ae24f3ed8b525334d2f059f242d27975f8c3a2e91b446");

// In CTestNetParams (testnet):
vAlertPubKey = ParseHex("02bf4aa6330f525ab91a25cd5c1362481d16d8c039b3d27cb48ac0870176202462");
```

### Alert Types

**CUnsignedAlert Fields:**
- `nVersion` - Alert version number
- `nRelayUntil` - Timestamp when nodes stop relaying to newer nodes
- `nExpiration` - When the alert expires
- `nID` - Unique alert identifier
- `nCancel` - ID of alert to cancel
- `setCancel` - Set of alert IDs to cancel
- `nMinVer` / `nMaxVer` - Version range the alert applies to
- `setSubVer` - Specific client versions (empty matches all)
- `nPriority` - Alert priority level
- `strStatusBar` - Message displayed in status bar
- `strComment` - Internal comment (not displayed)

### Signature Verification

**Method:** `CAlert::CheckSignature()`

```cpp
bool CAlert::CheckSignature() const
{
    if (!CPubKey(Params().AlertKey()).Verify(Hash(vchMsg), vchSig))
        return error("CAlert::CheckSignature() : verify signature failed");
    
    // Unserialize and validate...
    return true;
}
```

### Special Alert: Key Compromise

A special alert with `nID = max(int)` is reserved for key compromise situations:

**Requirements:**
- Must have ID = `std::numeric_limits<int>::max()`
- Must never expire (`nExpiration = maxInt`)
- Must cancel all previous alerts (`nCancel = maxInt-1`)
- Must apply to all versions (`nMinVer = 0`, `nMaxVer = maxInt`)
- Must have empty version filter (`setSubVer.empty()`)
- Must have maximum priority (`nPriority = maxInt`)
- Must have specific message: `"URGENT: Alert key compromised, upgrade required"`
- Must have empty comment and reserved fields
- Must be version 1

**Purpose:** If the alert key is compromised, this special alert can be issued with a pre-defined message that cannot be overridden by an attacker.

### Alert Limitations

- Maximum 5 concurrent active alerts
- Alerts can cancel previous alerts
- Expired alerts are automatically removed
- Alerts must be properly signed to be processed

---

## 2. Master Keys for Administrative Contracts

### Purpose
Master keys authorize administrative contract operations such as modifying the project whitelist, protocol settings, or removing malicious content.

### Implementation

**File:** `src/chainparams.cpp`

**Key Structure:**
```cpp
// Mainnet master keys
masterkeys = {
    {0,       ParseHex("049ac003b3318d9fe28b2830f6a95a2624ce2a69fb0c0c7ac0b513efcc1e93a6a6e8eba84481155dd82f2f1104e0ff62c69d662b0094639b7106abc5d84f948c0a")},
    {2671700, ParseHex("0288b33697c4c752f922764bf1a5075fa96bad46aaf4f0579bf7d19ab048e200f0")}
};

// Testnet master keys
masterkeys = {
    {0,       ParseHex("049ac003b3318d9fe28b2830f6a95a2624ce2a69fb0c0c7ac0b513efcc1e93a6a6e8eba84481155dd82f2f1104e0ff62c69d662b0094639b7106abc5d84f948c0a")},
    {1964600, ParseHex("031886a6776699cbd6362df7641c5d128146afabc769dfa36f1630889c706ce730")}
};
```

### Key Format

- **Original Key (Block 0):** Uncompressed format (130 hex characters)
- **Rotated Keys:** Compressed format (66 hex characters)

### Key Retrieval

**Method:** `Params().MasterKey(block_height)`

The system automatically selects the appropriate key based on block height. When multiple keys are defined, it uses the key with the highest block height that doesn't exceed the current block.

**Example:**
```cpp
// At block 1,000,000 on mainnet:
Params().MasterKey(1000000);  // Returns block 0 key

// At block 3,000,000 on mainnet:
Params().MasterKey(3000000);  // Returns block 2671700 key
```

---

## 3. Contract Authorization Requirements

### Implementation

**File:** `src/gridcoin/contract/contract.cpp`

**Method:** `Contract::RequiresMasterKey()`

This method determines which contract types require master key authorization:

```cpp
bool Contract::RequiresMasterKey() const
{
    switch (m_type.Value()) {
        case ContractType::BEACON:
            // v1: Admin can delete any beacon
            // v2+: Participants can revoke their own beacons
            return m_version == 1 && m_action == ContractAction::REMOVE;

        case ContractType::POLL:      
            return m_action == ContractAction::REMOVE;
            
        case ContractType::PROJECT:   
            return true;  // ALL actions require admin
            
        case ContractType::PROTOCOL:  
            return true;  // ALL actions require admin
            
        case ContractType::SCRAPER:   
            return true;  // ALL actions require admin
            
        case ContractType::VOTE:      
            return m_action == ContractAction::REMOVE;
            
        case ContractType::SIDESTAKE: 
            return true;  // ALL actions require admin
            
        default:                      
            return false;
    }
}
```

### Contract Type Details

| Contract Type | Add Requires Admin? | Delete Requires Admin? | Notes |
|---------------|---------------------|------------------------|-------|
| **BEACON** | No (v2+) | Yes (v1 only) | v2+ allows self-revocation with original key |
| **POLL** | No | Yes | Users can create polls; admins can remove |
| **PROJECT** | **Yes** | **Yes** | Full admin control over whitelist |
| **PROTOCOL** | **Yes** | **Yes** | Full admin control over protocol settings |
| **SCRAPER** | **Yes** | **Yes** | Full admin control over scraper authorization |
| **VOTE** | No | Yes | Users can vote; admins can remove votes |
| **SIDESTAKE** | **Yes** | **Yes** | Full admin control over mandatory sidestakes |
| **MRC** | No | N/A | Manual Reward Claims validated differently |
| **CLAIM** | No | N/A | Research reward claims (coinbase only) |
| **MESSAGE** | No | N/A | Transaction messages (not tracked in index) |

### Decentralized vs. Centralized Contract Types

**Decentralized (User-initiated):**
- BEACON (v2+) - Registration and self-revocation
- POLL - Creation
- VOTE - Submission
- MRC - Manual reward claims
- CLAIM - Research rewards (automatic)

**Centralized (Admin-controlled):**
- PROJECT - Whitelist management
- PROTOCOL - Protocol parameter configuration
- SCRAPER - Scraper authorization
- SIDESTAKE - Mandatory sidestake allocation

**Hybrid (User creation + Admin removal):**
- BEACON (v1) - User registration, admin removal
- POLL - User creation, admin removal
- VOTE - User submission, admin removal

---

## 4. Signature Verification Process

### Legacy Contract Validation

**Function:** `CheckLegacyContract()`

**File:** `src/gridcoin/contract/contract.cpp`

**Process:**

1. **Check Well-Formed Status**
   ```cpp
   if (!contract.WellFormed()) {
       return false;
   }
   ```

2. **Check Admin Requirement**
   ```cpp
   if (!contract.RequiresMasterKey()) {
       return true;  // No signature needed
   }
   ```

3. **Extract Signature from Transaction**
   ```cpp
   const std::string base64_sig = ExtractXML(tx.hashBoinc, "<MS>", "</MS>");
   if (base64_sig.empty()) {
       return false;
   }
   ```

4. **Decode Base64 Signature**
   ```cpp
   bool invalid;
   const std::vector<uint8_t> sig = DecodeBase64(base64_sig.c_str(), &invalid);
   if (invalid) {
       return false;
   }
   ```

5. **Hash Contract Body**
   ```cpp
   const std::string type_string = contract.m_type.ToString();
   const auto& body = static_cast<const LegacyPayload&>(*payload);
   const uint256 body_hash = Hash(type_string, body.m_key, body.m_value);
   ```

6. **Verify Signature**
   ```cpp
   return CPubKey(Params().MasterKey(block_height)).Verify(body_hash, sig);
   ```

### Version 2+ Contracts

Version 2+ contracts are validated upon receipt and do not include the signature in the serialized format. The validation is more robust and occurs in the mempool and during block validation.

**Validation Chain:**
1. `ValidateContracts()` - Called during transaction validation
2. `BlockValidateContracts()` - Called during block connection (`ConnectBlock`)
3. Contract-specific handlers perform additional validation

---

## 5. Key Management and Rotation

### Key Rotation History

**Mainnet:**
- Block 0: Original uncompressed key
- Block 2,671,700: Rotated to compressed key (Kermit's Mom update)

**Testnet:**
- Block 0: Same original key as mainnet
- Block 1,964,600: Rotated to compressed key

### Why Key Rotation?

**Reasons for rotation:**
1. **Security:** Periodic rotation limits exposure from potential compromise
2. **Format modernization:** Transition from uncompressed to compressed keys
3. **Coordinated updates:** Tied to major protocol upgrades (hard forks)

**Implementation:**
The `masterkeys` map allows seamless key transitions at specific block heights without breaking historical signature verification.

### Alert Key vs. Master Key

**Alert Key:**
- Single key per network (mainnet/testnet)
- Used exclusively for network alerts
- Different key for mainnet vs. testnet
- Never rotated (would require all nodes to update)

**Master Keys:**
- Multiple keys per network (supports rotation)
- Used for administrative contracts
- Height-based selection
- Can be rotated at hard fork boundaries

---

## 6. Security Considerations

### Private Key Security

**Critical Requirements:**
- Private keys must be stored offline (cold storage)
- Multi-signature or threshold schemes recommended
- Access should be restricted to core developers
- Keys should never be exposed in code or configuration files

### Testnet vs. Mainnet Separation

**Important Notes:**
- Testnet uses different keys than mainnet
- This prevents testnet actions from affecting mainnet
- Testnet keys may be more accessible for testing

### Signature Verification Failures

**Common Issues:**

1. **Wrong Key Used**
   - Signing with testnet key on mainnet (or vice versa)
   - Using outdated key after rotation

2. **Incorrect Hash**
   - Wrong contract body format
   - Incorrect serialization order

3. **Malformed Signature**
   - Invalid Base64 encoding
   - Corrupted signature data

4. **Replay Attacks**
   - Historical contracts are validated with appropriate keys for their block height
   - This prevents old signatures from being replayed with different keys

### Testnet Bad Contracts

The validation function specifically handles testnet issues:

```cpp
// Testnet contains some bad administrative contracts that this routine filters out.
```

This explains why `CheckLegacyContract()` performs sanity checks on historical messages.

---

## 7. Administrative Workflows

### Creating an Administrative Contract

**General Process:**

1. **Prepare Contract Data**
   - Determine contract type (PROJECT, PROTOCOL, etc.)
   - Prepare key and value fields
   - Determine action (ADD or REMOVE)

2. **Hash the Contract Body**
   ```cpp
   uint256 body_hash = Hash(type_string, key, value);
   ```

3. **Sign with Master Key** (Offline)
   ```bash
   # This would be done with offline signing tool
   # using the private key corresponding to Params().MasterKey()
   ```

4. **Encode Signature**
   ```cpp
   std::string base64_sig = EncodeBase64(signature);
   ```

5. **Build Transaction**
   - Create transaction with contract data
   - Include signature in `<MS>...</MS>` tags (v1 contracts)
   - Include required burn amount

6. **Broadcast**
   - Submit transaction to network
   - Network validates signature
   - Contract applied if valid

### Removing Malicious Content

**Use Cases:**
- Remove fraudulent poll
- Delete compromised beacon
- Remove malicious vote
- Delist problematic project

**Process:**
1. Identify the contract to remove (key/identifier)
2. Create REMOVE contract
3. Sign with master key
4. Broadcast removal transaction
5. Network applies deletion

---

## 8. Code Reference Summary

### Key Files

| File | Purpose |
|------|---------|
| `src/alert.h` | Alert data structures |
| `src/alert.cpp` | Alert processing and verification |
| `src/chainparams.h` | Chain parameter declarations |
| `src/chainparams.cpp` | Alert keys and master keys definition |
| `src/gridcoin/contract/contract.h` | Contract structure and interface |
| `src/gridcoin/contract/contract.cpp` | Contract validation and admin checks |
| `src/gridcoin/contract/handler.h` | Contract handler interface |
| `src/gridcoin/contract/registry.h` | Contract registry management |

### Key Methods

| Method | File | Purpose |
|--------|------|---------|
| `CAlert::CheckSignature()` | `alert.cpp` | Verify alert signature |
| `CAlert::ProcessAlert()` | `alert.cpp` | Process and apply alert |
| `Params().AlertKey()` | `chainparams.cpp` | Get alert public key |
| `Params().MasterKey(height)` | `chainparams.cpp` | Get master key for block height |
| `Contract::RequiresMasterKey()` | `contract.cpp` | Check if contract needs admin |
| `CheckLegacyContract()` | `contract.cpp` | Validate legacy v1 contract |
| `ValidateContracts()` | `contract.cpp` | Validate transaction contracts |
| `BlockValidateContracts()` | `contract.cpp` | Validate contracts in block context |

### Key Classes

| Class | File | Purpose |
|-------|------|---------|
| `CAlert` | `alert.h` | Network alert with signature |
| `CUnsignedAlert` | `alert.h` | Alert data without signature |
| `Contract` | `contract.h` | Contract message structure |
| `IContractHandler` | `handler.h` | Interface for contract handlers |
| `BeaconRegistry` | `beacon.h` | Beacon contract handler |
| `Whitelist` | `project.h` | Project whitelist handler |
| `ProtocolRegistry` | `protocol.h` | Protocol entry handler |
| `ScraperRegistry` | `scraper_registry.h` | Scraper authorization handler |
| `SideStakeRegistry` | `sidestake.h` | Sidestake contract handler |

---

## 9. Historical Context

### Evolution of Administrative Controls

**Early Gridcoin (Pre-v11):**
- All contracts used legacy XML-like string format
- Master key signatures embedded in transaction messages
- Less robust validation

**Block v11+ (Version 2 Contracts):**
- Binary contract format introduced
- More robust validation
- Better separation of contract types

**Block v13+ (Version 3 Contracts):**
- Native binary format for scraper and protocol entries
- No longer using legacy payload conversion

**Kermit's Mom (5.3.3.12 / 5.4.0.0):**
- Master key rotation (mainnet block 2,671,700)
- Alert key updates
- Improved contract validation

### Design Philosophy

**Balance of Power:**
- Most contracts are decentralized (user-initiated)
- Administrative controls limited to:
  - Network infrastructure (whitelist, protocol)
  - Content moderation (poll/vote removal)
  - Security (beacon removal in v1)
- Cryptographic proof prevents unauthorized admin actions

**Progressive Decentralization:**
- BEACON contracts moved from admin-removal (v1) to self-revocation (v2+)
- Future contracts may follow similar patterns
- Goal: Minimize reliance on administrative keys while maintaining security

---

## 10. Future Considerations

### Potential Enhancements

**Multi-Signature Support:**
- Require multiple admin keys for sensitive operations
- Increases security through threshold signatures
- Prevents single point of failure

**Time-Locked Admin Actions:**
- Announce admin actions before execution
- Community review period
- Automatic execution after timelock expires

**Community Governance:**
- On-chain voting for administrative decisions
- Transparent audit trail
- Gradual transition from key-based to vote-based admin

**Emergency Response:**
- Faster key rotation mechanisms
- Revocation lists for compromised keys
- Fallback procedures

---

## References

- **GitHub Repository:** https://github.com/gridcoin-community/Gridcoin-Research
- **Related Issues:**
  - Alert System: Historical Bitcoin-derived code
  - Master Keys: Gridcoin-specific administrative framework
- **Documentation:**
  - [Contract System Overview](./05-common-tasks.md#section-2-adding-contract-types)
  - [Pool Contract Design](./06-issue-1783-pool-contract-analysis.md)

---

**Last Updated:** 2025-12-26  
**Status:** Reference Documentation  
**Maintainer:** Gridcoin Development Team
