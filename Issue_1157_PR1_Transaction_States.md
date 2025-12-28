# Issue #1157 - PR1: Transaction State Infrastructure
## Implementation Guide

**PR Title**: Add Transaction State Types for Wallet Synchronization  
**Dependencies**: None  
**Estimated Effort**: 10 days (2 weeks)  
**Complexity**: Medium  
**Risk Level**: Low

---

## Overview

This PR creates the foundational infrastructure for explicit transaction state tracking in the Gridcoin wallet. It introduces Bitcoin Core's transaction state pattern, which will enable the wallet to properly track transactions in different states (mempool, confirmed, inactive).

### What This PR Accomplishes

✅ Creates transaction state type definitions  
✅ Updates `CWalletTx` with state member  
✅ Modifies `GetDepthInMainChain()` to use states  
✅ Adds state helper methods  
✅ Updates serialization for backward compatibility  
✅ Comprehensive unit tests

### What This PR Does NOT Do

❌ Does not change how transactions are added to wallet  
❌ Does not implement SyncTransaction mechanism  
❌ Does not modify validation layer  
❌ Does not affect user-visible behavior (yet)

---

## Table of Contents

1. [Prerequisites](#prerequisites)
2. [File Structure](#file-structure)
3. [Implementation Steps](#implementation-steps)
4. [Testing Strategy](#testing-strategy)
5. [Review Criteria](#review-criteria)
6. [Rollback Plan](#rollback-plan)

---

## Prerequisites

### Knowledge Requirements

- C++ (intermediate to advanced)
- Understanding of std::variant
- Gridcoin wallet architecture
- Transaction lifecycle concepts

### Tools Needed

- CMake build system
- Boost test framework
- Git for version control

### Before Starting

1. ✅ Read `Issue_1157_Overview.md`
2. ✅ Understand Bitcoin's transaction states
3. ✅ Familiar with CWalletTx structure
4. ✅ Development environment set up

---

## File Structure

### New Files to Create

```
src/wallet/
└── transaction.h     (NEW) - Transaction state type definitions
```

### Files to Modify

```
src/wallet/
├── wallet.h          (MODIFY) - Add state to CWalletTx
└── wallet.cpp        (MODIFY) - State helper implementations

src/
├── main.h            (MODIFY) - Update GetDepthInMainChain signature
└── main.cpp          (MODIFY) - Reimplement GetDepthInMainChain

src/test/
└── wallet_tests.cpp  (MODIFY) - Add state tests
```

---

## Implementation Steps

### Step 1: Create Transaction State Types (Day 1-2)

#### 1.1 Create `src/wallet/transaction.h`

**Objective**: Define all transaction state types

**Implementation**:

```cpp
// src/wallet/transaction.h
#ifndef BITCOIN_WALLET_TRANSACTION_H
#define BITCOIN_WALLET_TRANSACTION_H

#include "uint256.h"
#include <variant>

namespace wallet {

/**
 * State of a transaction in the wallet
 */

//! Transaction is unconfirmed and in the mempool
struct TxStateInMempool {
    // No additional data needed for mempool state
};

//! Transaction is confirmed in a block
struct TxStateConfirmed {
    uint256 confirmed_block_hash;
    int confirmed_block_height;
    int position_in_block;

    TxStateConfirmed() 
        : confirmed_block_height(-1), position_in_block(-1) {}
    
    TxStateConfirmed(const uint256& hash, int height, int pos)
        : confirmed_block_hash(hash)
        , confirmed_block_height(height)
        , position_in_block(pos) 
    {
    }

    ADD_SERIALIZE_METHODS;
    template <typename Stream, typename Operation>
    inline void SerializationOp(Stream& s, Operation ser_action)
    {
        READWRITE(confirmed_block_hash);
        READWRITE(confirmed_block_height);
        READWRITE(position_in_block);
    }
};

//! Transaction is not active (conflicted or abandoned)
struct TxStateInactive {
    bool abandoned;

    TxStateInactive() : abandoned(false) {}
    explicit TxStateInactive(bool _abandoned) : abandoned(_abandoned) {}

    ADD_SERIALIZE_METHODS;
    template <typename Stream, typename Operation>
    inline void SerializationOp(Stream& s, Operation ser_action)
    {
        READWRITE(abandoned);
    }
};

//! Transaction state is unrecognized (for backward compatibility)
struct TxStateUnrecognized {
    uint256 block_hash;
    int index;

    TxStateUnrecognized() : index(-1) {}

    ADD_SERIALIZE_METHODS;
    template <typename Stream, typename Operation>
    inline void SerializationOp(Stream& s, Operation ser_action)
    {
        READWRITE(block_hash);
        READWRITE(index);
    }
};

//! Variant type holding any transaction state
using TxState = std::variant<
    TxStateInMempool,
    TxStateConfirmed,
    TxStateInactive,
    TxStateUnrecognized
>;

} // namespace wallet

#endif // BITCOIN_WALLET_TRANSACTION_H
```

**Baby Step Checklist**:
- [ ] Create the file
- [ ] Add includes and namespace
- [ ] Define TxStateInMempool struct
- [ ] Define TxStateConfirmed struct with serialization
- [ ] Define TxStateInactive struct with serialization
- [ ] Define TxStateUnrecognized struct
- [ ] Define TxState variant
- [ ] Compile to verify no syntax errors

---

### Step 2: Update CWalletTx Structure (Day 2-3)

#### 2.1 Add State Member to CWalletTx

**File**: `src/wallet/wallet.h`

**Find**:
```cpp
class CWalletTx : public CMerkleTx
{
private:
    const CWallet* pwallet;

public:
    mapValue_t mapValue;
    std::vector<std::pair<std::string, std::string> > vOrderForm;
    unsigned int fTimeReceivedIsTxTime;
    unsigned int nTimeReceived;
    // ... existing members ...
```

**Add After Existing Members**:
```cpp
    /** 
     * New transaction state tracking
     * Replaces reliance on hashBlock for state determination
     */
    wallet::TxState m_state;
```

#### 2.2 Add State Helper Methods

**File**: `src/wallet/wallet.h`

**Add to CWalletTx class**:
```cpp
public:
    /**
     * Get transaction state of specific type
     * @return Pointer to state if type matches, nullptr otherwise
     */
    template<typename T>
    const T* state() const {
        return std::get_if<T>(&m_state);
    }

    template<typename T>
    T* state() {
        return std::get_if<T>(&m_state);
    }

    /**
     * Check if transaction is confirmed
     */
    bool isConfirmed() const {
        return std::holds_alternative<wallet::TxStateConfirmed>(m_state);
    }

    /**
     * Check if transaction is in mempool
     */
    bool isInMempool() const {
        return std::holds_alternative<wallet::TxStateInMempool>(m_state);
    }

    /**
     * Check if transaction is inactive/conflicted
     */
    bool isInactive() const {
        return std::holds_alternative<wallet::TxStateInactive>(m_state);
    }

    /**
     * Check if transaction is unrecognized (old format)
     */
    bool isUnrecognized() const {
        return std::holds_alternative<wallet::TxStateUnrecognized>(m_state);
    }
```

**Baby Step Checklist**:
- [ ] Add m_state member to CWalletTx
- [ ] Add state<T>() template methods
- [ ] Add isConfirmed() helper
- [ ] Add isInMempool() helper
- [ ] Add isInactive() helper
- [ ] Add isUnrecognized() helper
- [ ] Compile to verify

---

### Step 3: Update Serialization (Day 3-4)

#### 3.1 Modify CWalletTx Serialization

**File**: `src/wallet/wallet.h`

**Find the SerializationOp method in CWalletTx**:
```cpp
template <typename Stream, typename Operation>
inline void SerializationOp(Stream& s, Operation ser_action)
{
    // ... existing serialization code ...
}
```

**Add Before the Closing Brace**:
```cpp
    // Serialize transaction state (version-gated for backward compatibility)
    if (s.GetVersion() >= FEATURE_TRANSACTION_STATES_VERSION) {
        // Determine state type for serialization
        int state_type = m_state.index();
        READWRITE(state_type);
        
        switch (state_type) {
            case 0: { // TxStateInMempool
                // No additional data to serialize
                if (ser_action.ForRead()) {
                    m_state = wallet::TxStateInMempool{};
                }
                break;
            }
            case 1: { // TxStateConfirmed
                wallet::TxStateConfirmed conf;
                if (ser_action.ForRead()) {
                    READWRITE(conf);
                    m_state = conf;
                } else {
                    conf = *std::get_if<wallet::TxStateConfirmed>(&m_state);
                    READWRITE(conf);
                }
                break;
            }
            case 2: { // TxStateInactive
                wallet::TxStateInactive inactive;
                if (ser_action.ForRead()) {
                    READWRITE(inactive);
                    m_state = inactive;
                } else {
                    inactive = *std::get_if<wallet::TxStateInactive>(&m_state);
                    READWRITE(inactive);
                }
                break;
            }
            case 3: { // TxStateUnrecognized
                wallet::TxStateUnrecognized unrec;
                if (ser_action.ForRead()) {
                    READWRITE(unrec);
                    m_state = unrec;
                } else {
                    unrec = *std::get_if<wallet::TxStateUnrecognized>(&m_state);
                    READWRITE(unrec);
                }
                break;
            }
        }
    } else if (ser_action.ForRead()) {
        // Reading old wallet format - migrate to new state representation
        // If hashBlock is set, it's confirmed; otherwise unrecognized
        if (!hashBlock.IsNull()) {
            m_state = wallet::TxStateConfirmed(hashBlock, 0, nIndex);
        } else {
            m_state = wallet::TxStateUnrecognized{};
        }
    }
```

**Add to clientversion.h**:
```cpp
// In src/clientversion.h or appropriate location
static const int FEATURE_TRANSACTION_STATES_VERSION = 169900; // Version that introduced tx states
```

**Baby Step Checklist**:
- [ ] Add version constant
- [ ] Add state serialization logic
- [ ] Add migration logic for old wallets
- [ ] Test serialization roundtrip
- [ ] Verify old wallet.dat files load correctly

---

### Step 4: Update GetDepthInMainChain (Day 4-6)

#### 4.1 Reimplement GetDepthInMainChainINTERNAL

**File**: `src/main.cpp`

**Find**:
```cpp
int CMerkleTx::GetDepthInMainChainINTERNAL(CBlockIndex* &pindexRet) const
{
    // ... existing implementation ...
}
```

**Replace With**:
```cpp
int CMerkleTx::GetDepthInMainChainINTERNAL(CBlockIndex* &pindexRet) const
{
    // For CWalletTx, use state-based depth calculation
    const CWalletTx* pWalletTx = dynamic_cast<const CWalletTx*>(this);
    if (pWalletTx) {
        // Check if transaction is confirmed
        if (const auto* conf = pWalletTx->state<wallet::TxStateConfirmed>()) {
            auto it = mapBlockIndex.find(conf->confirmed_block_hash);
            if (it != mapBlockIndex.end()) {
                pindexRet = it->second;
                if (pindexRet && pindexRet->IsInMainChain()) {
                    return nBestHeight - pindexRet->nHeight + 1;
                }
                // Block exists but not in main chain (reorg scenario)
                return 0;
            }
            // Block not found - shouldn't happen but handle gracefully
            return 0;
        }
        
        // Check if in mempool
        if (pWalletTx->isInMempool()) {
            pindexRet = nullptr;
            return 0; // Mempool = 0 depth
        }
        
        // Inactive or unrecognized
        pindexRet = nullptr;
        return -1; // Conflicted/abandoned
    }
    
    // For non-wallet transactions, use legacy logic
    if (hashBlock.IsNull() || nIndex == -1) {
        return 0;
    }
    
    auto mi = mapBlockIndex.find(hashBlock);
    if (mi == mapBlockIndex.end()) {
        return 0;
    }
    
    pindexRet = mi->second;
    if (!pindexRet || !pindexRet->IsInMainChain()) {
        return 0;
    }
    
    return nBestHeight - pindexRet->nHeight + 1;
}
```

#### 4.2 Update GetDepthInMainChain Wrapper

**File**: `src/main.cpp`

**Keep the wrapper but verify mempool logic**:
```cpp
int CMerkleTx::GetDepthInMainChain(CBlockIndex* &pindexRet) const
{
    AssertLockHeld(cs_main);
    int nResult = GetDepthInMainChainINTERNAL(pindexRet);
    
    // For mempool transactions (depth = 0), verify still in mempool
    if (nResult == 0 && !mempool.exists(GetHash())) {
        return -1; // Was in mempool but removed
    }
    
    return nResult;
}
```

**Baby Step Checklist**:
- [ ] Modify GetDepthInMainChainINTERNAL
- [ ] Add state-based logic for CWalletTx
- [ ] Keep legacy logic for non-wallet tx
- [ ] Update wrapper method
- [ ] Compile and verify no regressions
- [ ] Test with existing transactions

---

### Step 5: Initialize States for Existing Code (Day 6-7)

#### 5.1 Add Constructor Initialization

**File**: `src/wallet/wallet.cpp`

**Find CWalletTx constructors and ensure state is initialized**:

```cpp
CWalletTx::CWalletTx()
{
    Init(nullptr);
    m_state = wallet::TxStateUnrecognized{}; // Default to unrecognized
}

CWalletTx::CWalletTx(const CWallet* pwalletIn, const CTransaction& txIn) 
    : CMerkleTx(txIn)
{
    Init(pwalletIn);
    m_state = wallet::TxStateUnrecognized{}; // Will be set properly later
}
```

#### 5.2 Update Existing AddToWallet Temporarily

**File**: `src/wallet/wallet.cpp`

**In AddToWallet method, add state initialization**:

**Find**:
```cpp
bool CWallet::AddToWallet(const CWalletTx& wtxIn, bool fFromLoadWallet)
{
    // ... existing code ...
```

**Add After Transaction is Added/Updated**:
```cpp
    // Temporarily initialize state based on existing hashBlock
    // (This will be replaced by SyncTransaction in PR2)
    if (!wtx.hashBlock.IsNull()) {
        // Transaction is in a block
        auto it = mapBlockIndex.find(wtx.hashBlock);
        if (it != mapBlockIndex.end()) {
            CBlockIndex* pindex = it->second;
            wtx.m_state = wallet::TxStateConfirmed(
                wtx.hashBlock,
                pindex->nHeight,
                wtx.nIndex
            );
        } else {
            wtx.m_state = wallet::TxStateUnrecognized{};
        }
    } else {
        // Transaction not in block - could be mempool or inactive
        if (mempool.exists(wtx.GetHash())) {
            wtx.m_state = wallet::TxStateInMempool{};
        } else {
            wtx.m_state = wallet::TxStateUnrecognized{};
        }
    }
```

**Baby Step Checklist**:
- [ ] Initialize state in constructors
- [ ] Add state initialization in AddToWallet
- [ ] Test with new transactions
- [ ] Test with existing wallet.dat
- [ ] Verify no behavior changes

---

## Testing Strategy

### Unit Tests to Add

**File**: `src/test/wallet_tests.cpp`

**Add Test Suite**:

```cpp
#include "wallet/transaction.h"

BOOST_AUTO_TEST_SUITE(wallet_state_tests)

BOOST_AUTO_TEST_CASE(state_type_creation)
{
    // Test creating each state type
    wallet::TxStateInMempool mempool_state;
    BOOST_CHECK(std::holds_alternative<wallet::TxStateInMempool>(
        wallet::TxState{mempool_state}));
    
    wallet::TxStateConfirmed conf_state(uint256S("00"), 100, 0);
    BOOST_CHECK_EQUAL(conf_state.confirmed_block_height, 100);
    
    wallet::TxStateInactive inactive_state(true);
    BOOST_CHECK_EQUAL(inactive_state.abandoned, true);
}

BOOST_AUTO_TEST_CASE(state_serialization)
{
    // Test each state type serializes/deserializes correctly
    {
        wallet::TxStateConfirmed original(uint256S("0123"), 12345, 7);
        CDataStream ss(SER_DISK, CLIENT_VERSION);
        ss << original;
        
        wallet::TxStateConfirmed restored;
        ss >> restored;
        
        BOOST_CHECK(original.confirmed_block_hash == restored.confirmed_block_hash);
        BOOST_CHECK_EQUAL(original.confirmed_block_height, restored.confirmed_block_height);
        BOOST_CHECK_EQUAL(original.position_in_block, restored.position_in_block);
    }
}

BOOST_AUTO_TEST_CASE(wallet_tx_state_helpers)
{
    CWalletTx wtx;
    
    // Test mempool state
    wtx.m_state = wallet::TxStateInMempool{};
    BOOST_CHECK(wtx.isInMempool());
    BOOST_CHECK(!wtx.isConfirmed());
    BOOST_CHECK(!wtx.isInactive());
    
    // Test confirmed state
    wtx.m_state = wallet::TxStateConfirmed(uint256S("00"), 100, 0);
    BOOST_CHECK(!wtx.isInMempool());
    BOOST_CHECK(wtx.isConfirmed());
    BOOST_CHECK(!wtx.isInactive());
    
    auto* conf = wtx.state<wallet::TxStateConfirmed>();
    BOOST_CHECK(conf != nullptr);
    BOOST_CHECK_EQUAL(conf->confirmed_block_height, 100);
}

BOOST_AUTO_TEST_CASE(wallet_tx_serialization_with_state)
{
    // Test that CWalletTx with state serializes correctly
    CWalletTx original;
    original.m_state = wallet::TxStateConfirmed(uint256S("abc"), 500, 3);
    
    CDataStream ss(SER_DISK, CLIENT_VERSION);
    ss << original;
    
    CWalletTx restored;
    ss >> restored;
    
    BOOST_CHECK(restored.isConfirmed());
    auto* conf = restored.state<wallet::TxStateConfirmed>();
    BOOST_CHECK(conf != nullptr);
    BOOST_CHECK_EQUAL(conf->confirmed_block_height, 500);
}

BOOST_AUTO_TEST_CASE(old_wallet_migration)
{
    // Test that old wallet format migrates to new state
    CWalletTx oldFormat;
    oldFormat.hashBlock = uint256S("def");
    oldFormat.nIndex = 5;
    
    // Serialize with old version
    CDataStream ss(SER_DISK, FEATURE_TRANSACTION_STATES_VERSION - 1);
    ss << oldFormat;
    
    // Deserialize with new version
    CWalletTx newFormat;
    ss.SetVersion(CLIENT_VERSION);
    ss >> newFormat;
    
    // Should have migrated to confirmed state
    BOOST_CHECK(newFormat.isConfirmed() || newFormat.isUnrecognized());
}

BOOST_AUTO_TEST_SUITE_END()
```

### Manual Testing Checklist

- [ ] Load existing wallet.dat file
- [ ] Verify all transactions appear correctly
- [ ] Check balance calculations unchanged
- [ ] Create new transaction (outgoing)
- [ ] Verify new transaction has proper state
- [ ] Wait for confirmation
- [ ] Verify state transitions to confirmed
- [ ] Check debug.log for errors

---

## Review Criteria

### Code Quality
- [ ] Follows Gridcoin coding standards
- [ ] Clear comments explaining state purpose
- [ ] No memory leaks (use valgrind if available)
- [ ] Proper const correctness

### Functionality
- [ ] All unit tests pass
- [ ] Backward compatibility maintained
- [ ] Old wallets load without errors
- [ ] No changes to user-visible behavior

### Documentation
- [ ] Code comments explain state transitions
- [ ] Doxygen comments for public methods
- [ ] This implementation guide updated if needed

### Performance
- [ ] No significant slowdown in wallet operations
- [ ] State checks are O(1) operations
- [ ] Serialization size reasonable

---

## Rollback Plan

### If Critical Issues Found

1. **Revert Commit**:
   ```bash
   git revert <commit-hash>
   ```

2. **Manual Rollback**:
   - Remove `src/wallet/transaction.h`
   - Remove state member from `CWalletTx`
   - Restore original `GetDepthInMainChain()`
   - Remove test additions

3. **Database Compatibility**:
   - Old wallet.dat files work with or without this change
   - New wallet.dat with states will fall back to unrecognized state on old client

---

## Success Metrics

✅ **Completion Checklist**:
- [ ] All new files created
- [ ] All modified files updated
- [ ] All unit tests passing
- [ ] Manual testing completed
- [ ] Code review approved
- [ ] No performance regressions
- [ ] Documentation complete

✅ **Ready for PR2 When**:
- All above items checked
- No open review comments
- CI/CD pipeline green

---

## Next Steps

After this PR is merged:
1. Begin PR2: SyncTransaction Implementation
2. Use these state types in actual transaction synchronization
3. Wire up validation callbacks

---

## Notes

- This PR is intentionally minimal and non-breaking
- Changes are additive, not destructive
- Actual behavior change comes in PR2 and PR3
- Focus on getting infrastructure right before using it

---

**Document Version**: 1.0  
**Created**: December 28, 2025  
**Status**: Ready for Implementation
