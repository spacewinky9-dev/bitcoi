# Sqoin Core - Current Implementation Strategy

## Overview

This document provides a comprehensive analysis of Sqoin Core's current implementation, inherited from Bitcoin Core. It details the existing logic, formulas, algorithms, and strategies used throughout the codebase.

---

## 1. Consensus Strategy

### 1.1 Block Generation & Mining

**Location**: `src/kernel/chainparams.cpp`, `src/consensus/params.h`

**Current Parameters**:
```cpp
// Block Time Target
consensus.nPowTargetSpacing = 10 * 60;  // 10 minutes per block

// Difficulty Adjustment
consensus.nPowTargetTimespan = 14 * 24 * 60 * 60;  // 2 weeks
// Adjustment Period = 2016 blocks (2 weeks / 10 minutes)

// Proof-of-Work Limit
consensus.powLimit = uint256{"00000000ffffffffffffffffffffffffffffffffffffffffffffffffffffffff"};
```

**Strategy**:
- **SHA-256d** double hashing algorithm for proof-of-work
- Difficulty adjusts every 2016 blocks to maintain 10-minute average
- Target: `MAX_TARGET / current_difficulty`

**Formula for Difficulty Adjustment**:
```
New_Difficulty = Old_Difficulty * (Actual_Time / Expected_Time)
Where:
  Expected_Time = 2016 * 600 seconds = 2 weeks
  Actual_Time = time to mine last 2016 blocks
  
Constraints:
  - Maximum adjustment: 4x increase or 0.25x decrease per period
  - Prevents sudden difficulty changes
```

### 1.2 Block Subsidy & Halving

**Location**: `src/validation.cpp` - `GetBlockSubsidy()`

**Current Strategy**:
```cpp
consensus.nSubsidyHalvingInterval = 210000;  // ~4 years
Initial reward = 50 * COIN

Formula:
CAmount nSubsidy = 50 * COIN;
nSubsidy >>= (nHeight / nSubsidyHalvingInterval);
// Right shift = divide by 2 for each halving period
```

**Halving Schedule**:
- Block 0-209,999: 50 coins
- Block 210,000-419,999: 25 coins
- Block 420,000-629,999: 12.5 coins
- Continues until subsidy reaches 0 (~year 2140)
- Total supply: ~21 million coins

### 1.3 Genesis Block

**Location**: `src/kernel/chainparams.cpp`

**Current Configuration**:
```cpp
genesis = CreateGenesisBlock(
    1231006505,      // Unix timestamp: Jan 3, 2009
    2083236893,      // Nonce (proof-of-work solution)
    0x1d00ffff,      // Initial difficulty bits
    1,               // Version
    50 * COIN        // Initial reward
);

// Embedded message in coinbase
const char* pszTimestamp = "The Times 03/Jan/2009 Chancellor on brink of second bailout for banks";

// Genesis hash
consensus.hashGenesisBlock = uint256{"000000000019d6689c085ae165831e934ff763ae46a2a6c172b3f1b60a8ce26f"};
```

**Strategy**: Genesis block is hardcoded and immutable. All nodes must have the same genesis block hash to be on the same network.

---

## 2. Transaction Validation Strategy

### 2.1 Transaction Structure

**Location**: `src/primitives/transaction.h`

**Key Components**:
```cpp
class CTransaction {
    int32_t nVersion;             // Transaction version
    std::vector<CTxIn> vin;       // Inputs
    std::vector<CTxOut> vout;     // Outputs
    uint32_t nLockTime;           // Lock time
    // Witness data (SegWit)
};

class CTxIn {
    COutPoint prevout;            // Previous output reference
    CScript scriptSig;            // Signature script
    uint32_t nSequence;           // Sequence number
    CScriptWitness scriptWitness; // Witness data
};

class CTxOut {
    CAmount nValue;               // Amount in satoshis
    CScript scriptPubKey;         // Locking script
};
```

### 2.2 Validation Rules

**Location**: `src/consensus/tx_verify.cpp`, `src/validation.cpp`

**Current Strategy**:

1. **Basic Checks** (`CheckTransaction`):
   - No inputs or outputs empty
   - Output values >= 0
   - Total output value <= MAX_MONEY
   - No duplicate inputs
   - Coinbase checks (if applicable)

2. **Script Verification** (`VerifyScript`):
   - Execute scriptSig + scriptPubKey
   - Verify signatures using secp256k1
   - Check locktime/sequence constraints

3. **UTXO Checks**:
   - All inputs reference existing unspent outputs
   - Input values >= output values + fees
   - No double-spending

**Formula for Transaction Fee**:
```
Fee = Sum(Input_Values) - Sum(Output_Values)
Must be: Fee >= 0
```

### 2.3 Replace-by-Fee (RBF)

**Location**: `src/policy/rbf.cpp`

**Current Strategy**:
- Transaction signals RBF if any input has `nSequence < 0xfffffffe`
- Replacement must pay higher fee
- Must pay for bandwidth: new fee >= old fee + min relay fee * size
- No new unconfirmed inputs unless they belonged to replaced tx

---

## 3. Mempool Strategy

### 3.1 Transaction Pool Management

**Location**: `src/txmempool.cpp`

**Current Strategy**:

**Data Structures**:
```cpp
class CTxMemPool {
    std::map<uint256, CTxMemPoolEntry> mapTx;  // All transactions
    // Indexed by: fee, time, ancestor score
    
    // Ancestor/descendant tracking
    // Limits: max 25 ancestors, max 25 descendants
    // Prevents long dependency chains
};
```

**Admission Rules**:
1. Transaction must be valid
2. Must pay minimum relay fee
3. Ancestor/descendant limits not exceeded
4. Not already in mempool or blockchain
5. Not conflicting with mempool (unless RBF)

**Eviction Strategy**:
- When mempool full (default 300MB)
- Evict transactions with lowest fee rate
- Keep high-fee transactions

### 3.2 Fee Estimation

**Location**: `src/policy/fees/`

**Current Strategy**:
- Historical data: track confirmation times at different fee rates
- Estimate probability of confirmation within N blocks
- Use exponential moving average for recent data
- Adjust for mempool size

**Algorithm**:
```
For each fee bucket:
  Track: confirmations at various block depths
  Calculate: success rate for N-block confirmation
  Return: minimum fee rate with >85% success probability
```

---

## 4. Script Execution Strategy

### 4.1 Script Types

**Location**: `src/script/interpreter.cpp`

**Supported Types**:

1. **P2PKH** (Pay to Public Key Hash):
   ```
   scriptPubKey: OP_DUP OP_HASH160 <pubKeyHash> OP_EQUALVERIFY OP_CHECKSIG
   scriptSig: <signature> <pubKey>
   ```

2. **P2SH** (Pay to Script Hash):
   ```
   scriptPubKey: OP_HASH160 <scriptHash> OP_EQUAL
   scriptSig: <data> <redeemScript>
   ```

3. **P2WPKH** (Pay to Witness Public Key Hash - SegWit):
   ```
   scriptPubKey: OP_0 <20-byte-pubkey-hash>
   witness: <signature> <pubkey>
   ```

4. **P2TR** (Pay to Taproot):
   ```
   scriptPubKey: OP_1 <32-byte-x-only-pubkey>
   witness: <signature> or <control_block> <script>
   ```

### 4.2 Script Execution Engine

**Current Strategy**:
- Stack-based execution
- No loops (not Turing-complete)
- Limited opcodes for security
- Signature verification using secp256k1

**Execution Flow**:
```
1. Execute scriptSig → stack
2. Execute scriptPubKey with stack
3. Verify top of stack is TRUE
4. For SegWit: verify witness
5. For Taproot: verify Schnorr signature
```

---

## 5. UTXO Management Strategy

### 5.1 UTXO Set

**Location**: `src/coins.cpp`, `src/coins.h`

**Current Implementation**:
```cpp
class Coin {
    CTxOut out;         // The output itself
    uint32_t nHeight;   // Height of block containing tx
    bool fCoinBase;     // Is it a coinbase output?
};

// UTXO database
class CCoinsViewDB : public CCoinsView {
    // LevelDB backend
    // Key: txid + output_index
    // Value: Coin data
};
```

**Strategy**:
- In-memory cache for recent UTXOs
- Batch writes to disk for performance
- Pruning old spent outputs
- Efficient lookup by outpoint (txid + index)

### 5.2 Caching Strategy

**Layers**:
1. **CCoinsViewCache** - In-memory cache
2. **CCoinsViewDB** - LevelDB persistent storage
3. **CCoinsViewErrorCatcher** - Error handling wrapper

**Cache Eviction**:
- Write through on block connection
- Flush every ~50-70 minutes (randomized)
- Flush on shutdown

---

## 6. P2P Network Strategy

### 6.1 Network Protocol

**Location**: `src/net.cpp`, `src/net_processing.cpp`

**Message Start Bytes**:
```cpp
// Network identifier (magic bytes)
pchMessageStart[0] = 0xf9;  // Mainnet
pchMessageStart[1] = 0xbe;
pchMessageStart[2] = 0xb4;
pchMessageStart[3] = 0xd9;

nDefaultPort = 8333;  // Default P2P port
```

### 6.2 Peer Discovery

**Current Strategy**:

1. **DNS Seeds** (hardcoded):
   ```cpp
   vSeeds.emplace_back("seed.bitcoin.sipa.be.");
   vSeeds.emplace_back("dnsseed.bluematt.me.");
   // etc.
   ```

2. **Address Manager** (`CAddrMan`):
   - Stores known peer addresses
   - Anti-Sybil attack protection
   - Bucketing by network group
   - Tries to maintain 8 outbound connections

3. **Connection Strategy**:
   - Max 125 connections (8 outbound, rest inbound)
   - Prefer different network groups
   - Prefer full nodes (NODE_NETWORK)

### 6.3 Block Propagation

**Current Strategies**:

1. **Headers-First Sync**:
   - Download headers before blocks
   - Parallel block download
   - Faster initial sync

2. **Compact Blocks** (BIP 152):
   - Send short transaction IDs instead of full txs
   - Reduce bandwidth by ~95%
   - Peers reconstruct from mempool

3. **Erlay** (Future):
   - Set reconciliation for transaction relay
   - Uses minisketch library
   - Further bandwidth reduction

---

## 7. Wallet Strategy

### 7.1 Key Management

**Location**: `src/wallet/scriptpubkeyman.cpp`

**Current Strategies**:

1. **HD Wallets** (BIP 32):
   ```
   Master seed → Master key → Derived keys
   Path: m/purpose'/coin_type'/account'/change/index
   
   Example: m/44'/0'/0'/0/0  (First receiving address)
   ```

2. **Descriptors** (Modern):
   - Describes how to derive addresses
   - Examples:
     - `pkh([fingerprint/44'/0'/0']xpub.../0/*)`
     - `wpkh([fingerprint/84'/0'/0']xpub.../0/*)`
     - `tr([fingerprint/86'/0'/0']xpub.../0/*)`

### 7.2 Coin Selection

**Location**: `src/wallet/coinselection.cpp`

**Algorithms Used**:

1. **Branch and Bound**:
   - Tries to find exact match
   - Avoids change output
   - Timeout: 0.5 seconds
   - Most efficient when successful

2. **Knapsack Solver**:
   - Approximate solution
   - Random selection with improvements
   - Fallback when B&B fails

3. **Single Random Draw**:
   - Simple random selection
   - Fastest but least optimal

**Strategy Selection**:
```
1. Try Branch and Bound first
2. If fails or times out → Knapsack
3. If still fails → Single Random Draw
```

### 7.3 Transaction Building

**Location**: `src/wallet/spend.cpp`

**Current Process**:
```
1. Select coins (coin selection algorithm)
2. Calculate fee (based on size estimate)
3. Create outputs (recipients + change)
4. Build transaction
5. Sign inputs
6. Verify transaction
7. Broadcast to network
```

**Fee Calculation**:
```cpp
// Fee estimation based on:
- Transaction size (bytes or vbytes for SegWit)
- Target confirmation blocks
- Current mempool state
- Fee rate (sat/vB)

Total_Fee = Fee_Rate * Transaction_Size
```

---

## 8. Database Strategy

### 8.1 LevelDB Usage

**Location**: `src/leveldb/`, `src/txdb.cpp`

**Current Databases**:

1. **Block Index** (`blocks/index/`):
   - Key: block hash
   - Value: CBlockIndex metadata
   - Stores all known blocks

2. **Chainstate** (`chainstate/`):
   - Key: 'c' + txid + output_index
   - Value: Coin data
   - Current UTXO set

3. **Block Files** (`blocks/blk*.dat`):
   - Raw block data
   - Sequential append-only
   - Separate undo files for reorgs

**Strategy**:
- Batch writes for performance
- Write-ahead logging
- Periodic compaction
- Corruption detection and recovery

### 8.2 Wallet Database

**Location**: `src/wallet/walletdb.cpp`

**Two Backends**:

1. **Berkeley DB** (Legacy):
   - Key-value store
   - Transaction support
   - Being phased out

2. **SQLite** (Descriptor Wallets):
   - Relational database
   - Better corruption resistance
   - Modern standard

---

## 9. Validation Strategy

### 9.1 Block Validation

**Location**: `src/validation.cpp`

**Process**:
```
CheckBlock() → Basic block checks
  ↓
AcceptBlock() → Context-independent checks
  ↓
ConnectBlock() → Apply to chain state
  ↓
Update UTXO set
  ↓
ActivateBestChain() → Switch if needed
```

**Checks Performed**:
1. Header validation (PoW, timestamp, version)
2. Merkle root verification
3. Transaction validation (all txs in block)
4. Block weight/size limits
5. Coinbase reward correct
6. No duplicate transactions
7. Block builds on valid parent

### 9.2 Reorganization Handling

**Strategy**:
```
If new chain has more work:
  1. Find fork point
  2. Disconnect blocks from old tip → fork point
  3. Connect blocks from fork point → new tip
  4. Update UTXO set (undo then apply)
  5. Notify wallet and other subsystems
```

**Maximum Reorg Depth**:
- No hardcoded limit
- Practical limit: ~100 blocks (depends on checkpoints)

---

## 10. Security Strategy

### 10.1 Cryptographic Primitives

**Location**: `src/crypto/`, `src/secp256k1/`

**Current Algorithms**:

1. **SHA-256**:
   - Block hashing
   - Transaction hashing
   - Merkle trees

2. **RIPEMD-160**:
   - Address generation (HASH160 = RIPEMD160(SHA256(x)))

3. **secp256k1**:
   - ECDSA signatures (legacy)
   - Schnorr signatures (Taproot)
   - Key derivation

**Signature Verification**:
```cpp
// ECDSA (legacy)
bool VerifySignature(pubkey, message, signature)

// Schnorr (Taproot)
bool VerifySchnorrSignature(pubkey, message, signature)
  - 64-byte signatures
  - Batch verification possible
  - Better privacy (key aggregation)
```

### 10.2 DoS Prevention

**Strategies**:

1. **Resource Limits**:
   - Max block size: 4MB (weight)
   - Max script size: 10,000 bytes
   - Max signature operations per block

2. **Rate Limiting**:
   - Limit messages per peer
   - Ban misbehaving peers
   - Connection limits

3. **Validation Costs**:
   - Script cache (avoid re-validation)
   - Signature cache
   - Checkqueue for parallel validation

---

## 11. Consensus Upgrades Strategy

### 11.1 Soft Fork Mechanism

**Location**: `src/versionbits.cpp`

**BIP 9 - Version Bits**:
```
1. DEFINED: Soft fork code deployed
2. STARTED: Miners begin signaling
3. LOCKED_IN: >90% miners signal (2016 blocks)
4. ACTIVE: Fork activated after lock-in period
5. FAILED: Timeout reached without activation
```

**Historical Upgrades**:
- BIP 34: Block height in coinbase (Height: 227931)
- BIP 65: CHECKLOCKTIMEVERIFY (Height: 388381)
- BIP 66: Strict DER signatures (Height: 363725)
- BIP 68/112/113: CSV (Height: 419328)
- BIP 141/143/144: SegWit (Height: 481824)
- BIP 340/341/342: Taproot (Height: 709632)

### 11.2 Hard Fork Prevention

**Strategy**:
- All upgrades are soft forks (backward compatible)
- Old nodes still validate most transactions
- Gradual network upgrade without split

---

## 12. Performance Optimizations

### 12.1 Caching Strategies

**Current Caches**:

1. **Script Cache** (`src/script/sigcache.cpp`):
   - Caches script execution results
   - 32MB default size
   - Prevents re-execution

2. **Signature Cache**:
   - Caches signature verifications
   - Uses cuckoo cache
   - Significantly speeds up validation

3. **UTXO Cache**:
   - In-memory UTXO set
   - 450MB default size
   - Batch flush to disk

### 12.2 Parallel Processing

**Strategies**:

1. **Script Verification**:
   - Up to 15 threads (MAX_SCRIPTCHECK_THREADS)
   - Parallel validation of block transactions
   - CheckQueue implementation

2. **Block Download**:
   - Parallel downloads from multiple peers
   - Headers-first allows parallel processing

---

## 13. Network Constants

### 13.1 Key Parameters

```cpp
// Blockchain Parameters
BLOCK_TIME_TARGET = 600 seconds (10 minutes)
DIFFICULTY_ADJUSTMENT_PERIOD = 2016 blocks (~2 weeks)
MAX_BLOCK_WEIGHT = 4,000,000 weight units
MAX_BLOCK_SIZE = 1,000,000 bytes (legacy)

// Economic Parameters
INITIAL_SUBSIDY = 50 * COIN
HALVING_INTERVAL = 210,000 blocks (~4 years)
MAX_MONEY = 21,000,000 * COIN (satoshis)
COIN = 100,000,000 satoshis

// Network Parameters
DEFAULT_PORT = 8333 (mainnet)
MAX_CONNECTIONS = 125
MAX_OUTBOUND = 8
MAX_FEELER = 1

// Validation Parameters
MAX_FUTURE_BLOCK_TIME = 2 hours
COINBASE_MATURITY = 100 blocks
MAX_SCRIPTCHECK_THREADS = 15
```

---

## 14. External References Found

### 14.1 Documentation Links Only

**Found in source code** (comments only, not functional dependencies):
- GitHub Bitcoin repository (documentation references)
- BIP repository (specification references)
- libevent repository (implementation notes)

**DNS Seeds** (functional dependency):
- seed.bitcoin.sipa.be
- dnsseed.bluematt.me
- dnsseed.bitcoin.dashjr-list-of-p2p-nodes.us
- seed.bitcoinstats.com
- seed.bitcoin.jonasschnelli.ch
- seed.btc.petertodd.net
- seed.bitcoin.sprovoost.nl
- dnsseed.emzy.de
- seed.bitcoin.wiz.biz

**Note**: DNS seeds are external services but can be configured. The codebase is otherwise self-contained.

---

## 15. Self-Contained Verification

### 15.1 Embedded Libraries

All cryptographic and database functionality is embedded:
- ✅ **LevelDB** - Included in `src/leveldb/`
- ✅ **secp256k1** - Included in `src/secp256k1/`
- ✅ **univalue** - Included in `src/univalue/`
- ✅ **minisketch** - Included in `src/minisketch/`
- ✅ **CRC32C** - Included in `src/crc32c/`

### 15.2 External Dependencies (Build-Time)

**Required** (not included):
- Boost (utilities)
- libevent (HTTP server)
- SQLite (wallet DB)
- Qt (GUI, optional)

**Conclusion**: Core functionality is self-contained. External deps are for infrastructure (DB, HTTP, GUI), not consensus logic.

---

## 16. Current Limitations & Design Decisions

### 16.1 Known Constraints

1. **Block Size**: Limited to 4MB weight (prevents spam)
2. **Block Time**: 10 minutes average (security vs speed tradeoff)
3. **Script Limitations**: No loops, limited opcodes (security)
4. **Memory Pool**: 300MB default (prevents DoS)
5. **Connection Limits**: 125 total (resource management)

### 16.2 Design Philosophy

- **Security First**: Conservative approach to changes
- **Decentralization**: Low resource requirements for full nodes
- **Backward Compatibility**: Soft forks only
- **No Central Authority**: No hardcoded trusted parties
- **Open Source**: MIT license, fully auditable

---

## Summary

This document has analyzed the complete current strategy of Sqoin Core. The implementation is:

1. **Self-contained** - No external code dependencies for core functionality
2. **Well-structured** - Clear separation of concerns
3. **Battle-tested** - Inherited from Bitcoin's 15+ years of development
4. **Secure** - Conservative approach with multiple validation layers
5. **Performant** - Multiple caching and optimization strategies

The only external connections are:
- DNS seeds (configurable, not required)
- Documentation links (comments only)

All consensus logic, cryptography, and core functionality is included in the repository.

---

**Next Steps**: See `changes.md` for detailed modification guide for enhancement and rebranding.
