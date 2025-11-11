# Sqoin Core - Complete Consensus Logic Documentation

## Deep Research into Consensus Implementation

This document provides an exhaustive analysis of Sqoin Core's consensus logic, derived from deep research into the codebase. It explains every consensus rule, validation step, formula, and algorithm used to maintain network agreement.

---

## Table of Contents

1. [Consensus Overview](#1-consensus-overview)
2. [Proof-of-Work Logic](#2-proof-of-work-logic)
3. [Block Validation Pipeline](#3-block-validation-pipeline)
4. [Transaction Validation Logic](#4-transaction-validation-logic)
5. [UTXO Verification](#5-utxo-verification)
6. [Script Validation](#6-script-validation)
7. [Block Reward & Subsidy](#7-block-reward--subsidy)
8. [Time-Lock Mechanisms](#8-time-lock-mechanisms)
9. [Signature Verification](#9-signature-verification)
10. [Consensus State Transitions](#10-consensus-state-transitions)
11. [Fork Detection & Resolution](#11-fork-detection--resolution)
12. [Consensus Parameters](#12-consensus-parameters)

---

## 1. Consensus Overview

### 1.1 What is Consensus?

**Consensus** in Sqoin is the mechanism by which all nodes in the network agree on:
- Which blocks are valid
- The order of transactions
- The current state of the blockchain
- The UTXO set (all spendable coins)

### 1.2 Consensus Components

```
┌─────────────────────────────────────────┐
│         CONSENSUS LAYER                  │
├─────────────────────────────────────────┤
│                                          │
│  ┌──────────────┐    ┌──────────────┐  │
│  │  Proof of    │    │   Block      │  │
│  │    Work      │───▶│  Validation  │  │
│  └──────────────┘    └──────────────┘  │
│         │                    │          │
│         ▼                    ▼          │
│  ┌──────────────┐    ┌──────────────┐  │
│  │ Difficulty   │    │ Transaction  │  │
│  │ Adjustment   │    │  Validation  │  │
│  └──────────────┘    └──────────────┘  │
│                             │          │
│                             ▼          │
│                      ┌──────────────┐  │
│                      │    UTXO      │  │
│                      │  Validation  │  │
│                      └──────────────┘  │
└─────────────────────────────────────────┘
```

### 1.3 Consensus vs Policy

**Consensus Rules** (MUST be followed):
- Block reward amount
- Proof-of-work difficulty
- Transaction structure
- Script execution
- Block size limits

**Policy Rules** (CAN vary between nodes):
- Minimum transaction fee
- Mempool size limits
- Transaction relay rules
- RBF preferences

---

## 2. Proof-of-Work Logic

### 2.1 Core PoW Algorithm

**Location**: `src/pow.cpp`

**Function**: `CheckProofOfWork()`

```cpp
bool CheckProofOfWork(uint256 hash, unsigned int nBits, const Consensus::Params& params)
{
    // Check if hash meets difficulty target
    bool fNegative;
    bool fOverflow;
    arith_uint256 bnTarget;
    
    bnTarget.SetCompact(nBits, &fNegative, &fOverflow);
    
    // Check range
    if (fNegative || bnTarget == 0 || fOverflow || bnTarget > UintToArith256(params.powLimit))
        return false;
    
    // Check proof of work matches claimed amount
    if (UintToArith256(hash) > bnTarget)
        return false;
    
    return true;
}
```

**Logic**:
1. Convert compact difficulty bits (`nBits`) to full 256-bit target
2. Verify target is within valid range
3. Check if block hash is less than or equal to target
4. Lower hash value = harder to find = more work done

**Formula**:
```
Block is valid IF:  SHA256(SHA256(block_header)) ≤ Target

Where Target is derived from nBits:
Target = Coefficient × 2^(8×(Exponent-3))
```

### 2.2 Difficulty Adjustment

**Location**: `src/pow.cpp`

**Function**: `GetNextWorkRequired()`

**Algorithm**:
```cpp
unsigned int GetNextWorkRequired(const CBlockIndex* pindexLast, const CBlockHeader *pblock, const Consensus::Params& params)
{
    // Only adjust every DifficultyAdjustmentInterval blocks
    if ((pindexLast->nHeight+1) % params.DifficultyAdjustmentInterval() != 0)
    {
        // Not adjustment time, return previous difficulty
        return pindexLast->nBits;
    }
    
    // Find first block of current period
    int nHeightFirst = pindexLast->nHeight - (params.DifficultyAdjustmentInterval()-1);
    const CBlockIndex* pindexFirst = pindexLast->GetAncestor(nHeightFirst);
    
    // Calculate new difficulty
    return CalculateNextWorkRequired(pindexLast, pindexFirst->GetBlockTime(), params);
}
```

**Calculation Logic**:
```cpp
unsigned int CalculateNextWorkRequired(const CBlockIndex* pindexLast, int64_t nFirstBlockTime, const Consensus::Params& params)
{
    // Calculate actual time taken
    int64_t nActualTimespan = pindexLast->GetBlockTime() - nFirstBlockTime;
    
    // Limit adjustment to 4x change (prevent wild swings)
    if (nActualTimespan < params.nPowTargetTimespan/4)
        nActualTimespan = params.nPowTargetTimespan/4;
    if (nActualTimespan > params.nPowTargetTimespan*4)
        nActualTimespan = params.nPowTargetTimespan*4;
    
    // Retarget formula
    arith_uint256 bnNew;
    bnNew.SetCompact(pindexLast->nBits);
    bnNew *= nActualTimespan;
    bnNew /= params.nPowTargetTimespan;
    
    // Ensure within limits
    if (bnNew > bnPowLimit)
        bnNew = bnPowLimit;
    
    return bnNew.GetCompact();
}
```

**Mathematical Formula**:
```
New_Difficulty = Old_Difficulty × (Actual_Time / Expected_Time)

Where:
- Expected_Time = 2 weeks = 2016 blocks × 10 minutes
- Actual_Time = time to mine last 2016 blocks
- Constraint: 0.25 ≤ (Actual_Time / Expected_Time) ≤ 4.0
```

**Example**:
```
If 2016 blocks took 1 week (too fast):
New_Difficulty = Old × (1 week / 2 weeks) = Old × 0.5
Result: Difficulty DECREASES (easier mining)

If 2016 blocks took 4 weeks (too slow):
New_Difficulty = Old × (4 weeks / 2 weeks) = Old × 2.0
Result: Difficulty INCREASES (harder mining)
```

### 2.3 Difficulty Bits Encoding

**Compact Format** (`nBits`):
```
Format: 0x1d00ffff (example)
        ^^---------- Exponent (0x1d = 29)
          ^^^^^^---- Coefficient (0x00ffff)

Target = 0x00ffff × 2^(8×(29-3))
       = 0x00ffff × 2^208
       = 0x00000000ffff0000000000000000000000000000000000000000000000000000
```

---

## 3. Block Validation Pipeline

### 3.1 Validation Stages

**Location**: `src/validation.cpp`

```
Block Received
    │
    ▼
┌──────────────────────┐
│ CheckBlockHeader()   │ ◀── PoW, version, timestamp
└──────────────────────┘
    │ PASS
    ▼
┌──────────────────────┐
│ CheckBlock()         │ ◀── Merkle root, tx format, size
└──────────────────────┘
    │ PASS
    ▼
┌──────────────────────┐
│ ContextualCheckBlock()│ ◀── Height, time, witnesses
└──────────────────────┘
    │ PASS
    ▼
┌──────────────────────┐
│ AcceptBlock()        │ ◀── Store to disk
└──────────────────────┘
    │ PASS
    ▼
┌──────────────────────┐
│ ConnectBlock()       │ ◀── Apply to chain state
└──────────────────────┘
    │ PASS
    ▼
┌──────────────────────┐
│ ActivateBestChain()  │ ◀── Switch to new tip if needed
└──────────────────────┘
```

### 3.2 CheckBlockHeader()

**Location**: `src/validation.cpp:3935`

```cpp
static bool CheckBlockHeader(const CBlockHeader& block, BlockValidationState& state, const Consensus::Params& consensusParams, bool fCheckPOW)
{
    // Check proof of work matches claimed amount
    if (fCheckPOW && !CheckProofOfWork(block.GetHash(), block.nBits, consensusParams))
        return state.Invalid(BlockValidationResult::BLOCK_INVALID_HEADER, "high-hash", "proof of work failed");
    
    return true;
}
```

**Checks**:
1. ✅ Proof-of-work hash meets difficulty target
2. ✅ Block header structure is valid

### 3.3 CheckBlock()

**Location**: `src/validation.cpp:4025`

```cpp
bool CheckBlock(const CBlock& block, BlockValidationState& state, const Consensus::Params& consensusParams, bool fCheckPOW, bool fCheckMerkleRoot)
{
    // Check header
    if (!CheckBlockHeader(block, state, consensusParams, fCheckPOW))
        return false;
    
    // Check merkle root
    if (fCheckMerkleRoot && !CheckMerkleRoot(block, state))
        return false;
    
    // Size limits
    if (block.vtx.empty() || block.vtx.size() * WITNESS_SCALE_FACTOR > MAX_BLOCK_WEIGHT)
        return state.Invalid(..., "bad-blk-length", "size limits failed");
    
    // First transaction must be coinbase, rest must not be
    if (block.vtx.empty() || !block.vtx[0]->IsCoinBase())
        return state.Invalid(..., "bad-cb-missing", "first tx is not coinbase");
    
    for (unsigned int i = 1; i < block.vtx.size(); i++)
        if (block.vtx[i]->IsCoinBase())
            return state.Invalid(..., "bad-cb-multiple", "more than one coinbase");
    
    // Check all transactions
    for (const auto& tx : block.vtx) {
        if (!CheckTransaction(*tx, tx_state))
            return state.Invalid(...);
    }
    
    // Check signature operations
    unsigned int nSigOps = 0;
    for (const auto& tx : block.vtx)
        nSigOps += GetLegacySigOpCount(*tx);
    
    if (nSigOps * WITNESS_SCALE_FACTOR > MAX_BLOCK_SIGOPS_COST)
        return state.Invalid(..., "bad-blk-sigops");
    
    return true;
}
```

**Checks**:
1. ✅ Valid PoW and header
2. ✅ Valid Merkle root
3. ✅ Block size within limits (4MB weight)
4. ✅ Exactly one coinbase transaction (first)
5. ✅ All transactions valid
6. ✅ Signature operations within limits

### 3.4 ConnectBlock()

**Location**: `src/validation.cpp:2375`

**Purpose**: Apply block to chain state (update UTXO set)

**Logic**:
```cpp
bool Chainstate::ConnectBlock(const CBlock& block, BlockValidationState& state, CBlockIndex* pindex, CCoinsViewCache& view, bool fJustCheck)
{
    // Verify transactions against UTXO set
    CAmount nFees = 0;
    
    for (unsigned int i = 0; i < block.vtx.size(); i++)
    {
        const CTransaction &tx = *(block.vtx[i]);
        
        // Skip coinbase
        if (!tx.IsCoinBase())
        {
            // Verify inputs exist and get values
            CAmount nTxValueIn = 0;
            for (const CTxIn &txin : tx.vin) {
                // Get coin from UTXO set
                const Coin& coin = view.AccessCoin(txin.prevout);
                
                // Verify coin exists
                if (coin.IsSpent())
                    return state.Invalid(..., "bad-txns-inputs-missingorspent");
                
                // Verify script
                if (!VerifyScript(...))
                    return state.Invalid(...);
                
                nTxValueIn += coin.out.nValue;
            }
            
            // Check input value >= output value
            if (nTxValueIn < tx.GetValueOut())
                return state.Invalid(..., "bad-txns-in-belowout");
            
            // Calculate fee
            CAmount nTxFee = nTxValueIn - tx.GetValueOut();
            nFees += nTxFee;
            
            // Update UTXO set (spend inputs, create outputs)
            UpdateCoins(tx, view, pindex->nHeight);
        }
    }
    
    // Verify block reward
    CAmount blockReward = nFees + GetBlockSubsidy(pindex->nHeight, params);
    if (block.vtx[0]->GetValueOut() > blockReward)
        return state.Invalid(..., "bad-cb-amount");
    
    return true;
}
```

**Checks**:
1. ✅ All inputs reference existing UTXOs
2. ✅ All scripts verify correctly
3. ✅ Input values ≥ output values (no inflation)
4. ✅ Coinbase reward correct (subsidy + fees)
5. ✅ UTXO set updated correctly

---

## 4. Transaction Validation Logic

### 4.1 CheckTransaction()

**Location**: `src/consensus/tx_check.cpp:11`

```cpp
bool CheckTransaction(const CTransaction& tx, TxValidationState& state)
{
    // Basic checks that don't depend on context
    
    // Must have inputs and outputs
    if (tx.vin.empty())
        return state.Invalid(..., "bad-txns-vin-empty");
    if (tx.vout.empty())
        return state.Invalid(..., "bad-txns-vout-empty");
    
    // Size limits
    if (::GetSerializeSize(TX_NO_WITNESS(tx)) * WITNESS_SCALE_FACTOR > MAX_BLOCK_WEIGHT)
        return state.Invalid(..., "bad-txns-oversize");
    
    // Check for negative or overflow output values
    CAmount nValueOut = 0;
    for (const auto& txout : tx.vout)
    {
        if (txout.nValue < 0)
            return state.Invalid(..., "bad-txns-vout-negative");
        if (txout.nValue > MAX_MONEY)
            return state.Invalid(..., "bad-txns-vout-toolarge");
        nValueOut += txout.nValue;
        if (!MoneyRange(nValueOut))
            return state.Invalid(..., "bad-txns-txouttotal-toolarge");
    }
    
    // Check for duplicate inputs
    std::set<COutPoint> vInOutPoints;
    for (const auto& txin : tx.vin) {
        if (!vInOutPoints.insert(txin.prevout).second)
            return state.Invalid(..., "bad-txns-inputs-duplicate");
    }
    
    // Coinbase checks
    if (tx.IsCoinBase())
    {
        if (tx.vin[0].scriptSig.size() < 2 || tx.vin[0].scriptSig.size() > 100)
            return state.Invalid(..., "bad-cb-length");
    }
    else
    {
        for (const auto& txin : tx.vin)
            if (txin.prevout.IsNull())
                return state.Invalid(..., "bad-txns-prevout-null");
    }
    
    return true;
}
```

**Validation Rules**:
1. ✅ At least one input
2. ✅ At least one output
3. ✅ Size ≤ MAX_BLOCK_WEIGHT
4. ✅ All output values ≥ 0
5. ✅ All output values ≤ MAX_MONEY
6. ✅ Total output ≤ MAX_MONEY (overflow check)
7. ✅ No duplicate inputs
8. ✅ Coinbase scriptSig size 2-100 bytes
9. ✅ Non-coinbase inputs reference valid outputs

### 4.2 Money Range Check

**Formula**:
```cpp
static const CAmount MAX_MONEY = 21000000 * COIN;  // 21 million coins

inline bool MoneyRange(const CAmount& nValue) {
    return (nValue >= 0 && nValue <= MAX_MONEY);
}
```

**Logic**: Prevents integer overflow and ensures values are within valid range.

---

## 5. UTXO Verification

### 5.1 UTXO Set Structure

**Location**: `src/coins.h`

```cpp
class Coin {
public:
    CTxOut out;         // The output (value + scriptPubKey)
    uint32_t nHeight;   // Height at which tx was included
    bool fCoinBase;     // Is this a coinbase output?
    
    bool IsSpent() const {
        return out.IsNull();
    }
};
```

### 5.2 Input Verification Logic

**Process**:
```
For each input in transaction:
  1. Look up UTXO by (txid, vout_index)
  2. Verify UTXO exists (not spent)
  3. Verify scriptSig + scriptPubKey
  4. Verify value
  5. Add to input_total
  
After all inputs:
  6. Verify input_total ≥ output_total
  7. Calculate fee = input_total - output_total
```

### 5.3 Double-Spend Prevention

**Mechanism**:
```cpp
// When checking transaction
for (const CTxIn &txin : tx.vin) {
    const Coin& coin = view.AccessCoin(txin.prevout);
    
    if (coin.IsSpent()) {
        // This output was already spent!
        return error("double-spend detected");
    }
    
    // Mark as spent
    view.SpendCoin(txin.prevout);
}
```

**Logic**:
- Each UTXO can only be spent once
- Attempting to spend same UTXO twice = double-spend
- UTXO database tracks spent/unspent state
- Once spent in valid block, UTXO is removed

---

## 6. Script Validation

### 6.1 Script Execution

**Location**: `src/script/interpreter.cpp`

**Process**:
```
1. Execute scriptSig (from input)
   → Pushes data onto stack
   
2. Execute scriptPubKey (from previous output)
   → Operates on stack
   
3. Check result
   → Top of stack must be TRUE (non-zero)
```

### 6.2 Script Verification Function

```cpp
bool VerifyScript(const CScript& scriptSig, const CScript& scriptPubKey, 
                  const CScriptWitness* witness, unsigned int flags,
                  const BaseSignatureChecker& checker, ScriptError* serror)
{
    // Execute scriptSig
    std::vector<std::vector<unsigned char>> stack;
    if (!EvalScript(stack, scriptSig, flags, checker, serror))
        return false;
    
    // Execute scriptPubKey
    if (!EvalScript(stack, scriptPubKey, flags, checker, serror))
        return false;
    
    // Stack must not be empty
    if (stack.empty())
        return false;
    
    // Top of stack must be true
    if (!CastToBool(stack.back()))
        return false;
    
    // P2SH verification
    if ((flags & SCRIPT_VERIFY_P2SH) && scriptPubKey.IsPayToScriptHash())
    {
        // Additional verification for P2SH
        // ...
    }
    
    // Witness verification (SegWit)
    if ((flags & SCRIPT_VERIFY_WITNESS) && witness)
    {
        // Verify witness program
        // ...
    }
    
    return true;
}
```

### 6.3 Signature Verification

**ECDSA Signature Check**:
```cpp
bool CheckSig(const std::vector<unsigned char>& vchSig,
              const std::vector<unsigned char>& vchPubKey,
              const CScript& scriptCode,
              const BaseSignatureChecker& checker)
{
    CPubKey pubkey(vchPubKey);
    
    // Verify public key is valid
    if (!pubkey.IsValid())
        return false;
    
    // Get transaction hash to sign
    uint256 sighash = SignatureHash(scriptCode, txTo, nIn, nHashType);
    
    // Verify signature
    if (!pubkey.Verify(sighash, vchSig))
        return false;
    
    return true;
}
```

**Signature Hash Calculation**:
```
sighash = SHA256(SHA256(
    nVersion ||
    hashPrevouts ||
    hashSequence ||
    outpoint ||
    scriptCode ||
    value ||
    nSequence ||
    hashOutputs ||
    nLockTime ||
    nHashType
))
```

---

## 7. Block Reward & Subsidy

### 7.1 Subsidy Calculation

**Location**: `src/validation.cpp:1919`

```cpp
CAmount GetBlockSubsidy(int nHeight, const Consensus::Params& consensusParams)
{
    int halvings = nHeight / consensusParams.nSubsidyHalvingInterval;
    
    // Force block reward to zero after 64 halvings
    if (halvings >= 64)
        return 0;
    
    CAmount nSubsidy = 50 * COIN;  // Initial reward: 50 coins
    
    // Halve the subsidy
    nSubsidy >>= halvings;  // Right shift = divide by 2^halvings
    
    return nSubsidy;
}
```

**Formula**:
```
subsidy = 50 * COIN / (2 ^ halvings)

where halvings = block_height / 210000

Examples:
Block 0-209,999:    50 / 2^0 = 50 coins
Block 210,000-...:  50 / 2^1 = 25 coins
Block 420,000-...:  50 / 2^2 = 12.5 coins
Block 630,000-...:  50 / 2^3 = 6.25 coins
```

### 7.2 Block Reward Verification

**Location**: `src/validation.cpp:2689`

```cpp
// Calculate total allowed reward
CAmount blockReward = nFees + GetBlockSubsidy(pindex->nHeight, params);

// Verify coinbase doesn't create too much
if (block.vtx[0]->GetValueOut() > blockReward)
    return state.Invalid(..., "bad-cb-amount", 
                         strprintf("coinbase pays too much"));
```

**Logic**:
```
Total Block Reward = Block Subsidy + Transaction Fees

where:
- Block Subsidy = calculated from height (halving schedule)
- Transaction Fees = sum of all (inputs - outputs) in block

Coinbase output MUST be ≤ Total Block Reward
```

---

## 8. Time-Lock Mechanisms

### 8.1 Transaction Lock Time

**Location**: `src/consensus/tx_verify.cpp:17`

```cpp
bool IsFinalTx(const CTransaction &tx, int nBlockHeight, int64_t nBlockTime)
{
    // If nLockTime is 0, transaction is final
    if (tx.nLockTime == 0)
        return true;
    
    // Check if locktime has been reached
    int64_t lockTime = (int64_t)tx.nLockTime;
    int64_t threshold = (lockTime < LOCKTIME_THRESHOLD) ? nBlockHeight : nBlockTime;
    
    if (lockTime < threshold)
        return true;  // Lock time has passed
    
    // Even if locktime not reached, tx is final if all inputs have max sequence
    for (const auto& txin : tx.vin) {
        if (txin.nSequence != CTxIn::SEQUENCE_FINAL)
            return false;
    }
    
    return true;
}
```

**Logic**:
```
LOCKTIME_THRESHOLD = 500,000,000

If nLockTime < LOCKTIME_THRESHOLD:
    → Interpreted as block height
    → Transaction valid when chain height ≥ nLockTime
    
If nLockTime ≥ LOCKTIME_THRESHOLD:
    → Interpreted as Unix timestamp
    → Transaction valid when block time ≥ nLockTime

Special case:
    If all inputs have nSequence = 0xFFFFFFFF (SEQUENCE_FINAL)
    → nLockTime is ignored, transaction is immediately final
```

### 8.2 Sequence Locks (BIP 68)

**Location**: `src/consensus/tx_verify.cpp:39`

```cpp
std::pair<int, int64_t> CalculateSequenceLocks(const CTransaction &tx, int flags, 
                                                std::vector<int>& prevHeights, 
                                                const CBlockIndex& block)
{
    int nMinHeight = -1;
    int64_t nMinTime = -1;
    
    bool fEnforceBIP68 = tx.version >= 2 && flags & LOCKTIME_VERIFY_SEQUENCE;
    
    if (!fEnforceBIP68)
        return std::make_pair(nMinHeight, nMinTime);
    
    for (size_t txinIndex = 0; txinIndex < tx.vin.size(); txinIndex++) {
        const CTxIn& txin = tx.vin[txinIndex];
        
        // Check if sequence lock disabled for this input
        if (txin.nSequence & CTxIn::SEQUENCE_LOCKTIME_DISABLE_FLAG)
            continue;
        
        int nCoinHeight = prevHeights[txinIndex];
        
        if (txin.nSequence & CTxIn::SEQUENCE_LOCKTIME_TYPE_FLAG) {
            // Time-based lock
            int64_t nCoinTime = block.GetAncestor(max(nCoinHeight - 1, 0))->GetMedianTimePast();
            nMinTime = max(nMinTime, nCoinTime + 
                          (int64_t)((txin.nSequence & CTxIn::SEQUENCE_LOCKTIME_MASK) 
                          << CTxIn::SEQUENCE_LOCKTIME_GRANULARITY) - 1);
        } else {
            // Height-based lock
            nMinHeight = max(nMinHeight, nCoinHeight + 
                            (int)(txin.nSequence & CTxIn::SEQUENCE_LOCKTIME_MASK) - 1);
        }
    }
    
    return std::make_pair(nMinHeight, nMinTime);
}
```

**Logic**:
```
Sequence Lock Format (32-bit):
  Bit 31: Disable flag (1 = disabled)
  Bit 22: Type flag (0 = height, 1 = time)
  Bits 0-15: Value

Height-based:
  Value = number of blocks to wait
  Granularity: 1 block
  
Time-based:
  Value = number of 512-second intervals to wait
  Granularity: 512 seconds (~8.5 minutes)

Example:
  nSequence = 0x00000010 (16 in decimal)
  → Wait 16 blocks after input was created
```

---

## 9. Signature Verification

### 9.1 ECDSA Signature

**Curve**: secp256k1
**Location**: `src/secp256k1/`

**Verification Process**:
```
1. Parse signature (r, s values)
2. Parse public key point (x, y)
3. Calculate message hash (sighash)
4. Verify: r = x-coordinate of (s^-1 × hash × G + s^-1 × r × PubKey)
```

**Implementation**:
```cpp
bool CPubKey::Verify(const uint256 &hash, const std::vector<unsigned char>& vchSig) const
{
    if (!IsValid())
        return false;
    
    secp256k1_pubkey pubkey;
    secp256k1_ecdsa_signature sig;
    
    // Parse public key
    if (!secp256k1_ec_pubkey_parse(secp256k1_context_verify, &pubkey, data(), size()))
        return false;
    
    // Parse signature
    if (!secp256k1_ecdsa_signature_parse_der(secp256k1_context_verify, &sig, 
                                              vchSig.data(), vchSig.size()))
        return false;
    
    // Verify signature
    return secp256k1_ecdsa_verify(secp256k1_context_verify, &sig, hash.begin(), &pubkey);
}
```

### 9.2 Schnorr Signature (Taproot)

**BIP 340 Schnorr Signatures**

**Format**:
- 64 bytes total
- First 32 bytes: r value
- Last 32 bytes: s value

**Verification**:
```
Verify: R = s×G - e×P
where:
  R = commitment point
  s = signature scalar
  e = hash of (R || P || message)
  P = public key
  G = generator point
```

**Advantages**:
1. Smaller signatures (64 bytes vs ~71 bytes for ECDSA)
2. Provably secure
3. Enables key aggregation (MuSig)
4. Batch verification possible

---

## 10. Consensus State Transitions

### 10.1 State Machine

```
                    ┌─────────────┐
                    │   GENESIS   │
                    └──────┬──────┘
                           │
                    Block 1 received
                           │
                           ▼
                    ┌─────────────┐
                    │  Height: 1  │
                    │  UTXO Set   │
                    └──────┬──────┘
                           │
                    Block 2 received
                           │
                           ▼
                    ┌─────────────┐
                    │  Height: 2  │
                    │  UTXO Set   │
                    └──────┬──────┘
                           │
                           ⋮
```

### 10.2 State Transition Logic

```cpp
bool ConnectBlock(Block b) {
    // Current state
    UTXOSet utxo_before;
    
    // Apply transactions
    for (tx in b.transactions) {
        // Spend inputs
        for (input in tx.inputs) {
            utxo_before.remove(input.prevout);
        }
        
        // Create outputs
        for (i, output in enumerate(tx.outputs)) {
            utxo_before.add(COutPoint(tx.hash, i), output);
        }
    }
    
    // New state
    UTXOSet utxo_after = utxo_before;
    
    return true;
}
```

### 10.3 Reorg Handling

**Scenario**: New chain has more work

```
Original Chain:
  A ← B ← C ← D ← E (our tip)

New Chain:
  A ← B ← C ← F ← G ← H (more work!)

Reorganization Process:
  1. Find fork point: Block C
  2. Disconnect: E, D
  3. Connect: F, G, H
  4. New tip: H
```

**Logic**:
```cpp
bool ActivateBestChain() {
    CBlockIndex* pindexOldTip = chainActive.Tip();
    CBlockIndex* pindexNewTip = FindMostWorkChain();
    
    if (pindexNewTip == pindexOldTip)
        return true;  // No change
    
    // Find fork point
    CBlockIndex* pindexFork = chainActive.FindFork(pindexNewTip);
    
    // Disconnect old blocks
    while (chainActive.Tip() != pindexFork) {
        CBlock block;
        DisconnectBlock(block, chainActive.Tip());
        chainActive.SetTip(chainActive.Tip()->pprev);
    }
    
    // Connect new blocks
    std::vector<CBlockIndex*> vpindexToConnect;
    while (pindexNewTip != pindexFork) {
        vpindexToConnect.push_back(pindexNewTip);
        pindexNewTip = pindexNewTip->pprev;
    }
    
    for (int i = vpindexToConnect.size() - 1; i >= 0; i--) {
        CBlock block;
        ConnectBlock(block, vpindexToConnect[i]);
        chainActive.SetTip(vpindexToConnect[i]);
    }
    
    return true;
}
```

---

## 11. Fork Detection & Resolution

### 11.1 Chain Work Calculation

**Formula**:
```
chain_work = Σ(2^256 / (target + 1))

For each block:
  block_work = 2^256 / (target + 1)
  chain_work += block_work
```

**Implementation**:
```cpp
arith_uint256 GetBlockProof(const CBlockIndex& block)
{
    arith_uint256 bnTarget;
    bnTarget.SetCompact(block.nBits);
    
    if (bnTarget <= 0 || bnTarget > bnPowLimit)
        return 0;
    
    // We need to compute 2^256 / (bnTarget+1), but we can't represent 2^256
    // Instead: (~bnTarget / (bnTarget+1)) + 1
    return (~bnTarget / (bnTarget + 1)) + 1;
}
```

### 11.2 Best Chain Selection

**Rule**: Chain with most accumulated work wins

```cpp
CBlockIndex* FindMostWorkChain()
{
    CBlockIndex* pindexBest = nullptr;
    arith_uint256 nBestWork = 0;
    
    // Find tip with most work
    for (auto& entry : mapBlockIndex) {
        CBlockIndex* pindex = entry.second;
        
        if (pindex->nChainWork > nBestWork) {
            nBestWork = pindex->nChainWork;
            pindexBest = pindex;
        }
    }
    
    return pindexBest;
}
```

---

## 12. Consensus Parameters

### 12.1 Critical Parameters

**Location**: `src/kernel/chainparams.cpp`

```cpp
// Mainnet Consensus Parameters
consensus.nSubsidyHalvingInterval = 210000;           // ~4 years
consensus.nPowTargetTimespan = 14 * 24 * 60 * 60;   // 2 weeks
consensus.nPowTargetSpacing = 10 * 60;               // 10 minutes
consensus.fPowAllowMinDifficultyBlocks = false;      // No easy blocks
consensus.fPowNoRetargeting = false;                 // Enable difficulty adjustment
consensus.powLimit = uint256{"00000000ffffffffffffffffffffffffffffffffffffffffffffffffffffffff"};

// BIP activation heights
consensus.BIP34Height = 227931;
consensus.BIP65Height = 388381;
consensus.BIP66Height = 363725;
consensus.CSVHeight = 419328;
consensus.SegwitHeight = 481824;
```

### 12.2 Derived Parameters

```cpp
// Difficulty adjustment interval
inline int DifficultyAdjustmentInterval() const { 
    return nPowTargetTimespan / nPowTargetSpacing; 
}
// = 1209600 / 600 = 2016 blocks

// Coinbase maturity
static const int COINBASE_MATURITY = 100;  // blocks

// Maximum block weight
static const unsigned int MAX_BLOCK_WEIGHT = 4000000;

// Maximum block sigops
static const int64_t MAX_BLOCK_SIGOPS_COST = 80000;
```

---

## 13. Consensus Logic Summary

### 13.1 Complete Validation Checklist

**For a block to be valid**, ALL of these must pass:

**Block Header**:
- [ ] Valid proof-of-work hash
- [ ] Timestamp not too far in future (< 2 hours)
- [ ] Version number acceptable

**Block Structure**:
- [ ] Valid merkle root
- [ ] Size within limits (≤ 4MB weight)
- [ ] Exactly one coinbase transaction (first)
- [ ] All transactions valid

**Transactions**:
- [ ] All inputs reference existing UTXOs
- [ ] All scripts verify successfully
- [ ] No double-spends
- [ ] Input values ≥ output values
- [ ] Proper signatures

**Coinbase**:
- [ ] Reward ≤ subsidy + fees
- [ ] ScriptSig length 2-100 bytes
- [ ] Maturity respected (100 blocks)

**Consensus Rules**:
- [ ] Difficulty correct
- [ ] Locktime rules followed
- [ ] Sequence locks respected
- [ ] BIP activations at correct heights

### 13.2 Consensus Failure = Chain Split

**Critical**: If nodes disagree on consensus rules, the network splits:

```
             Block N (all agree)
                   │
        ┌──────────┴──────────┐
        │                     │
   Old Rules              New Rules
  (rejects block)        (accepts block)
        │                     │
    Chain A                Chain B
   (minority)             (majority)
```

**Example Scenarios**:
1. Different block size limits → split
2. Different subsidy calculations → split
3. Different script rules → split

**Non-Consensus Differences** (no split):
1. Different mempool policies
2. Different fee estimations
3. Different peer selection

---

## 14. Key Consensus Insights

### 14.1 Security Properties

1. **No Inflation**: UTXO checks ensure no coins created from thin air
2. **No Double-Spend**: Each UTXO can only be spent once
3. **Proof-of-Work**: Expensive to create alternative histories
4. **Finality**: Deep blocks extremely hard to reverse (6 confirmations standard)

### 14.2 Attack Resistance

**51% Attack**:
- Attacker with >50% hash power can:
  - Reverse recent transactions (double-spend)
  - Prevent transactions from confirming
- Attacker CANNOT:
  - Create coins from nothing
  - Steal coins without private keys
  - Change consensus rules

**Protection**: Distributed hash power, checkpoint blocks

### 14.3 Consensus Evolution

**Soft Fork**: Tightens rules (backward compatible)
- Old nodes: accept new blocks
- New nodes: enforce stricter rules
- Example: SegWit, Taproot

**Hard Fork**: Loosens rules (NOT backward compatible)
- Requires all nodes upgrade
- Creates chain split if some don't upgrade
- Example: changing block size limit

---

## Conclusion

Sqoin Core's consensus logic is a carefully designed system that ensures all nodes agree on the blockchain state without central authority. Key mechanisms:

1. **Proof-of-Work**: Makes it expensive to create blocks
2. **Difficulty Adjustment**: Maintains consistent block times
3. **UTXO Validation**: Prevents double-spending and inflation
4. **Script System**: Enables flexible spending conditions
5. **Time Locks**: Allows delayed transactions
6. **Chain Selection**: Most work wins during forks

All consensus rules are deterministic and verifiable, ensuring network-wide agreement on transaction validity and blockchain state.

---

**Reference Files**:
- `src/pow.cpp` - Proof-of-work logic
- `src/validation.cpp` - Block/tx validation
- `src/consensus/tx_check.cpp` - Transaction checks
- `src/consensus/tx_verify.cpp` - UTXO verification
- `src/script/interpreter.cpp` - Script execution
- `src/kernel/chainparams.cpp` - Consensus parameters

**Next Steps**: See `docs/changes.md` for how to modify consensus parameters.
