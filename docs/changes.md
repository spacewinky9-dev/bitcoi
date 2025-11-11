# Sqoin Core - Complete Modification Guide

## Overview

This document provides comprehensive tables of all modifiable components in Sqoin Core. It details what files to change, what parameters to modify, and what the effects will be for complete enhancement and rebranding.

---

## Table of Contents

1. [Consensus Parameters](#1-consensus-parameters)
2. [Network Configuration](#2-network-configuration)
3. [Economic Parameters](#3-economic-parameters)
4. [Core Classes & Functions](#4-core-classes--functions)
5. [Database Modifications](#5-database-modifications)
6. [Wallet Configuration](#6-wallet-configuration)
7. [GUI Customization](#7-gui-customization)
8. [Build System Changes](#8-build-system-changes)
9. [Complete Rebranding Checklist](#9-complete-rebranding-checklist)

---

## 1. Consensus Parameters

### 1.1 Critical Consensus Changes

| Parameter | File Location | Current Value | Effect of Change | Risk Level |
|-----------|--------------|---------------|------------------|------------|
| **Block Time** | `src/kernel/chainparams.cpp` | `nPowTargetSpacing = 600` | Changes block generation rate (600s = 10min) | 🔴 CRITICAL |
| **Difficulty Period** | `src/kernel/chainparams.cpp` | `nPowTargetTimespan = 1209600` | Changes difficulty adjustment frequency (2 weeks) | 🔴 CRITICAL |
| **PoW Algorithm** | `src/pow.cpp` | SHA-256d | Change mining algorithm (requires new implementation) | 🔴 CRITICAL |
| **Block Subsidy** | `src/validation.cpp` - `GetBlockSubsidy()` | `50 * COIN` | Initial mining reward | 🔴 CRITICAL |
| **Halving Interval** | `src/kernel/chainparams.cpp` | `nSubsidyHalvingInterval = 210000` | How often reward halves (~4 years) | 🔴 CRITICAL |
| **Max Block Weight** | `src/consensus/consensus.h` | `MAX_BLOCK_WEIGHT = 4000000` | Maximum block size | 🔴 CRITICAL |
| **Coin Supply** | Calculated | ~21 million | Total coins (halving × interval) | 🔴 CRITICAL |

**Instructions**:
```cpp
// File: src/kernel/chainparams.cpp
// Change block time (example: 2 minutes)
consensus.nPowTargetSpacing = 2 * 60;  // 120 seconds

// Change difficulty adjustment (example: daily)
consensus.nPowTargetTimespan = 24 * 60 * 60;  // 1 day

// Change halving interval (example: every 100k blocks)
consensus.nSubsidyHalvingInterval = 100000;
```

### 1.2 Genesis Block Modification

| Component | File Location | Current Value | Effect of Change |
|-----------|--------------|---------------|------------------|
| **Genesis Timestamp** | `src/kernel/chainparams.cpp` | `1231006505` | Launch date/time | 🔴 CRITICAL |
| **Genesis Message** | `src/kernel/chainparams.cpp` | "The Times 03/Jan/2009..." | Embedded message in first block | 🔴 CRITICAL |
| **Genesis Nonce** | `src/kernel/chainparams.cpp` | `2083236893` | Proof-of-work solution | 🔴 CRITICAL |
| **Genesis Reward** | `src/kernel/chainparams.cpp` | `50 * COIN` | First block reward | 🔴 CRITICAL |
| **Genesis Hash** | `src/kernel/chainparams.cpp` | `000000000019d6...` | Unique chain identifier | 🔴 CRITICAL |

**Instructions for New Genesis Block**:
```cpp
// File: src/kernel/chainparams.cpp

// 1. Change timestamp
uint32_t nTime = GetTime();  // Current timestamp

// 2. Change message
const char* pszTimestamp = "Your custom message here";

// 3. Mine new nonce (run mining to find valid nonce)
uint32_t nNonce = 0;  // Must mine to find valid value

// 4. Create genesis
genesis = CreateGenesisBlock(nTime, nNonce, 0x1d00ffff, 1, 50 * COIN);

// 5. Update hash assertions
consensus.hashGenesisBlock = genesis.GetHash();
assert(consensus.hashGenesisBlock == uint256{"NEW_HASH_HERE"});
```

---

## 2. Network Configuration

### 2.1 Network Parameters

| Parameter | File Location | Current Value | Effect of Change | Risk Level |
|-----------|--------------|---------------|------------------|------------|
| **Magic Bytes** | `src/kernel/chainparams.cpp` | `0xf9, 0xbe, 0xb4, 0xd9` | Network identification (must be unique) | 🔴 CRITICAL |
| **Default Port** | `src/kernel/chainparams.cpp` | `8333` | P2P network port | 🟡 MODERATE |
| **RPC Port** | `src/chainparamsbase.cpp` | `8332` | RPC interface port | 🟢 LOW |
| **DNS Seeds** | `src/kernel/chainparams.cpp` | Bitcoin DNS seeds | Peer discovery servers | 🟡 MODERATE |
| **Fixed Seeds** | `src/chainparamsseeds.h` | Bitcoin node IPs | Hardcoded backup peers | 🟡 MODERATE |

**Instructions**:
```cpp
// File: src/kernel/chainparams.cpp

// Change magic bytes (must be unique for your network)
pchMessageStart[0] = 0xAA;  // Choose random values
pchMessageStart[1] = 0xBB;
pchMessageStart[2] = 0xCC;
pchMessageStart[3] = 0xDD;

// Change default port
nDefaultPort = 9333;  // Choose unused port

// Add your DNS seeds
vSeeds.clear();  // Remove Bitcoin seeds
vSeeds.emplace_back("seed1.sqoin.org.");
vSeeds.emplace_back("seed2.sqoin.org.");
```

### 2.2 DNS Seed Configuration

| Component | File Location | Purpose | How to Modify |
|-----------|--------------|---------|---------------|
| **DNS Seeds List** | `src/kernel/chainparams.cpp` | Peer discovery | Remove Bitcoin seeds, add yours |
| **Seed Generation** | External | Create DNS seed server | Set up separate DNS service |
| **Fixed Seeds** | `src/chainparamsseeds.h` | Hardcoded fallback nodes | Replace with your node IPs |

---

## 3. Economic Parameters

### 3.1 Monetary Policy

| Parameter | File Location | Formula/Value | Effect of Change |
|-----------|--------------|---------------|------------------|
| **COIN Definition** | `src/consensus/amount.h` | `1 COIN = 100000000 satoshis` | Base unit (8 decimals) |
| **Initial Reward** | `src/validation.cpp` | `50 * COIN` | Mining reward at start |
| **Max Money** | `src/consensus/amount.h` | `MAX_MONEY = 21000000 * COIN` | Supply cap |
| **Min Relay Fee** | `src/policy/policy.h` | `DEFAULT_MIN_RELAY_TX_FEE = 1000` | Minimum tx fee (sat/kB) |
| **Dust Threshold** | `src/policy/policy.h` | `DUST_RELAY_TX_FEE = 3000` | Minimum output value |

**Instructions**:
```cpp
// File: src/validation.cpp
CAmount GetBlockSubsidy(int nHeight, const Consensus::Params& consensusParams)
{
    int halvings = nHeight / consensusParams.nSubsidyHalvingInterval;
    if (halvings >= 64)
        return 0;

    // Change initial reward (example: 100 coins)
    CAmount nSubsidy = 100 * COIN;  
    nSubsidy >>= halvings;
    return nSubsidy;
}

// File: src/consensus/amount.h
// Change max supply
static const CAmount MAX_MONEY = 42000000 * COIN;  // 42 million cap
```

---

## 4. Core Classes & Functions

### 4.1 Transaction Processing

| Class/Function | File Location | Purpose | Modification Impact |
|----------------|--------------|---------|---------------------|
| **CTransaction** | `src/primitives/transaction.h` | Transaction structure | 🔴 Changes tx format |
| **CTxIn** | `src/primitives/transaction.h` | Transaction input | 🔴 Changes input format |
| **CTxOut** | `src/primitives/transaction.h` | Transaction output | 🔴 Changes output format |
| **CheckTransaction()** | `src/consensus/tx_check.cpp` | Basic tx validation | 🟡 Changes validation rules |
| **VerifyScript()** | `src/script/interpreter.cpp` | Script execution | 🔴 Changes script behavior |

### 4.2 Block Processing

| Class/Function | File Location | Purpose | Modification Impact |
|----------------|--------------|---------|---------------------|
| **CBlock** | `src/primitives/block.h` | Block structure | 🔴 Changes block format |
| **CBlockHeader** | `src/primitives/block.h` | Block header | 🔴 Changes header format |
| **CheckBlock()** | `src/validation.cpp` | Block validation | 🟡 Changes block rules |
| **ConnectBlock()** | `src/validation.cpp` | Apply block to chain | 🔴 Changes state transitions |
| **GetBlockProof()** | `src/chain.cpp` | Calculate chain work | 🟡 Changes difficulty calculation |

### 4.3 UTXO Management

| Class/Function | File Location | Purpose | Modification Impact |
|----------------|--------------|---------|---------------------|
| **Coin** | `src/coins.h` | UTXO representation | 🟡 Changes UTXO format |
| **CCoinsView** | `src/coins.h` | UTXO set interface | 🟡 Changes storage interface |
| **CCoinsViewCache** | `src/coins.h` | UTXO cache | 🟢 Changes caching behavior |
| **GetCoin()** | `src/coins.cpp` | Retrieve UTXO | 🟢 Changes lookup behavior |
| **BatchWrite()** | `src/coins.cpp` | Write UTXO batch | 🟢 Changes persistence behavior |

---

## 5. Database Modifications

### 5.1 Database Schema

| Database | File Location | Purpose | Modification Risk |
|----------|--------------|---------|-------------------|
| **Block Index** | `src/txdb.h` | Block metadata storage | 🟡 MODERATE |
| **Chainstate** | `src/txdb.h` | UTXO set storage | 🔴 CRITICAL |
| **Wallet DB** | `src/wallet/walletdb.h` | Wallet data storage | 🟡 MODERATE |

### 5.2 Key Prefixes

| Key Type | File Location | Current Prefix | Purpose |
|----------|--------------|----------------|---------|
| **Block Index** | `src/txdb.cpp` | `'b' + block_hash` | Block metadata |
| **UTXO** | `src/txdb.cpp` | `'C' + txid + n` | Coin entries |
| **Block File Info** | `src/txdb.cpp` | `'f' + file_num` | File metadata |
| **Last Block File** | `src/txdb.cpp` | `'l'` | Current file number |

**Instructions**:
```cpp
// File: src/txdb.cpp
// Change database key prefixes if needed
static constexpr uint8_t DB_COIN = 'C';  // Change to different letter
static constexpr uint8_t DB_BLOCK_INDEX = 'b';
// etc.
```

---

## 6. Wallet Configuration

### 6.1 Wallet Parameters

| Parameter | File Location | Current Value | Effect of Change |
|-----------|--------------|---------------|------------------|
| **Keypool Size** | `src/wallet/wallet.h` | `DEFAULT_KEYPOOL_SIZE = 1000` | Number of pre-generated keys |
| **Min Conf** | `src/wallet/wallet.h` | `DEFAULT_MIN_DEPTH = 0` | Minimum confirmations for balance |
| **Coinbase Maturity** | `src/consensus/consensus.h` | `COINBASE_MATURITY = 100` | Blocks before coinbase spendable |
| **Default Fee** | `src/wallet/wallet.h` | Dynamic | Transaction fee calculation |

### 6.2 HD Wallet Paths

| Path Type | File Location | Current Format | Purpose |
|-----------|--------------|----------------|---------|
| **BIP44** | `src/wallet/scriptpubkeyman.cpp` | `m/44'/0'/account'` | Legacy addresses |
| **BIP49** | `src/wallet/scriptpubkeyman.cpp` | `m/49'/0'/account'` | P2SH-wrapped SegWit |
| **BIP84** | `src/wallet/scriptpubkeyman.cpp` | `m/84'/0'/account'` | Native SegWit |
| **BIP86** | `src/wallet/scriptpubkeyman.cpp` | `m/86'/0'/account'` | Taproot |

**Instructions for New Coin Type**:
```cpp
// BIP44 coin type for Sqoin
// Register at: https://github.com/satoshilabs/slips/blob/master/slip-0044.md
// Example: Use coin_type 9999 for Sqoin

// File: src/wallet/scriptpubkeyman.cpp
// Change derivation path
const std::string path = "m/44'/9999'/0'";  // 9999 = your coin type
```

---

## 7. GUI Customization

### 7.1 Visual Elements

| Element | File Location | Current Value | Modification |
|---------|--------------|---------------|--------------|
| **Window Title** | `src/qt/bitcoingui.cpp` | "Sqoin Core" | Already updated |
| **Application Name** | `src/qt/bitcoin.cpp` | "Sqoin-Qt" | Already updated |
| **Icons** | `src/qt/res/icons/` | Bitcoin icons | Replace with Sqoin icons |
| **Splash Screen** | `src/qt/res/images/` | Bitcoin splash | Replace with Sqoin splash |
| **About Dialog** | `src/qt/bitcoingui.cpp` | Version/copyright info | Update branding |

### 7.2 GUI Strings

| String Type | File Location | Examples | How to Change |
|-------------|--------------|----------|---------------|
| **UI Labels** | `src/qt/*.cpp` | "Send Coins", "Receive" | Edit directly in files |
| **Translations** | `src/qt/locale/*.ts` | Translated strings | Update translation files |
| **Help Text** | `src/qt/*.cpp` | Tooltips, descriptions | Edit directly |

---

## 8. Build System Changes

### 8.1 CMake Configuration

| File | Parameters to Change | Purpose |
|------|---------------------|---------|
| **CMakeLists.txt** | `CLIENT_NAME`, `PROJECT_NAME` | Project identification |
| **CMakeLists.txt** | `CLIENT_VERSION_MAJOR/MINOR` | Version numbers |
| **CMakeLists.txt** | `CLIENT_BUGREPORT` | Issue tracker URL |
| **CMakeLists.txt** | `HOMEPAGE_URL` | Project website |

### 8.2 Package Metadata

| File | Purpose | What to Change |
|------|---------|----------------|
| **libsqoinkernel.pc.in** | pkg-config metadata | Library name, description |
| **configure.ac** (if exists) | Autotools config | Project name, version |
| **share/*.desktop** | Desktop entry | Application metadata |

---

## 9. Complete Rebranding Checklist

### 9.1 Phase 1: Basic Rebranding (✅ COMPLETED)

| Task | Files Affected | Status |
|------|---------------|--------|
| Executable names | `src/*.cpp` | ✅ Done |
| Library names | `src/CMakeLists.txt` | ✅ Done |
| Project name | `CMakeLists.txt` | ✅ Done |
| Copyright headers | All source files | ✅ Done |
| Build configuration | `cmake/*.in` | ✅ Done |
| README | `README.md` | ✅ Done |

### 9.2 Phase 2: Network Customization (🔴 REQUIRED FOR LAUNCH)

| Task | Files to Modify | Priority | Difficulty |
|------|----------------|----------|------------|
| **Change Magic Bytes** | `src/kernel/chainparams.cpp` | 🔴 CRITICAL | 🟢 Easy |
| **Change Default Ports** | `src/kernel/chainparams.cpp`, `src/chainparamsbase.cpp` | 🔴 CRITICAL | 🟢 Easy |
| **Create New Genesis** | `src/kernel/chainparams.cpp` | 🔴 CRITICAL | 🟡 Medium |
| **Setup DNS Seeds** | `src/kernel/chainparams.cpp` + External | 🔴 CRITICAL | 🔴 Hard |
| **Generate Fixed Seeds** | `src/chainparamsseeds.h` | 🟡 IMPORTANT | 🟡 Medium |
| **Update Checkpoints** | `src/kernel/chainparams.cpp` | 🟢 OPTIONAL | 🟢 Easy |

### 9.3 Phase 3: Economic Parameters (🟡 CONSIDER CAREFULLY)

| Task | Files to Modify | Risk | Impact |
|------|----------------|------|--------|
| **Block Time** | `src/kernel/chainparams.cpp` | 🔴 HIGH | Changes tx speed |
| **Block Reward** | `src/validation.cpp` | 🔴 HIGH | Changes inflation |
| **Halving Schedule** | `src/kernel/chainparams.cpp` | 🔴 HIGH | Changes supply curve |
| **Max Supply** | `src/consensus/amount.h` | 🔴 HIGH | Changes total coins |
| **Difficulty Adjustment** | `src/kernel/chainparams.cpp` | 🔴 HIGH | Changes mining stability |

### 9.4 Phase 4: Advanced Features (🟢 OPTIONAL)

| Feature | Files to Modify | Complexity | Benefit |
|---------|----------------|------------|---------|
| **New PoW Algorithm** | `src/pow.cpp`, `src/primitives/block.h` | 🔴 VERY HIGH | Unique mining |
| **Custom Script Opcodes** | `src/script/interpreter.cpp` | 🔴 VERY HIGH | New functionality |
| **Modified UTXO Model** | `src/coins.h`, `src/validation.cpp` | 🔴 VERY HIGH | Different accounting |
| **Enhanced Privacy** | Multiple files | 🔴 VERY HIGH | Better privacy |
| **Smart Contracts** | New subsystem | 🔴 EXTREME | Programmability |

---

## 10. Modification Impact Matrix

### 10.1 Risk vs Benefit Analysis

| Modification Type | Consensus Impact | Network Impact | User Impact | Recommended? |
|-------------------|------------------|----------------|-------------|--------------|
| **Branding Only** | None | None | Cosmetic | ✅ YES |
| **Network Params** | None | HIGH | None | ✅ YES |
| **Economic Params** | HIGH | HIGH | HIGH | ⚠️ CAREFUL |
| **Block Time** | HIGH | HIGH | HIGH | ⚠️ CAREFUL |
| **PoW Algorithm** | CRITICAL | CRITICAL | CRITICAL | ❌ EXPERT ONLY |
| **Transaction Format** | CRITICAL | CRITICAL | CRITICAL | ❌ EXPERT ONLY |

### 10.2 Testing Requirements

| Change Type | Test Requirements | Minimum Test Duration |
|-------------|-------------------|----------------------|
| **Branding** | Build test | 1 day |
| **Network Params** | Multi-node testnet | 1 week |
| **Economic Params** | Full testnet with mining | 2-4 weeks |
| **Consensus Changes** | Extensive testnet + audits | 3-6 months |

---

## 11. File Modification Quick Reference

### 11.1 Most Important Files

| File | Purpose | Change Frequency | Risk Level |
|------|---------|------------------|------------|
| **src/kernel/chainparams.cpp** | Network/consensus params | For each network | 🔴 CRITICAL |
| **src/consensus/params.h** | Consensus definitions | Rarely | 🔴 CRITICAL |
| **src/validation.cpp** | Block/tx validation | For rule changes | 🔴 CRITICAL |
| **src/pow.cpp** | Proof-of-work logic | For PoW changes | 🔴 CRITICAL |
| **CMakeLists.txt** | Build configuration | For releases | 🟢 LOW |

### 11.2 Configuration Files

| File | Purpose | Format | When to Change |
|------|---------|--------|----------------|
| **sqoin.conf** | User configuration | INI-style | User configurable |
| **chainparams** | Network rules | C++ code | Network launch |
| **CMakeLists.txt** | Build settings | CMake | Version updates |

---

## 12. Network Launch Checklist

### 12.1 Pre-Launch Requirements

- [ ] Genesis block created and validated
- [ ] Magic bytes changed to unique values
- [ ] Network ports changed (default 8333 → new port)
- [ ] DNS seeds configured and running
- [ ] Fixed seed nodes operational
- [ ] Checkpoint blocks prepared (optional)
- [ ] All branding completed
- [ ] Testnet fully tested (minimum 2 weeks)
- [ ] Documentation updated
- [ ] Build system tested on all platforms

### 12.2 Launch Day Tasks

- [ ] Deploy DNS seed servers
- [ ] Start initial seed nodes (minimum 3)
- [ ] Announce network parameters
- [ ] Release binaries for all platforms
- [ ] Monitor network stability
- [ ] Be ready to add additional seed nodes

---

## 13. Common Modification Patterns

### 13.1 Creating a Faster Blockchain

```cpp
// File: src/kernel/chainparams.cpp

// 1. Reduce block time (example: 2 minutes)
consensus.nPowTargetSpacing = 2 * 60;  // 120 seconds

// 2. Adjust difficulty period (keep ~2 weeks of blocks)
// New period = (2 weeks in seconds) / (new block time)
// = (14 * 24 * 60 * 60) / 120 = 10080 blocks
consensus.nPowTargetTimespan = 14 * 24 * 60 * 60;  // Still 2 weeks

// 3. Consider adjusting halving interval
// To keep ~4 year halvings:
// = (4 years in seconds) / (new block time)
// = (4 * 365 * 24 * 60 * 60) / 120 = 1,051,200 blocks
consensus.nSubsidyHalvingInterval = 1051200;
```

### 13.2 Changing Total Supply

```cpp
// File: src/validation.cpp

CAmount GetBlockSubsidy(int nHeight, const Consensus::Params& consensusParams)
{
    int halvings = nHeight / consensusParams.nSubsidyHalvingInterval;
    if (halvings >= 64)
        return 0;

    // Example: 100 million total supply
    // With halving every 210k blocks
    // Initial reward = 100M / (210k * (1 + 0.5 + 0.25 + ...))
    // = 100M / (210k * 2) = 238.095 coins/block
    CAmount nSubsidy = 23810 * COIN / 100;  // 238.10 coins
    nSubsidy >>= halvings;
    return nSubsidy;
}
```

### 13.3 Implementing Unique Network

```cpp
// File: src/kernel/chainparams.cpp

class CMainParams : public CChainParams {
public:
    CMainParams() {
        // 1. Unique chain type
        m_chain_type = ChainType::MAIN;

        // 2. Unique magic bytes
        pchMessageStart[0] = 0xSQ;  // S
        pchMessageStart[1] = 0x55;  // Q  
        pchMessageStart[2] = 0x4F;  // O
        pchMessageStart[3] = 0x49;  // I

        // 3. Unique port
        nDefaultPort = 9333;

        // 4. Your genesis block
        genesis = CreateGenesisBlock(
            GetTime(),           // Current time
            0,                   // Nonce (mine this)
            0x1d00ffff,         // Initial difficulty
            1,                   // Version
            100 * COIN          // Initial reward
        );

        // 5. Your DNS seeds
        vSeeds.clear();
        vSeeds.emplace_back("seed.sqoin.org.");
        
        // 6. Your checkpoints (add as chain grows)
        checkpointData = {
            {
                {     0, genesis.GetHash()},
                // Add more as network grows
            }
        };
    }
};
```

---

## 14. Testing Modifications

### 14.1 Test Networks

| Network | Purpose | File Location |
|---------|---------|--------------|
| **Mainnet** | Production | `src/kernel/chainparams.cpp` - CMainParams |
| **Testnet** | Public testing | `src/kernel/chainparams.cpp` - CTestNetParams |
| **Regtest** | Local testing | `src/kernel/chainparams.cpp` - CRegTestParams |
| **Signet** | Controlled testing | `src/kernel/chainparams.cpp` - SigNetParams |

### 14.2 Recommended Testing Process

1. **Regtest** (Days 1-3):
   - Test basic functionality
   - Mine blocks easily
   - Fast iteration

2. **Private Testnet** (Week 1-2):
   - Multi-node testing
   - Network synchronization
   - Real mining simulation

3. **Public Testnet** (Weeks 3-6):
   - Community testing
   - Load testing
   - Bug discovery

4. **Mainnet Launch** (After 6+ weeks):
   - All tests passed
   - Security audit complete
   - Community ready

---

## 15. Summary

This guide provides complete modification paths for Sqoin Core. Key points:

1. **Branding Complete**: ✅ All name changes done
2. **Network Launch**: 🔴 Requires new genesis, magic bytes, seeds
3. **Economic Changes**: ⚠️ Optional but significant impact
4. **Advanced Features**: ❌ Require expert knowledge

**Recommended Next Steps**:
1. Create new genesis block
2. Change magic bytes and ports
3. Setup DNS seeds
4. Test on private network
5. Launch public testnet
6. After thorough testing, launch mainnet

**Remember**: Any consensus changes create a new blockchain. Existing Bitcoin blocks cannot be used.

---

**Questions? See**: `docs/codebase.md` for implementation details, `docs/current-strategy.md` for how things currently work.
