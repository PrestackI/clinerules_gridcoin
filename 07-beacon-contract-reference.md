# Beacon Contract Reference

## Purpose

This document provides a technical reference for the beacon contract implementation, serving as a template for implementing the pool contract system (Issue #1783). Understanding the beacon pattern is essential for creating consistent, robust contract handlers.

---

## Architecture Overview

```
┌─────────────────────────────────────────────────────────────┐
│                    Beacon Contract System                    │
└─────────────────────────────────────────────────────────────┘
                              │
        ┌─────────────────────┼─────────────────────┐
        │                     │                     │
    ┌───▼────┐          ┌─────▼──────┐      ┌─────▼──────┐
    │ Beacon │          │  Beacon    │      │  Beacon    │
    │  Data  │          │  Payload   │      │  Registry  │
    │ (Entry)│          │ (Contract) │      │ (Handler)  │
    └────────┘          └────────────┘      └────────────┘
```

---

## 1. Data Structures

### 1.1 Beacon Class (`src/gridcoin/beacon.h`)

**Core beacon data container**

```cpp
class Beacon {
public:
    static constexpr int64_t MAX_AGE = 60 * 60 * 24 * 30 * 6;      // 6 months
    static constexpr int64_t RENEWAL_AGE = 60 * 60 * 24 * 30 * 5;  // 5 months

    Cpid m_cpid;              // Researcher's CPID
    CPubKey m_public_key;     // Verification key for blocks
    int64_t m_timestamp;      // Advertisement time
    uint256 m_hash;           // Transaction hash
    uint256 m_previous_hash;  // Previous beacon (chainlet link)
    Status m_status;          // BeaconStatusForStorage enum
    
    // Key methods
    bool WellFormed() const;
    bool Expired(const int64_t now) const;
    bool Renewable(const int64_t now) const;
    bool Renewed() const;
    CKeyID GetId() const;  // Hash of public key
    std::string GetVerificationCode() const;
};
```

**Key Pattern**: Stores both current state and historical linkage (`m_previous_hash`)

### 1.2 Beacon Status Enum

```cpp
enum class BeaconStatusForStorage {
    UNKNOWN,          // Invalid/uninitialized
    PENDING,          // Awaiting verification in superblock
    ACTIVE,           // Verified and active
    RENEWAL,          // Renewed beacon (still active)
    EXPIRED_PENDING,  // Expired while pending
    DELETED,          // Revoked by owner
    OUT_OF_BOUND      // Enum boundary marker
};
```

**Lifecycle Flow:**
```
UNKNOWN → PENDING → ACTIVE → RENEWAL → (aged out)
             ↓
     EXPIRED_PENDING
```

### 1.3 BeaconPayload Class (Contract Payload)

```cpp
class BeaconPayload : public IContractPayload {
public:
    static constexpr uint32_t CURRENT_VERSION = 2;
    
    uint32_t m_version = CURRENT_VERSION;
    Cpid m_cpid;                      // Researcher CPID
    Beacon m_beacon;                  // Beacon data
    std::vector<uint8_t> m_signature; // Signed by beacon private key
    
    // IContractPayload interface
    GRC::ContractType ContractType() const override {
        return GRC::ContractType::BEACON;
    }
    
    bool WellFormed(const ContractAction action) const override;
    std::string LegacyKeyString() const override;
    std::string LegacyValueString() const override;
    
    CAmount RequiredBurnAmount() const override {
        return 0.5 * COIN;  // 0.5 GRC burn fee
    }
    
    // Signature methods
    bool Sign(CKey& private_key);
    bool VerifySignature() const;
};
```

**Key Pattern**: Payload wraps entry data + adds signature for authentication

---

## 2. Registry Handler

### 2.1 BeaconRegistry Class

```cpp
class BeaconRegistry : public IContractHandler {
public:
    // Type definitions
    typedef std::unordered_map<Cpid, Beacon_ptr> BeaconMap;
    typedef std::map<CKeyID, Beacon_ptr> PendingBeaconMap;
    typedef std::map<uint256, Beacon_ptr> HistoricalBeaconMap;
    
    // IContractHandler interface
    void Reset() override;
    bool Validate(const Contract& contract, const CTransaction& tx, int& DoS) const override;
    bool BlockValidate(const ContractContext& ctx, int& DoS) const override;
    void Add(const ContractContext& ctx) override;
    void Delete(const ContractContext& ctx) override;
    void Revert(const ContractContext& ctx) override;
    
    // Query methods
    BeaconOption Try(const Cpid& cpid) const;
    BeaconOption TryActive(const Cpid& cpid, const int64_t now) const;
    bool ContainsActive(const Cpid& cpid, const int64_t now) const;
    std::vector<Beacon_ptr> FindPending(const Cpid& cpid) const;
    
    // Superblock integration
    void ActivatePending(const std::vector<uint160>& beacon_ids,
                         const int64_t superblock_time,
                         const uint256& block_hash,
                         const int& height);
    void Deactivate(const uint256 superblock_hash);
    
    // Database management
    int Initialize() override;
    int GetDBHeight() override;
    void SetDBHeight(int& height) override;
    uint64_t PassivateDB() override;
    
private:
    CCriticalSection cs_lock;           // Thread safety
    BeaconMap m_beacons;                // Active beacons by CPID
    PendingBeaconMap m_pending;         // Pending beacons by public key ID
    std::set<Beacon_ptr> m_expired_pending; // Expired pending beacons
    BeaconDB m_beacon_db;               // LevelDB storage
};
```

**Key Pattern**: Three-map system (active, pending, expired) + LevelDB backing

---

## 3. Core Operations

### 3.1 Add() - Advertisement

**Purpose**: Process new beacon advertisement or renewal

**Implementation Pattern** (`beacon.cpp`):
```cpp
void BeaconRegistry::Add(const ContractContext& ctx) {
    int height = ctx.m_pindex ? ctx.m_pindex->nHeight : -1;
    BeaconPayload payload = ctx->CopyPayloadAs<BeaconPayload>();
    
    // Set transaction metadata
    payload.m_beacon.m_timestamp = ctx.m_tx.nTime;
    payload.m_beacon.m_hash = ctx.m_tx.GetHash();
    
    // Check for existing beacon
    auto beacon_pair_iter = m_beacons.find(payload.m_cpid);
    bool current_beacon_present = (beacon_pair_iter != m_beacons.end());
    
    // Set previous hash link
    if (current_beacon_present) {
        payload.m_beacon.m_previous_hash = beacon_pair_iter->second->m_hash;
    }
    
    // Legacy v1 contracts: Direct activation
    if (ctx->m_version == 1) {
        Beacon historical(payload.m_beacon);
        historical.m_cpid = payload.m_cpid;
        historical.m_status = BeaconStatusForStorage::ACTIVE;
        m_beacon_db.insert(ctx.m_tx.GetHash(), height, historical);
        m_beacons[payload.m_cpid] = m_beacon_db.find(ctx.m_tx.GetHash())->second;
        return;
    }
    
    // Renewal attempt
    if (current_beacon_present && TryRenewal(current_beacon_ptr, height, payload)) {
        return;
    }
    
    // New beacon: Set to PENDING
    PendingBeacon pending(payload.m_cpid, std::move(payload.m_beacon));
    pending.m_status = BeaconStatusForStorage::PENDING;
    m_beacon_db.insert(ctx.m_tx.GetHash(), height, pending);
    m_pending[pending.GetId()] = m_beacon_db.find(ctx.m_tx.GetHash())->second;
}
```

**Key Steps**:
1. Extract payload from context
2. Set transaction metadata
3. Link to previous beacon (chainlet)
4. Check if renewal (same public key)
5. If renewal: Update existing with RENEWAL status
6. If new: Create PENDING entry
7. Store in LevelDB and appropriate map

### 3.2 TryRenewal() - Beacon Renewal

```cpp
bool BeaconRegistry::TryRenewal(Beacon_ptr& current_beacon_ptr,
                                int& height,
                                const BeaconPayload& payload) {
    // Check not expired
    if (current_beacon_ptr->Expired(payload.m_beacon.m_timestamp)) {
        return false;
    }
    
    // Check same public key
    if (current_beacon_ptr->m_public_key != payload.m_beacon.m_public_key) {
        return false;
    }
    
    // Create renewal beacon
    PendingBeacon renewal(payload.m_cpid, payload.m_beacon);
    renewal.m_status = BeaconStatusForStorage::RENEWAL;
    renewal.m_previous_hash = current_beacon_ptr->m_hash;
    
    // Store in DB
    m_beacon_db.insert(renewal.m_hash, height, renewal);
    
    // Replace in active map
    m_beacons[payload.m_cpid] = m_beacon_db.find(renewal.m_hash)->second;
    
    return true;
}
```

**Key Pattern**: Renewal creates new entry with RENEWAL status, links to previous

### 3.3 Delete() - Revocation

```cpp
void BeaconRegistry::Delete(const ContractContext& ctx) {
    int height = ctx.m_pindex ? ctx.m_pindex->nHeight : -1;
    const auto payload = ctx->SharePayloadAs<BeaconPayload>();
    
    // Remove from pending (v2+)
    if (ctx->m_version >= 2) {
        m_pending.erase(payload->m_beacon.GetId());
    }
    
    // Remove from active map
    auto iter = m_beacons.find(payload->m_cpid);
    uint256 last_active_ctx_hash;
    
    if (iter != m_beacons.end()) {
        last_active_ctx_hash = iter->second->m_hash;
        m_beacons.erase(payload->m_cpid);
    }
    
    // Create DELETED entry
    Beacon deleted_beacon(payload->m_beacon);
    deleted_beacon.m_cpid = payload->m_cpid;
    deleted_beacon.m_hash = ctx.m_tx.GetHash();
    deleted_beacon.m_previous_hash = last_active_ctx_hash;
    deleted_beacon.m_status = BeaconStatusForStorage::DELETED;
    
    // Store in DB
    m_beacon_db.insert(deleted_beacon.m_hash, height, deleted_beacon);
}
```

**Key Pattern**: Remove from maps but maintain DELETED record in DB

### 3.4 Revert() - Reorganization Handling

**Purpose**: Undo contract effects during blockchain reorg

**Implementation** (simplified):
```cpp
void BeaconRegistry::Revert(const ContractContext& ctx) {
    const auto payload = ctx->SharePayloadAs<BeaconPayload>();
    
    if (ctx->m_action == ContractAction::ADD) {
        // Revert advertisement
        if (ctx->m_version == 1) {
            // V1: Direct removal
            m_beacons.erase(payload->m_cpid);
            m_beacon_db.erase(ctx.m_tx.GetHash());
        } else {
            // V2+: Remove from pending
            auto pending_to_revert = m_pending.find(payload->m_beacon.GetId());
            if (pending_to_revert != m_pending.end()) {
                m_pending.erase(pending_to_revert);
                m_beacon_db.erase(ctx.m_tx.GetHash());
            }
            
            // Handle renewal reversion
            auto iter = m_beacons.find(payload->m_cpid);
            if (iter != m_beacons.end() && 
                iter->second->m_status == BeaconStatusForStorage::RENEWAL) {
                // Resurrect previous beacon
                uint256 resurrect_hash = iter->second->m_previous_hash;
                m_beacons.erase(iter);
                auto resurrect_iter = m_beacon_db.find(resurrect_hash);
                if (resurrect_iter != m_beacon_db.end()) {
                    m_beacons[payload->m_cpid] = resurrect_iter->second;
                }
                m_beacon_db.erase(ctx.m_tx.GetHash());
            }
        }
    }
    
    if (ctx->m_action == ContractAction::REMOVE) {
        // Revert deletion - resurrect previous state
        auto deleted_beacon_record = m_beacon_db.find(ctx.m_tx.GetHash());
        if (deleted_beacon_record != m_beacon_db.end()) {
            auto record_to_restore = m_beacon_db.find(
                deleted_beacon_record->second->m_previous_hash);
            if (record_to_restore != m_beacon_db.end()) {
                Beacon_ptr beacon_to_restore_ptr = record_to_restore->second;
                
                if (beacon_to_restore_ptr->m_status == BeaconStatusForStorage::ACTIVE ||
                    beacon_to_restore_ptr->m_status == BeaconStatusForStorage::RENEWAL) {
                    m_beacons[beacon_to_restore_ptr->m_cpid] = beacon_to_restore_ptr;
                } else if (beacon_to_restore_ptr->m_status == BeaconStatusForStorage::PENDING) {
                    m_pending[beacon_to_restore_ptr->GetId()] = beacon_to_restore_ptr;
                }
            }
        }
    }
}
```

**Key Pattern**: Use `m_previous_hash` to resurrect prior state from LevelDB

---

## 4. Superblock Integration

### 4.1 ActivatePending()

**Purpose**: Activate verified pending beacons when superblock commits

```cpp
void BeaconRegistry::ActivatePending(
    const std::vector<uint160>& beacon_ids,
    const int64_t superblock_time,
    const uint256& block_hash,
    const int& height)
{
    // Collect verified beacons
    BeaconMap verified_beacons;
    for (const auto& id : beacon_ids) {
        auto iter_pair = m_pending.find(id);
        if (iter_pair != m_pending.end()) {
            verified_beacons[iter_pair->second->m_cpid] = iter_pair->second;
        }
    }
    
    // Activate verified beacons
    for (const auto& iter_pair : verified_beacons) {
        Beacon activated_beacon(*iter_pair.second);
        activated_beacon.m_previous_hash = iter_pair.second->m_hash;
        activated_beacon.m_status = BeaconStatusForStorage::ACTIVE;
        activated_beacon.m_hash = Hash(block_hash, iter_pair.second->m_hash);
        
        m_beacon_db.insert(activated_beacon.m_hash, height, activated_beacon);
        m_beacons[activated_beacon.m_cpid] = m_beacon_db.find(activated_beacon.m_hash)->second;
        m_pending.erase(iter_pair.second->GetId());
    }
    
    // Expire unverified pending beacons
    m_expired_pending.clear();
    for (auto iter = m_pending.begin(); iter != m_pending.end(); ) {
        PendingBeacon pending_beacon(*iter->second);
        if (pending_beacon.PendingExpired(superblock_time)) {
            pending_beacon.m_previous_hash = pending_beacon.m_hash;
            pending_beacon.m_status = BeaconStatusForStorage::EXPIRED_PENDING;
            pending_beacon.m_hash = Hash(block_hash, pending_beacon.m_hash);
            
            m_beacon_db.insert(pending_beacon.m_hash, height, pending_beacon);
            m_expired_pending.insert(m_beacon_db.find(pending_beacon.m_hash)->second);
            iter = m_pending.erase(iter);
        } else {
            ++iter;
        }
    }
}
```

**Key Pattern**: Batch activation at superblock boundaries

### 4.2 Deactivate()

**Purpose**: Revert superblock activation during reorg

```cpp
void BeaconRegistry::Deactivate(const uint256 superblock_hash) {
    // Revert activated beacons to pending
    for (auto iter = m_beacons.begin(); iter != m_beacons.end();) {
        uint256 activation_hash = Hash(superblock_hash, iter->second->m_previous_hash);
        if (iter->second->m_hash == activation_hash) {
            Cpid cpid = iter->second->m_cpid;
            auto pending_beacon_entry = m_beacon_db.find(iter->second->m_previous_hash);
            if (pending_beacon_entry != m_beacon_db.end()) {
                m_pending[pending_beacon_entry->second->GetId()] = pending_beacon_entry->second;
            }
            iter = m_beacons.erase(iter);
            m_beacon_db.erase(activation_hash);
        } else {
            ++iter;
        }
    }
    
    // Resurrect expired pending beacons
    for (const auto& iter : m_expired_pending) {
        auto pending_beacon_entry = m_beacon_db.find(iter->m_previous_hash);
        m_pending.insert(std::make_pair(pending_beacon_entry->second->GetId(),
                                        pending_beacon_entry->second));
    }
    m_expired_pending.clear();
}
```

---

## 5. Validation Logic

### 5.1 Validate() Method

```cpp
bool BeaconRegistry::Validate(const Contract& contract,
                              const CTransaction& tx,
                              int& DoS) const {
    // Skip legacy v1 contracts
    if (contract.m_version <= 1) return true;
    
    const auto payload = contract.SharePayloadAs<BeaconPayload>();
    
    // Version check
    if (payload->m_version < 2) {
        DoS = 25;
        return false;
    }
    
    // Well-formed check
    if (!payload->WellFormed(contract.m_action.Value())) {
        DoS = 25;
        return false;
    }
    
    // Signature verification
    if (!payload->VerifySignature()) {
        DoS = 25;
        return false;
    }
    
    const BeaconOption current_beacon = Try(payload->m_cpid);
    
    // No existing beacon or expired: Allow
    if (!current_beacon || current_beacon->Expired(tx.nTime)) {
        return true;
    }
    
    // Beacon removal: Verify signature matches
    if (contract.m_action == ContractAction::REMOVE) {
        if (current_beacon->m_public_key != payload->m_beacon.m_public_key) {
            DoS = 25;
            return false;
        }
        return true;
    }
    
    // Beacon replacement with different key: Allowed (scrapers verify)
    if (current_beacon->m_public_key != payload->m_beacon.m_public_key) {
        return true;
    }
    
    // Renewal validation
    if (current_beacon->m_timestamp <= g_v11_timestamp) {
        DoS = 25;
        return false; // Can't renew legacy beacons
    }
    
    if (!current_beacon->Renewable(tx.nTime)) {
        DoS = 25;
        return false;
    }
    
    return true;
}
```

**Validation Checks**:
1. ✅ Version compatibility
2. ✅ Payload well-formed
3. ✅ Signature valid
4. ✅ Context-specific rules (renewal, removal, replacement)

---

## 6. LevelDB Integration

### 6.1 RegistryDB Template

**Type Definition**:
```cpp
typedef RegistryDB<Beacon,
                   StorageBeacon,
                   BeaconStatusForStorage,
                   BeaconMap,
                   PendingBeaconMap,
                   std::set<Beacon_ptr>,
                   HistoricalBeaconMap> BeaconDB;
```

**Key Operations**:
- `insert(hash, height, entry)` - Store entry in DB
- `erase(hash)` - Remove entry from DB
- `find(hash)` - Retrieve entry (auto-loads from LevelDB if needed)
- `passivate_db()` - Free memory for unused historical entries
- `Initialize(maps...)` - Load registry state from LevelDB on startup

### 6.2 DB Height Tracking

```cpp
int BeaconRegistry::GetDBHeight() {
    int height = 0;
    m_beacon_db.LoadDBHeight(height);
    return height;
}

void BeaconRegistry::SetDBHeight(int& height) {
    m_beacon_db.StoreDBHeight(height);
}
```

**Purpose**: Track sync state for registry initialization

---

## 7. RPC Commands

### 7.1 advertisebeacon

**Location**: `src/rpc/blockchain.cpp`

**Function Signature**:
```cpp
UniValue advertisebeacon(const UniValue& params, bool fHelp)
```

**Parameters**:
- `force` (optional, boolean) - Force new beacon even if one exists

**Process**:
1. Get researcher context
2. Check eligibility (valid CPID)
3. Call `researcher->AdvertiseBeacon(force)`
4. Return result (success or error code)

### 7.2 beaconstatus

**Function Signature**:
```cpp
UniValue beaconstatus(const UniValue& params, bool fHelp)
```

**Parameters**:
- `cpid` (optional, string) - Query specific CPID

**Returns**:
- Current beacon status
- Pending beacons
- Beacon age and renewal eligibility

### 7.3 revokebeacon

**Function Signature**:
```cpp
UniValue revokebeacon(const UniValue& params, bool fHelp)
```

**Parameters**:
- `cpid` (required, string) - CPID of beacon to revoke

**Process**:
1. Validate CPID
2. Check wallet has private key
3. Create DELETE contract
4. Broadcast transaction

---

## 8. Thread Safety

### 8.1 Lock Usage

```cpp
class BeaconRegistry : public IContractHandler {
private:
    mutable CCriticalSection cs_lock; // Protects registry access
    // ...
};
```

**Lock Order**:
1. `cs_main` (blockchain state)
2. `pwalletMain->cs_wallet` (wallet state)
3. `cs_lock` (registry internal - managed automatically)

**Pattern**: Registry methods acquire internal lock automatically, callers must hold cs_main

---

## 9. Key Patterns for Pool Implementation

### Pattern 1: Entry → Payload → Registry Structure
```
PoolEntry (data) → PoolPayload (contract) → PoolRegistry (handler)
```

### Pattern 2: Status Lifecycle Management
```
Define clear status enum → Track transitions → Handle each state
```

### Pattern 3: Chainlet Linking
```
Use m_previous_hash to link entries → Enables reversion and history
```

### Pattern 4: Three-Map System
```
Active map (by key) + Pending map (by public key) + Historical DB
```

### Pattern 5: LevelDB Integration
```
RegistryDB template → Automatic persistence → Passivation for memory
```

### Pattern 6: Signature Verification
```
Payload includes signature → Verify in Validate() → Authenticate operations
```

### Pattern 7: Reversion Support
```
Store previous state reference → Implement Revert() → Test reorgs thoroughly
```

---

## 10. Common Pitfalls & Solutions

### Pitfall 1: Lock Order Deadlocks
**Solution**: Always acquire cs_main before registry locks

### Pitfall 2: Memory Leaks with Shared Pointers
**Solution**: Use RegistryDB passivation, maintain minimal references

### Pitfall 3: Reorg Corruption
**Solution**: Robust Revert() implementation, use m_previous_hash links

### Pitfall 4: Signature Verification Bypass
**Solution**: Verify in both Validate() and BlockValidate()

### Pitfall 5: DB Height Desync
**Solution**: Properly track and update DB height in all operations

---

## 11. Testing Checklist

- [ ] Entry serialization/deserialization
- [ ] Payload signature generation and verification
- [ ] Registry Add() with new entries
- [ ] Registry Add() with updates/renewals
- [ ] Registry Delete() operation
- [ ] Registry Revert() for ADD actions
- [ ] Registry Revert() for REMOVE actions
- [ ] Validation with well-formed payloads
- [ ] Validation with malformed payloads
- [ ] Validation with invalid signatures
- [ ] RPC command functionality
- [ ] LevelDB persistence across restarts
- [ ] Passivation and memory management
- [ ] Concurrent access thread safety
- [ ] Blockchain reorganization scenarios

---

## References

- **Beacon Header**: `src/gridcoin/beacon.h`
- **Beacon Implementation**: `src/gridcoin/beacon.cpp`
- **Registry DB Template**: `src/gridcoin/contract/registry_db.h`
- **Contract Handler Interface**: `src/gridcoin/contract/handler.h`
- **RPC Commands**: `src/rpc/blockchain.cpp`

---

**Last Updated**: 2025-12-26  
**Referenced By**: Issue #1783 Implementation  
**See Also**: `06-issue-1783-pool-contract-analysis.md`, `08-pool-contract-implementation-steps.md`
