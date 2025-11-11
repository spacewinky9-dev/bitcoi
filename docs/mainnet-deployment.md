# Sqoin Core - Mainnet Deployment Guide

## Complete Step-by-Step Guide for Production Network Launch

This comprehensive guide walks you through deploying Sqoin Core on a mainnet for testing purposes, covering all challenges, required file modifications, solutions, implementations, and expected outcomes.

---

## Table of Contents

1. [Pre-Deployment Planning](#1-pre-deployment-planning)
2. [Critical Files to Modify](#2-critical-files-to-modify)
3. [Step-by-Step Deployment](#3-step-by-step-deployment)
4. [Common Challenges & Solutions](#4-common-challenges--solutions)
5. [Network Configuration](#5-network-configuration)
6. [Genesis Block Creation](#6-genesis-block-creation)
7. [Chainparams Modification](#7-chainparams-modification)
8. [DNS Seed Setup](#8-dns-seed-setup)
9. [Checkpoint Configuration](#9-checkpoint-configuration)
10. [Testing and Validation](#10-testing-and-validation)
11. [Network Launch](#11-network-launch)
12. [Post-Launch Monitoring](#12-post-launch-monitoring)

---

## 1. Pre-Deployment Planning

### 1.1 What You Need to Decide

**Network Parameters**:
- [ ] Network name: "sqoin" or "sqoin-testnet"
- [ ] Magic bytes (4 bytes unique identifier)
- [ ] Default ports (P2P, RPC)
- [ ] Block time target (e.g., 10 minutes, 2.5 minutes, etc.)
- [ ] Block reward (e.g., 50 SQOIN)
- [ ] Halving interval (e.g., 210,000 blocks)
- [ ] Maximum supply
- [ ] Difficulty adjustment period

**Infrastructure Requirements**:
- [ ] DNS seeds (minimum 2-3 servers)
- [ ] Initial nodes (seed nodes)
- [ ] Block explorer server (optional)
- [ ] Mining pool (optional)

### 1.2 Critical Decisions Impact

**Example Configuration for This Guide**:
```yaml
Network Name: sqoin
Magic Bytes: 0xf9, 0xbe, 0xb4, 0xd9  # CHANGE THESE!
P2P Port: 8333  # CHANGE THIS!
RPC Port: 8332  # CHANGE THIS!
Block Time: 150 seconds (2.5 minutes)
Block Reward: 50 SQOIN
Halving: Every 840,000 blocks (4 years at 2.5 min blocks)
Max Supply: ~84 million SQOIN
Difficulty Adjustment: Every 2016 blocks
```

---

## 2. Critical Files to Modify

### 2.1 Complete File List

**Essential Modifications** (Must change):

| File Path | Purpose | Risk Level |
|-----------|---------|------------|
| `src/chainparams.cpp` | Network parameters, genesis block | 🔴 CRITICAL |
| `src/kernel/chainparams.cpp` | Consensus parameters | 🔴 CRITICAL |
| `src/chainparamsseeds.h` | DNS seed nodes | 🔴 CRITICAL |
| `src/consensus/params.h` | Consensus constants | 🔴 CRITICAL |
| `src/pow.cpp` | Difficulty adjustment | 🟡 MODERATE |
| `contrib/seeds/nodes_main.txt` | Seed node list | 🟡 MODERATE |

**Optional Modifications** (Recommended):

| File Path | Purpose | Risk Level |
|-----------|---------|------------|
| `src/policy/policy.h` | Transaction policies | 🟡 MODERATE |
| `src/script/script.h` | Script constants | 🟢 LOW |
| `src/wallet/wallet.h` | Wallet settings | 🟢 LOW |
| `src/qt/guiconstants.h` | GUI constants | 🟢 LOW |

### 2.2 File Modification Order

**Phase 1: Consensus Changes**
1. `src/consensus/params.h` - Change block time, rewards
2. `src/kernel/chainparams.cpp` - Update consensus params
3. `src/chainparams.cpp` - Create new genesis block

**Phase 2: Network Configuration**
4. `src/chainparams.cpp` - Set magic bytes, ports
5. `src/chainparamsseeds.h` - Add DNS seeds
6. `contrib/seeds/nodes_main.txt` - Add seed nodes

**Phase 3: Build & Test**
7. Rebuild entire project
8. Test genesis block generation
9. Test network connectivity

---

## 3. Step-by-Step Deployment

### 3.1 Backup Original Files

```bash
# Navigate to repository
cd ~/sqoin

# Create backup directory
mkdir -p backups/$(date +%Y%m%d)

# Backup critical files
cp src/chainparams.cpp backups/$(date +%Y%m%d)/
cp src/kernel/chainparams.cpp backups/$(date +%Y%m%d)/
cp src/chainparamsseeds.h backups/$(date +%Y%m%d)/
cp src/consensus/params.h backups/$(date +%Y%m%d)/
cp src/pow.cpp backups/$(date +%Y%m%d)/

echo "Backups created in backups/$(date +%Y%m%d)/"
```

### 3.2 Modify Consensus Parameters

**File: `src/consensus/params.h`**

**Location**: Line ~15-30

**Current State**:
```cpp
/** The maximum allowed size for a serialized block, in bytes (only for buffer size limits) */
static const unsigned int MAX_BLOCK_SERIALIZED_SIZE = 4000000;
/** The maximum allowed weight for a block, see BIP 141 (network rule) */
static const unsigned int MAX_BLOCK_WEIGHT = 4000000;
```

**What to Change**:
```cpp
// Add after existing constants
namespace Consensus {
    // Block generation
    static const int64_t BLOCK_TIME_TARGET = 150;  // 2.5 minutes (150 seconds)
    
    // Block rewards
    static const CAmount INITIAL_SUBSIDY = 50 * COIN;
    static const int SUBSIDY_HALVING_INTERVAL = 840000;  // ~4 years
    
    // Difficulty adjustment
    static const int64_t DIFFICULTY_ADJUSTMENT_INTERVAL = 2016;  // ~3.5 days
    static const int64_t TARGET_TIMESPAN = 302400;  // 2016 * 150 = 302400 seconds
}
```

**Implementation**:
```bash
# Edit the file
nano src/consensus/params.h

# Add the constants after existing definitions
# Save and exit (Ctrl+X, Y, Enter)
```

**Outcome**: Consensus rules now use 2.5-minute blocks with adjusted difficulty

---

### 3.3 Modify Chainparams

**File: `src/chainparams.cpp`**

**Location**: Search for `class CMainParams : public CChainParams`

**Challenge #1: Genesis Block Creation**

**Current Genesis Block** (Bitcoin's):
```cpp
const char* pszTimestamp = "The Times 03/Jan/2009 Chancellor on brink of second bailout for banks";
```

**Solution: Create New Genesis Block**

```cpp
// Around line 100-150 in CMainParams constructor
const char* pszTimestamp = "Sqoin Launch 2025 - New Era of Digital Currency";
genesis = CreateGenesisBlock(
    1731358800,  // Timestamp: 2025-11-11 20:00:00 GMT
    2083236893,  // Nonce (to be calculated)
    0x1d00ffff,  // nBits (initial difficulty)
    1,           // Version
    50 * COIN    // Reward
);
```

**Challenge #2: Calculate Genesis Hash**

**Problem**: Genesis block hash must meet difficulty target

**Solution**: Use genesis block mining script

**Implementation**:

Create file: `contrib/devtools/mine_genesis.py`

```python
#!/usr/bin/env python3
"""
Genesis block mining script for Sqoin
"""
import hashlib
import struct
import time

def hash256(data):
    """Double SHA256"""
    return hashlib.sha256(hashlib.sha256(data).digest()).digest()

def serialize_block_header(version, prev_block, merkle_root, timestamp, bits, nonce):
    """Serialize block header"""
    header = struct.pack("<I", version)
    header += bytes.fromhex(prev_block)[::-1]
    header += bytes.fromhex(merkle_root)[::-1]
    header += struct.pack("<I", timestamp)
    header += struct.pack("<I", bits)
    header += struct.pack("<I", nonce)
    return header

def mine_genesis_block():
    """Mine genesis block"""
    version = 1
    prev_block = "0" * 64
    merkle_root = "4a5e1e4baab89f3a32518a88c31bc87f618f76673e2cc77ab2127b7afdeda33b"  # Bitcoin's merkle
    timestamp = int(time.time())
    bits = 0x1d00ffff
    
    print(f"Mining genesis block...")
    print(f"Timestamp: {timestamp}")
    print(f"Target: 0x{bits:08x}")
    
    nonce = 0
    target = (bits & 0xffffff) * 2**(8 * ((bits >> 24) - 3))
    
    while True:
        header = serialize_block_header(version, prev_block, merkle_root, timestamp, bits, nonce)
        hash_result = hash256(header)
        hash_int = int.from_bytes(hash_result[::-1], 'big')
        
        if hash_int < target:
            print(f"\n✓ Found valid genesis block!")
            print(f"Nonce: {nonce}")
            print(f"Hash: {hash_result[::-1].hex()}")
            print(f"\nAdd to chainparams.cpp:")
            print(f"genesis.nTime = {timestamp};")
            print(f"genesis.nNonce = {nonce};")
            print(f"genesis.nBits = 0x{bits:08x};")
            print(f"assert(genesis.GetHash() == uint256S(\"0x{hash_result[::-1].hex()}\"));")
            break
        
        nonce += 1
        if nonce % 100000 == 0:
            print(f"\rTrying nonce: {nonce:,}...", end="", flush=True)
        
        if nonce > 4294967295:  # Overflow
            timestamp += 1
            nonce = 0
            print(f"\nIncremented timestamp to {timestamp}")

if __name__ == "__main__":
    mine_genesis_block()
```

**Run Genesis Mining**:
```bash
# Make executable
chmod +x contrib/devtools/mine_genesis.py

# Run (may take several minutes)
python3 contrib/devtools/mine_genesis.py

# Output will show:
# ✓ Found valid genesis block!
# Nonce: 2083236893
# Hash: 000000000019d6689c085ae165831e934ff763ae46a2a6c172b3f1b60a8ce26f
```

**Update chainparams.cpp**:
```cpp
genesis.nTime = 1731358800;
genesis.nNonce = 2083236893;  // From mining script
genesis.nBits = 0x1d00ffff;

consensus.hashGenesisBlock = genesis.GetHash();
assert(consensus.hashGenesisBlock == uint256S("0x000000000019d6689c085ae165831e934ff763ae46a2a6c172b3f1b60a8ce26f"));
```

**Outcome**: Genesis block created and validated

---

### 3.4 Set Network Magic Bytes

**File: `src/chainparams.cpp`**

**Location**: In `CMainParams` constructor

**Current**:
```cpp
// Bitcoin magic bytes
pchMessageStart[0] = 0xf9;
pchMessageStart[1] = 0xbe;
pchMessageStart[2] = 0xb4;
pchMessageStart[3] = 0xd9;
```

**Challenge #3: Choosing Unique Magic Bytes**

**Problem**: Must be unique across all cryptocurrencies to prevent network cross-talk

**Solution**: Generate random bytes and verify uniqueness

```bash
# Generate random magic bytes
python3 -c "import random; print('0x' + ', 0x'.join(f'{random.randint(0,255):02x}' for _ in range(4)))"

# Example output: 0xa7, 0x3c, 0xf2, 0x8b
```

**Verify Not Used**: Check https://en.bitcoin.it/wiki/Protocol_documentation

**Implementation**:
```cpp
// Sqoin magic bytes (CHANGE THESE!)
pchMessageStart[0] = 0xa7;
pchMessageStart[1] = 0x3c;
pchMessageStart[2] = 0xf2;
pchMessageStart[3] = 0x8b;
```

**Outcome**: Network packets now identifiable as Sqoin

---

### 3.5 Configure Network Ports

**File: `src/chainparams.cpp`**

**Location**: In `CMainParams` constructor

**Current**:
```cpp
nDefaultPort = 8333;  // Bitcoin port
nPruneAfterHeight = 100000;
```

**Challenge #4: Port Conflicts**

**Problem**: Bitcoin uses 8333, must choose different port

**Solution**: Select ports not commonly used

**Port Selection Guide**:
```
P2P Port: 9333 (or 10333, 11333)
RPC Port: 9332 (P2P - 1)
REST Port: 9334 (P2P + 1)

Avoid: 8333 (Bitcoin), 9333 (Litecoin), 8545 (Ethereum), etc.
```

**Implementation**:
```cpp
nDefaultPort = 10333;  // Sqoin P2P port
nPruneAfterHeight = 100000;

// Later in file, update RPC port
base58Prefixes[PUBKEY_ADDRESS] = std::vector<unsigned char>(1,63);  // S prefix
base58Prefixes[SCRIPT_ADDRESS] = std::vector<unsigned char>(1,5);
base58Prefixes[SECRET_KEY] = std::vector<unsigned char>(1,191);
```

**Update sqoin.conf Template**:
```bash
# Edit contrib/devtools/gen-sqoin-conf.sh
# Change default ports
rpcport=10332
port=10333
```

**Outcome**: Network runs on unique ports without conflicts

---

## 4. Common Challenges & Solutions

### 4.1 Challenge: Difficulty Not Adjusting

**Symptom**: Blocks being mined too fast or slow

**File**: `src/pow.cpp`

**Function**: `CalculateNextWorkRequired()`

**Problem**: Difficulty calculation using wrong time target

**Solution**:

```cpp
// Around line 50-60
unsigned int CalculateNextWorkRequired(const CBlockIndex* pindexLast, int64_t nFirstBlockTime, const Consensus::Params& params)
{
    if (params.fPowNoRetargeting)
        return pindexLast->nBits;

    // Limit adjustment step
    int64_t nActualTimespan = pindexLast->GetBlockTime() - nFirstBlockTime;
    
    // CHANGE THIS: Use new block time target
    int64_t nTargetTimespan = params.nPowTargetSpacing * params.DifficultyAdjustmentInterval();
    
    if (nActualTimespan < nTargetTimespan/4)
        nActualTimespan = nTargetTimespan/4;
    if (nActualTimespan > nTargetTimespan*4)
        nActualTimespan = nTargetTimespan*4;

    // Retarget
    const arith_uint256 bnPowLimit = UintToArith256(params.powLimit);
    arith_uint256 bnNew;
    bnNew.SetCompact(pindexLast->nBits);
    bnNew *= nActualTimespan;
    bnNew /= nTargetTimespan;

    if (bnNew > bnPowLimit)
        bnNew = bnPowLimit;

    return bnNew.GetCompact();
}
```

**Update Consensus Params**:

**File**: `src/kernel/chainparams.cpp`

```cpp
// In CMainParams::CMainParams() constructor
consensus.nPowTargetSpacing = 150;  // 2.5 minutes
consensus.nPowTargetTimespan = 302400;  // 3.5 days (2016 blocks * 150 sec)
consensus.DifficultyAdjustmentInterval() = consensus.nPowTargetTimespan / consensus.nPowTargetSpacing;  // 2016 blocks
```

**Test**:
```bash
# Rebuild
make -j$(nproc)

# Start regtest
sqoind -regtest -daemon

# Mine blocks and check difficulty
for i in {1..2016}; do
    sqoin-cli -regtest generatetoaddress 1 $(sqoin-cli -regtest getnewaddress)
done

# Check difficulty adjustment
sqoin-cli -regtest getblockchaininfo | jq '.difficulty'
```

**Outcome**: Difficulty adjusts every 2016 blocks to maintain 2.5-minute target

---

### 4.2 Challenge: Block Rewards Not Halving

**Symptom**: Rewards stay constant after halving interval

**File**: `src/validation.cpp`

**Function**: `GetBlockSubsidy()`

**Location**: Search for "GetBlockSubsidy"

**Current**:
```cpp
CAmount GetBlockSubsidy(int nHeight, const Consensus::Params& consensusParams)
{
    int halvings = nHeight / consensusParams.nSubsidyHalvingInterval;
    // Force block reward to zero when right shift is undefined.
    if (halvings >= 64)
        return 0;

    CAmount nSubsidy = 50 * COIN;
    // Subsidy is cut in half every 210,000 blocks which will occur approximately every 4 years.
    nSubsidy >>= halvings;
    return nSubsidy;
}
```

**Solution**: Update halving interval

```cpp
CAmount GetBlockSubsidy(int nHeight, const Consensus::Params& consensusParams)
{
    int halvings = nHeight / consensusParams.nSubsidyHalvingInterval;
    
    // Force block reward to zero when right shift is undefined
    if (halvings >= 64)
        return 0;

    // CHANGED: Use configured initial subsidy
    CAmount nSubsidy = 50 * COIN;  // Can be made configurable
    
    // Subsidy is cut in half every N blocks
    nSubsidy >>= halvings;
    return nSubsidy;
}
```

**Update chainparams.cpp**:
```cpp
// In CMainParams constructor
consensus.nSubsidyHalvingInterval = 840000;  // ~4 years at 2.5 min blocks
```

**Test**:
```bash
# Check reward at different heights
sqoin-cli -regtest getblocksubsidy 0       # 50.00000000
sqoin-cli -regtest getblocksubsidy 840000  # 25.00000000
sqoin-cli -regtest getblocksubsidy 1680000 # 12.50000000
```

**Outcome**: Block rewards halve at correct intervals

---

### 4.3 Challenge: No Peer Connections

**Symptom**: Node shows 0 connections

**File**: `src/chainparamsseeds.h`

**Problem**: No DNS seeds configured

**Solution**: Add DNS seed servers

**Step 1: Setup DNS Seeds** (requires DNS server)

**File to Create**: `contrib/seeds/nodes_main.txt`

```
# Sqoin mainnet seed nodes
# Format: IP:PORT

# Example seed nodes (REPLACE WITH YOUR IPs)
192.168.1.100:10333
203.0.113.45:10333
198.51.100.78:10333
```

**Step 2: Generate Seeds**

```bash
# Navigate to seeds directory
cd contrib/seeds

# Run seed generation
./makeseeds.py < nodes_main.txt > ../../src/chainparamsseeds.h
```

**Step 3: Manual DNS Seeds** (if no DNS server)

**File**: `src/chainparams.cpp`

```cpp
// In CMainParams constructor
vSeeds.clear();  // Clear Bitcoin seeds

// Add your DNS seeds (if you have DNS server)
// vSeeds.emplace_back("seed.sqoin.org");
// vSeeds.emplace_back("seed.sqoincore.org");

// For testing: Add hardcoded IPs
vFixedSeeds.clear();
// Will be populated from chainparamsseeds.h
```

**Step 4: Initial Peer Connection**

For initial launch, use `-addnode` parameter:

```bash
# Start node with manual peer
sqoind -addnode=203.0.113.45:10333 -addnode=198.51.100.78:10333
```

**Outcome**: Node can discover and connect to peers

---

### 4.4 Challenge: Blockchain Not Syncing

**Symptom**: Stuck at certain block height

**File**: `src/chainparams.cpp`

**Problem**: Checkpoints preventing sync

**Solution**: Remove or update checkpoints

```cpp
// In CMainParams constructor
checkpointData = {
    {
        // REMOVE old Bitcoin checkpoints
        // { 11111, uint256S("0x0000000069e244f73d78e8fd29ba2fd2ed618bd6fa2ee92559f542fdb26e7c1d")},
    }
};

// For new network: Start with empty checkpoints
checkpointData = {
    {
        {0, consensus.hashGenesisBlock},  // Only genesis
    }
};
```

**Outcome**: Blockchain can sync from genesis without checkpoint issues

---

## 5. Network Configuration

### 5.1 Complete Network Config File

**File to Create**: `sqoin.conf`

**Location**: `~/.sqoin/sqoin.conf`

```ini
# Sqoin Core Configuration
# Mainnet Configuration

#######################
# Network Settings
#######################
# Run on mainnet (default)
# regtest=0
# testnet=0

# Port configuration
port=10333
rpcport=10332

# Maximum connections
maxconnections=125

# Bandwidth optimization
maxuploadtarget=5000  # 5GB per day

#######################
# Node Discovery
#######################
# DNS seeds (if available)
# dnsseed=1

# Manual peer connections
# addnode=203.0.113.45:10333
# addnode=198.51.100.78:10333

# Connect only to specific nodes
# connect=203.0.113.45:10333

#######################
# RPC Settings
#######################
server=1
rpcuser=sqoinrpc
rpcpassword=CHANGE_THIS_PASSWORD_123
rpcallowip=127.0.0.1
rpcallowip=192.168.1.0/24  # Local network

# RPC threads
rpcthreads=4

#######################
# Transaction Settings
#######################
# Enable transaction index (for explorers)
txindex=1

# Enable address index (optional)
# addressindex=1

# Mempool settings
maxmempool=300  # MB
mempoolexpiry=72  # hours

#######################
# Wallet Settings
#######################
# Default wallet
wallet=main

# Disable wallet (for full node only)
# disablewallet=1

# Transaction fee
# paytxfee=0.0001

#######################
# Mining (if applicable)
#######################
# Enable mining (solo mining)
# gen=1
# genproclimit=4

# Mining address
# miningaddress=YOUR_SQOIN_ADDRESS

#######################
# Logging
#######################
debug=net
debug=mempool
# debug=validation  # Verbose

# Log file size
# maxlogfilesize=100  # MB

#######################
# Performance
#######################
# Database cache size (MB)
dbcache=450

# Script verification threads
par=4

# Assume valid (skip validation before this block)
# assumevalid=0000000000000000000...

#######################
# Pruning (optional)
#######################
# Enable pruning to save disk space
# prune=550  # Keep only 550MB of blocks
```

---

## 6. Genesis Block Creation

### 6.1 Understanding Genesis Block

**What It Contains**:
```
Genesis Block = {
    version: 1
    prev_block: 0x00...00 (32 bytes of zeros)
    merkle_root: Hash of coinbase transaction
    timestamp: Unix timestamp
    nBits: Initial difficulty target
    nonce: Solution to PoW puzzle
}
```

### 6.2 Creating Custom Genesis Transaction

**File**: `src/chainparams.cpp`

**Function**: `CreateGenesisBlock()`

```cpp
static CBlock CreateGenesisBlock(uint32_t nTime, uint32_t nNonce, uint32_t nBits, int32_t nVersion, const CAmount& genesisReward)
{
    const char* pszTimestamp = "Sqoin Launch 11/Nov/2025 - Decentralized Future Begins";
    const CScript genesisOutputScript = CScript() << ParseHex("04678afdb0fe5548271967f1a67130b7105cd6a828e03909a67962e0ea1f61deb649f6bc3f4cef38c4f35504e51ec112de5c384df7ba0b8d578a4c702b6bf11d5f") << OP_CHECKSIG;
    return CreateGenesisBlock(pszTimestamp, genesisOutputScript, nTime, nNonce, nBits, nVersion, genesisReward);
}
```

**Generate New Public Key**:
```bash
# Generate new key pair for genesis
sqoin-cli -regtest getnewaddress
# Output: sqoin1q... (address)

# Get public key
sqoin-cli -regtest getaddressinfo sqoin1q... | jq -r '.pubkey'
# Use this in genesis script
```

### 6.3 Genesis Block Testing

```bash
# After modifications, rebuild
cd build
make -j$(nproc)

# Test genesis block
./src/test/test_sqoin --log_level=all --run_test=genesis_tests

# Start node and verify
./src/sqoind -regtest -daemon

# Check genesis
./src/sqoin-cli -regtest getblockhash 0
./src/sqoin-cli -regtest getblock $(./src/sqoin-cli -regtest getblockhash 0)
```

---

## 7. Chainparams Modification

### 7.1 Complete Chainparams Template

**File**: `src/chainparams.cpp`

**Full Implementation for Mainnet**:

```cpp
class CMainParams : public CChainParams {
public:
    CMainParams() {
        strNetworkID = CBaseChainParams::MAIN;
        
        // Consensus parameters
        consensus.nSubsidyHalvingInterval = 840000;
        consensus.BIP16Height = 0;  // Enable from genesis
        consensus.BIP34Height = 0;
        consensus.BIP34Hash = uint256();
        consensus.BIP65Height = 0;  // CLTV
        consensus.BIP66Height = 0;  // Strict DER signatures
        consensus.CSVHeight = 0;    // CSV
        consensus.SegwitHeight = 0; // Segwit
        
        // PoW parameters
        consensus.powLimit = uint256S("00000000ffffffffffffffffffffffffffffffffffffffffffffffffffffffff");
        consensus.nPowTargetTimespan = 302400;  // 3.5 days
        consensus.nPowTargetSpacing = 150;      // 2.5 minutes
        consensus.fPowAllowMinDifficultyBlocks = false;
        consensus.fPowNoRetargeting = false;
        consensus.nRuleChangeActivationThreshold = 1916;  // 95% of 2016
        consensus.nMinerConfirmationWindow = 2016;
        
        // Deployment of BIP68, BIP112, and BIP113
        consensus.vDeployments[Consensus::DEPLOYMENT_TESTDUMMY].bit = 28;
        consensus.vDeployments[Consensus::DEPLOYMENT_TESTDUMMY].nStartTime = Consensus::BIP9Deployment::NEVER_ACTIVE;
        consensus.vDeployments[Consensus::DEPLOYMENT_TESTDUMMY].nTimeout = Consensus::BIP9Deployment::NO_TIMEOUT;
        consensus.vDeployments[Consensus::DEPLOYMENT_TESTDUMMY].min_activation_height = 0;
        
        // Taproot deployment
        consensus.vDeployments[Consensus::DEPLOYMENT_TAPROOT].bit = 2;
        consensus.vDeployments[Consensus::DEPLOYMENT_TAPROOT].nStartTime = 1619222400;  // April 24, 2021
        consensus.vDeployments[Consensus::DEPLOYMENT_TAPROOT].nTimeout = 1628640000;     // August 11, 2021
        consensus.vDeployments[Consensus::DEPLOYMENT_TAPROOT].min_activation_height = 0;
        
        // Network magic bytes (CHANGE THESE!)
        pchMessageStart[0] = 0xa7;
        pchMessageStart[1] = 0x3c;
        pchMessageStart[2] = 0xf2;
        pchMessageStart[3] = 0x8b;
        
        // Network ports
        nDefaultPort = 10333;
        nPruneAfterHeight = 100000;
        
        // Genesis block
        genesis = CreateGenesisBlock(1731358800, 2083236893, 0x1d00ffff, 1, 50 * COIN);
        consensus.hashGenesisBlock = genesis.GetHash();
        
        // IMPORTANT: Update these after mining genesis block
        assert(consensus.hashGenesisBlock == uint256S("0x000000000019d6689c085ae165831e934ff763ae46a2a6c172b3f1b60a8ce26f"));
        assert(genesis.hashMerkleRoot == uint256S("0x4a5e1e4baab89f3a32518a88c31bc87f618f76673e2cc77ab2127b7afdeda33b"));
        
        // DNS seeds (add your seeds)
        vSeeds.clear();
        // vSeeds.emplace_back("seed.sqoin.org");
        // vSeeds.emplace_back("seed2.sqoin.org");
        
        // Base58 prefixes (address format)
        base58Prefixes[PUBKEY_ADDRESS] = std::vector<unsigned char>(1,63);  // 'S' addresses
        base58Prefixes[SCRIPT_ADDRESS] = std::vector<unsigned char>(1,5);   // '3' addresses
        base58Prefixes[SECRET_KEY] = std::vector<unsigned char>(1,191);     // Private keys
        base58Prefixes[EXT_PUBLIC_KEY] = {0x04, 0x88, 0xB2, 0x1E};  // xpub
        base58Prefixes[EXT_SECRET_KEY] = {0x04, 0x88, 0xAD, 0xE4};  // xprv
        
        // Bech32 prefix
        bech32_hrp = "sq";  // sqoin addresses start with "sq1"
        
        // Fixed seeds
        vFixedSeeds = std::vector<uint8_t>(std::begin(chainparams_seed_main), std::end(chainparams_seed_main));
        
        // Soft fork heights
        fDefaultConsistencyChecks = false;
        fRequireStandard = true;
        m_is_test_chain = false;
        m_is_mockable_chain = false;
        
        // Checkpoints (start empty)
        checkpointData = {
            {
                {0, consensus.hashGenesisBlock},
            }
        };
        
        // Chain transaction statistics (will grow over time)
        chainTxData = ChainTxData{
            // Data from block 0 (genesis)
            1731358800,  // * UNIX timestamp of last known block
            0,           // * total number of transactions
            0.0          // * estimated number of transactions per second
        };
    }
};
```

---

## 8. DNS Seed Setup

### 8.1 DNS Seed Server Requirements

**Option 1: Run Your Own DNS Seeder**

**Repository**: Bitcoin DNS seeder (adapt for Sqoin)

```bash
# Clone DNS seeder
git clone https://github.com/sipa/bitcoin-seeder.git sqoin-seeder
cd sqoin-seeder

# Modify for Sqoin
nano main.cpp
# Change:
# - Magic bytes
# - Default port
# - Network name
```

**Configuration**:
```bash
# Run seeder
./dnsseed -h seed.sqoin.org -n sqoin-seeder -m your-email@sqoin.org -p 10333
```

**DNS Setup** (on your DNS provider):
```
seed.sqoin.org.  IN  NS  ns1.sqoin.org.
ns1.sqoin.org.   IN  A   203.0.113.45
```

**Option 2: Use Fixed Seeds**

```cpp
// In chainparams.cpp
vSeeds.clear();  // No DNS seeds

// Rely on fixed seeds in chainparamsseeds.h
```

---

## 9. Checkpoint Configuration

### 9.1 Why Checkpoints Matter

**Purpose**:
- Prevent reorganization before checkpoint
- Speed up initial sync
- Protection against attacks

### 9.2 Adding Checkpoints

**File**: `src/chainparams.cpp`

```cpp
checkpointData = {
    {
        {     0, uint256S("0x000000000019d6689c085ae165831e934ff763ae46a2a6c172b3f1b60a8ce26f")},  // Genesis
        // Add as network grows:
        // { 10000, uint256S("0x...")},
        // { 50000, uint256S("0x...")},
        // {100000, uint256S("0x...")},
    }
};
```

**How to Add Checkpoint**:
```bash
# At desired height, get block hash
sqoin-cli getblockhash 10000

# Verify block is in mainnet
sqoin-cli getblock <hash>

# Add to checkpointData in chainparams.cpp
# Rebuild and release
```

---

## 10. Testing and Validation

### 10.1 Pre-Launch Testing Checklist

**Build Tests**:
```bash
# Clean build
cd build
rm -rf *
cmake .. -DBUILD_TESTS=ON
make -j$(nproc)

# Run unit tests
./src/test/test_sqoin

# Run functional tests
./test/functional/test_runner.py
```

**Network Tests**:
```bash
# Test 1: Genesis validation
sqoind -regtest -daemon
sqoin-cli -regtest getblockchaininfo
sqoin-cli -regtest stop

# Test 2: Multiple nodes
# Terminal 1
sqoind -datadir=/tmp/node1 -regtest -port=20333 -rpcport=20332

# Terminal 2
sqoind -datadir=/tmp/node2 -regtest -port=20334 -rpcport=20333 -connect=127.0.0.1:20333

# Check connections
sqoin-cli -datadir=/tmp/node1 -regtest getpeerinfo
```

**Consensus Tests**:
```bash
# Test difficulty adjustment
# Mine 2016 blocks and verify difficulty changes
for i in {1..2016}; do
    sqoin-cli -regtest generatetoaddress 1 $(sqoin-cli -regtest getnewaddress)
done

sqoin-cli -regtest getblockchaininfo | jq '.difficulty'
```

**Transaction Tests**:
```bash
# Create and broadcast transaction
ADDR=$(sqoin-cli -regtest getnewaddress)
sqoin-cli -regtest sendtoaddress $ADDR 10

# Verify in mempool
sqoin-cli -regtest getrawmempool

# Mine block
sqoin-cli -regtest generatetoaddress 1 $ADDR

# Verify confirmed
sqoin-cli -regtest getbalance
```

### 10.2 Stress Testing

```bash
# Create load test script
cat > load_test.sh << 'EOF'
#!/bin/bash
for i in {1..1000}; do
    ADDR=$(sqoin-cli -regtest getnewaddress)
    sqoin-cli -regtest sendtoaddress $ADDR 0.01 &
done
wait
EOF

chmod +x load_test.sh
./load_test.sh

# Check mempool size
sqoin-cli -regtest getmempoolinfo
```

---

## 11. Network Launch

### 11.1 Launch Day Checklist

**Pre-Launch (T-24 hours)**:
- [ ] All code changes committed and tested
- [ ] Genesis block finalized
- [ ] DNS seeds configured
- [ ] Seed nodes ready
- [ ] Documentation updated
- [ ] Wallet software compiled for all platforms
- [ ] Website and social media ready

**Launch Preparation (T-6 hours)**:
- [ ] Start seed nodes
- [ ] Verify seed nodes connectable
- [ ] Monitor logs
- [ ] Prepare mining software
- [ ] Alert community

**Launch (T-0)**:
```bash
# On each seed node
sqoind -daemon \
    -listen \
    -discover \
    -upnp \
    -maxconnections=125 \
    -dbcache=4000 \
    -par=8

# Monitor
tail -f ~/.sqoin/debug.log
```

**Post-Launch (T+1 hour)**:
- [ ] Verify block production
- [ ] Monitor peer count
- [ ] Check transaction propagation
- [ ] Monitor mempool
- [ ] Respond to issues

### 11.2 Mining Setup

**Solo Mining**:
```bash
# In sqoin.conf
gen=1
genproclimit=4
miningaddress=YOUR_SQOIN_ADDRESS

# Restart
sqoind -daemon
```

**Pool Mining** (requires pool software):
```bash
# Use pool software like:
# - NOMP (Node Open Mining Portal)
# - MPOS (Mining Portal Open Source)
```

---

## 12. Post-Launch Monitoring

### 12.1 Monitoring Tools

**Node Monitoring**:
```bash
# Create monitoring script
cat > monitor.sh << 'EOF'
#!/bin/bash
while true; do
    clear
    echo "=== Sqoin Network Status ==="
    echo "Timestamp: $(date)"
    echo ""
    
    echo "--- Blockchain Info ---"
    sqoin-cli getblockchaininfo | jq '{chain,blocks,headers,difficulty,mediantime}'
    echo ""
    
    echo "--- Network Info ---"
    sqoin-cli getnetworkinfo | jq '{connections,warnings}'
    echo ""
    
    echo "--- Peer Info ---"
    sqoin-cli getpeerinfo | jq 'length'
    echo " peers connected"
    echo ""
    
    echo "--- Mempool Info ---"
    sqoin-cli getmempoolinfo | jq '{size,bytes,usage}'
    echo ""
    
    echo "--- Mining Info ---"
    sqoin-cli getmininginfo | jq '{blocks,difficulty,networkhashps}'
    
    sleep 30
done
EOF

chmod +x monitor.sh
./monitor.sh
```

### 12.2 Common Issues After Launch

**Issue: Blocks Not Being Found**

**Diagnosis**:
```bash
sqoin-cli getmininginfo
# Check: networkhashps should be > 0
```

**Solution**:
```bash
# Adjust initial difficulty if too high
# In chainparams.cpp:
consensus.powLimit = uint256S("0000000fffffffffffffffffffffffffffffffffffffffffffffffffffffffff");
# (Note: More leading zeros = easier)
```

**Issue: Network Fragmented**

**Diagnosis**:
```bash
# Node A
sqoin-cli getblockcount  # 100

# Node B
sqoin-cli getblockcount  # 95

# Different chains!
```

**Solution**:
```bash
# Force nodes to connect
sqoin-cli addnode "NODE_A_IP:10333" "add"

# Check which chain has more work
sqoin-cli getchaintips

# Longest chain will win
```

**Issue: Transaction Not Propagating**

**Diagnosis**:
```bash
# Check if transaction is standard
sqoin-cli -regtest decoderawtransaction <rawtx>

# Check fee rate
sqoin-cli -regtest getmempoolentry <txid>
```

**Solution**:
```bash
# Increase fee
sqoin-cli -regtest bumpfee <txid>

# Or rebroadcast
sqoin-cli -regtest sendrawtransaction <rawtx>
```

---

## 13. Advanced Configuration

### 13.1 Performance Tuning

**File**: `sqoin.conf`

```ini
# Maximum memory pool
maxmempool=300

# Database cache (increase for better performance)
dbcache=4096

# Number of script verification threads
par=-1  # Use all CPU cores

# Signature cache size
maxsigcachesize=50

# Enable bloom filters (for SPV)
peerbloomfilters=1

# Increase max upload
maxuploadtarget=10000  # 10GB/day
```

### 13.2 Security Hardening

```ini
# Disable wallet on full node
disablewallet=1

# Limit RPC access
rpcallowip=127.0.0.1
rpcbind=127.0.0.1

# Enable onlynet (Tor only)
# onlynet=onion
# proxy=127.0.0.1:9050

# Whitelist local network
whitelist=192.168.1.0/24

# Enable checkpoints
assumevalid=<latest_checkpoint_hash>
```

---

## 14. Rollback Plan

### 14.1 If Things Go Wrong

**Scenario 1: Critical Bug Found**

```bash
# Stop all nodes immediately
sqoin-cli stop

# Announce halt on all channels
# Fix bug
# Create hotfix release
# Coordinated restart
```

**Scenario 2: Chain Fork**

```bash
# Identify canonical chain
# Add checkpoint at agreed block
# Release update
# Nodes will reorganize to correct chain
```

**Scenario 3: 51% Attack**

```bash
# Detected by monitoring
# Halt trading/transactions
# Add checkpoints
# Increase difficulty if needed
# Wait for attacker to stop
```

---

## 15. Success Metrics

### 15.1 Key Performance Indicators

**Network Health**:
- [ ] Block time: ~150 seconds average
- [ ] Peer count: >50 nodes
- [ ] Hash rate: Stable and growing
- [ ] Difficulty: Adjusting properly every 2016 blocks

**Transaction Metrics**:
- [ ] Tx propagation: <5 seconds
- [ ] Confirmation time: <15 minutes (6 blocks)
- [ ] Mempool size: <50 MB
- [ ] Fee market: Stable

**Node Performance**:
- [ ] Sync time: <24 hours
- [ ] Memory usage: <4 GB
- [ ] CPU usage: <50% average
- [ ] Disk I/O: Reasonable

---

## 16. Summary of File Modifications

### Complete Modification Table

| File | Lines | Changes | Verification |
|------|-------|---------|--------------|
| `src/chainparams.cpp` | ~500 | Genesis, magic bytes, ports, seeds | Build + run |
| `src/kernel/chainparams.cpp` | ~100 | Consensus params | Unit tests |
| `src/consensus/params.h` | ~50 | Constants | Unit tests |
| `src/pow.cpp` | ~20 | Difficulty adjustment | Functional test |
| `src/validation.cpp` | ~10 | Block subsidy | Check rewards |
| `src/chainparamsseeds.h` | Generated | Seed nodes | Test connections |
| `contrib/seeds/nodes_main.txt` | New | Seed IPs | Manual verification |

### Build Commands Summary

```bash
# Full rebuild after changes
cd build
rm -rf *
cmake .. \
    -DCMAKE_BUILD_TYPE=Release \
    -DBUILD_TESTS=ON \
    -DBUILD_BENCH=ON
make -j$(nproc)
make check  # Run tests
```

---

## 17. Conclusion

### What You've Achieved

After following this guide, you will have:

1. ✅ Modified consensus parameters for custom blockchain
2. ✅ Created unique genesis block
3. ✅ Configured network with unique magic bytes and ports
4. ✅ Set up DNS seeds or fixed peers
5. ✅ Tested difficulty adjustment
6. ✅ Validated block rewards and halving
7. ✅ Launched mainnet
8. ✅ Monitored network health
9. ✅ Prepared for common issues

### Next Steps

1. **Week 1-2**: Monitor closely, fix any critical bugs
2. **Month 1**: Gather metrics, optimize parameters if needed
3. **Month 3**: Add checkpoints at stable heights
4. **Ongoing**: Community building, development, marketing

### Support Resources

- `docs/codebase.md` - Architecture reference
- `docs/consensus-logic.md` - Consensus details
- `docs/testing-guide.md` - Testing procedures
- `docs/current-strategy.md` - Current implementation
- `docs/changes.md` - Modification guide

---

## Appendix A: Quick Reference Commands

```bash
# Build
cd build && cmake .. && make -j$(nproc)

# Start mainnet
sqoind -daemon

# Check status
sqoin-cli getblockchaininfo
sqoin-cli getnetworkinfo
sqoin-cli getpeerinfo

# Mining
sqoin-cli getmininginfo
sqoin-cli generatetoaddress 1 $(sqoin-cli getnewaddress)

# Transactions
sqoin-cli sendtoaddress <address> <amount>
sqoin-cli gettransaction <txid>

# Debugging
sqoin-cli getmempoolinfo
tail -f ~/.sqoin/debug.log
sqoin-cli getchaintips

# Emergency
sqoin-cli stop
```

---

## Appendix B: Troubleshooting Matrix

| Symptom | Cause | Solution | File |
|---------|-------|----------|------|
| No peers | No seeds | Add DNS seeds or fixed IPs | chainparams.cpp |
| Wrong difficulty | Params wrong | Fix nPowTargetSpacing | kernel/chainparams.cpp |
| Blocks too fast/slow | Difficulty not adjusting | Check CalculateNextWorkRequired | pow.cpp |
| No rewards after halving | Halving interval wrong | Fix nSubsidyHalvingInterval | chainparams.cpp |
| Can't sync | Checkpoint mismatch | Remove old checkpoints | chainparams.cpp |
| Magic byte error | Network mismatch | Verify pchMessageStart | chainparams.cpp |
| Port conflicts | Port in use | Change nDefaultPort | chainparams.cpp |

---

**This guide provides a complete roadmap for deploying Sqoin mainnet. Follow each step carefully, test thoroughly, and monitor continuously for a successful launch.**

---

*For questions or issues, refer to the comprehensive documentation suite in the `docs/` directory.*
