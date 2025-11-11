# Sqoin Core - Complete Codebase Documentation

## Table of Contents
1. [Project Overview](#project-overview)
2. [Architecture](#architecture)
3. [Directory Structure](#directory-structure)
4. [Core Components](#core-components)
5. [Libraries](#libraries)
6. [Executables](#executables)
7. [Key Classes and Functions](#key-classes-and-functions)
8. [Module Dependencies](#module-dependencies)
9. [Build System](#build-system)
10. [Testing Framework](#testing-framework)

---

## Project Overview

**Sqoin Core** is a cryptocurrency full node implementation derived from Bitcoin Core. It provides a complete peer-to-peer network client with wallet functionality, transaction validation, and blockchain management capabilities.

### Project Metadata
- **Name**: Sqoin Core
- **Version**: 30.99.0 (development)
- **Language**: C++20
- **Build System**: CMake 3.22+
- **License**: MIT

### Key Features
- Full blockchain validation
- P2P network protocol
- Wallet management with HD support
- JSON-RPC interface
- GUI client (Qt-based, optional)
- ZMQ notifications
- Multi-signature support

---

## Architecture

Sqoin Core follows a modular layered architecture:

```
┌─────────────────────────────────────────────────────────────┐
│                     Application Layer                        │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐      │
│  │   sqoin-qt   │  │   sqoind     │  │  sqoin-cli   │      │
│  │    (GUI)     │  │   (Daemon)   │  │    (RPC)     │      │
│  └──────────────┘  └──────────────┘  └──────────────┘      │
└─────────────────────────────────────────────────────────────┘
                            │
┌─────────────────────────────────────────────────────────────┐
│                    Interface Layer                           │
│  ┌────────────┐  ┌────────────┐  ┌────────────┐            │
│  │   Chain    │  │    Node    │  │   Wallet   │            │
│  │ Interface  │  │ Interface  │  │ Interface  │            │
│  └────────────┘  └────────────┘  └────────────┘            │
└─────────────────────────────────────────────────────────────┘
                            │
┌─────────────────────────────────────────────────────────────┐
│                      Core Layer                              │
│  ┌────────────┐  ┌────────────┐  ┌────────────┐            │
│  │   Node     │  │   Wallet   │  │    RPC     │            │
│  │  (P2P +    │  │  (Key Mgmt │  │  (Server)  │            │
│  │   Server)  │  │  + Coins)  │  │            │            │
│  └────────────┘  └────────────┘  └────────────┘            │
└─────────────────────────────────────────────────────────────┘
                            │
┌─────────────────────────────────────────────────────────────┐
│                   Consensus Layer                            │
│  ┌────────────┐  ┌────────────┐  ┌────────────┐            │
│  │   Kernel   │  │ Validation │  │   Script   │            │
│  │ (Consensus │  │  (Chain    │  │ Interpreter│            │
│  │   Engine)  │  │   State)   │  │            │            │
│  └────────────┘  └────────────┘  └────────────┘            │
└─────────────────────────────────────────────────────────────┘
                            │
┌─────────────────────────────────────────────────────────────┐
│                  Foundation Layer                            │
│  ┌────────────┐  ┌────────────┐  ┌────────────┐            │
│  │   Crypto   │  │    Util    │  │  Common    │            │
│  │ (Hashing,  │  │ (Helpers,  │  │ (Shared    │            │
│  │   Signing) │  │   Logging) │  │   Code)    │            │
│  └────────────┘  └────────────┘  └────────────┘            │
└─────────────────────────────────────────────────────────────┘
```

---

## Directory Structure

### Root Level
```
/
├── cmake/               # CMake modules and build configuration
├── ci/                  # Continuous Integration scripts
├── contrib/             # Contributed utilities and tools
├── depends/             # Dependency management system
├── doc/                 # Technical documentation
├── docs/                # Essential project documentation
├── share/               # Platform-specific files and resources
├── src/                 # Core source code
└── test/                # Test suites
```

### Source Directory (`src/`)
```
src/
├── bench/               # Benchmarking tools
├── common/              # Shared high-level functionality
├── compat/              # Platform compatibility layer
├── consensus/           # Consensus-critical code
├── crc32c/              # CRC32C implementation
├── crypto/              # Cryptographic primitives
├── index/               # Blockchain indices
├── init/                # Initialization code
├── interfaces/          # Abstract interfaces between components
├── ipc/                 # Inter-process communication
├── kernel/              # Consensus kernel
├── leveldb/             # LevelDB database (embedded)
├── logging/             # Logging framework
├── minisketch/          # Set reconciliation (Erlay)
├── node/                # Full node functionality
├── policy/              # Transaction policy rules
├── primitives/          # Basic data structures (Block, Transaction)
├── qt/                  # Qt GUI implementation
├── rpc/                 # JSON-RPC server
├── script/              # Script interpreter and descriptors
├── secp256k1/           # Elliptic curve cryptography
├── support/             # Memory management and utilities
├── test/                # Unit tests
├── univalue/            # JSON parsing library
├── util/                # Low-level utilities
├── wallet/              # Wallet implementation
└── zmq/                 # ZeroMQ notification interface
```

---

## Core Components

### 1. Network Layer (`src/net*.cpp`, `src/net*.h`)

**Purpose**: Manages P2P network connections and protocol messages.

**Key Files**:
- `src/net.cpp` - Connection management, socket handling
- `src/net_processing.cpp` - Protocol message processing
- `src/netaddress.cpp` - Network address handling (IPv4/6, Tor, I2P)
- `src/netbase.cpp` - Low-level networking utilities
- `src/addrman.cpp` - Peer address database
- `src/banman.cpp` - Peer ban management

**Key Classes**:
- `CConnman` - Main connection manager
- `PeerManager` - Peer message handler
- `CNode` - Represents a peer connection
- `CAddress` - Network address representation
- `CAddrMan` - Address manager with anti-Sybil logic

**Functions**:
- `CConnman::ThreadSocketHandler()` - Socket I/O event loop
- `PeerManager::ProcessMessages()` - Message processing
- `CConnman::ThreadOpenConnections()` - Outbound connection management

### 2. Consensus Layer (`src/consensus/`, `src/kernel/`)

**Purpose**: Implements consensus rules and validation logic.

**Key Files**:
- `src/consensus/consensus.h` - Consensus parameters
- `src/consensus/tx_verify.cpp` - Transaction verification
- `src/consensus/merkle.cpp` - Merkle tree operations
- `src/kernel/chainparams.cpp` - Chain parameters
- `src/kernel/coinstats.cpp` - UTXO set statistics

**Key Classes**:
- `CConsensusParams` - Consensus parameters
- `CChainParams` - Network-specific parameters
- `TxValidationState` - Validation result state

### 3. Validation Engine (`src/validation.cpp`, `src/validation.h`)

**Purpose**: Core blockchain validation and state management.

**Key Classes**:
- `CChainState` - Manages blockchain state
- `CBlockIndex` - Block metadata in chain
- `CCoinsView` - UTXO set interface
- `CBlockTreeDB` - Block index database

**Key Functions**:
- `CChainState::ConnectBlock()` - Apply block to chain state
- `CChainState::DisconnectBlock()` - Revert block
- `CChainState::ActivateBestChain()` - Switch to best chain
- `CheckBlock()` - Block validation
- `CheckTransaction()` - Transaction validation

### 4. Transaction Mempool (`src/txmempool.cpp`, `src/txmempool.h`)

**Purpose**: Manages unconfirmed transactions.

**Key Classes**:
- `CTxMemPool` - Main mempool class
- `CTxMemPoolEntry` - Single transaction entry
- `TxMempoolInfo` - Transaction metadata

**Key Functions**:
- `CTxMemPool::addUnchecked()` - Add transaction
- `CTxMemPool::removeRecursive()` - Remove transaction and descendants
- `CTxMemPool::check()` - Consistency checks
- `CTxMemPool::estimateFee()` - Fee estimation

### 5. Wallet System (`src/wallet/`)

**Purpose**: Key management, coin selection, transaction creation.

**Key Files**:
- `src/wallet/wallet.cpp` - Main wallet logic
- `src/wallet/spend.cpp` - Transaction creation
- `src/wallet/coinselection.cpp` - Coin selection algorithms
- `src/wallet/walletdb.cpp` - Wallet database
- `src/wallet/scriptpubkeyman.cpp` - Key/script management

**Key Classes**:
- `CWallet` - Main wallet class
- `ScriptPubKeyMan` - Key and script manager
- `CWalletTx` - Wallet transaction
- `COutput` - Available coin for spending

**Key Functions**:
- `CWallet::CreateTransaction()` - Build transaction
- `CWallet::SignTransaction()` - Sign inputs
- `SelectCoins()` - Coin selection
- `CWallet::SyncMetaData()` - Metadata synchronization

### 6. Script System (`src/script/`)

**Purpose**: Script execution and validation.

**Key Files**:
- `src/script/interpreter.cpp` - Script VM
- `src/script/descriptor.cpp` - Output descriptors
- `src/script/sign.cpp` - Transaction signing
- `src/script/miniscript.cpp` - Miniscript support

**Key Classes**:
- `CScript` - Script container
- `ScriptExecutionData` - Execution context
- `Descriptor` - Output descriptor base

**Key Functions**:
- `EvalScript()` - Script execution
- `VerifyScript()` - Script verification
- `SignTransaction()` - Sign transaction inputs

### 7. RPC Interface (`src/rpc/`)

**Purpose**: JSON-RPC API for external applications.

**Key Files**:
- `src/rpc/server.cpp` - RPC server
- `src/rpc/blockchain.cpp` - Blockchain RPCs
- `src/rpc/mining.cpp` - Mining RPCs
- `src/rpc/net.cpp` - Network RPCs
- `src/rpc/rawtransaction.cpp` - Transaction RPCs

**Key Functions**:
- `JSONRPCExec()` - Execute RPC command
- `getblockchaininfo()` - Blockchain info
- `sendrawtransaction()` - Broadcast transaction
- `getpeerinfo()` - Peer information

### 8. GUI (`src/qt/`)

**Purpose**: Graphical user interface (Qt-based).

**Key Files**:
- `src/qt/sqoin.cpp` - Application entry point
- `src/qt/sqoingui.cpp` - Main window
- `src/qt/walletmodel.cpp` - Wallet model
- `src/qt/transactionview.cpp` - Transaction list

**Key Classes**:
- `SqoinGUI` - Main window
- `WalletModel` - Wallet interface
- `ClientModel` - Node interface
- `TransactionTableModel` - Transaction data

---

## Libraries

### Core Libraries

| Library | Purpose | Location |
|---------|---------|----------|
| **libsqoin_consensus** | Consensus validation | `src/consensus/` |
| **libsqoin_crypto** | Cryptographic primitives | `src/crypto/` |
| **libsqoin_kernel** | Consensus engine | `src/kernel/` |
| **libsqoin_common** | Shared high-level code | `src/` (various) |
| **libsqoin_util** | Low-level utilities | `src/util/` |
| **libsqoin_node** | Full node functionality | `src/node/` |
| **libsqoin_wallet** | Wallet implementation | `src/wallet/` |
| **libsqoinqt** | GUI implementation | `src/qt/` |
| **libsqoin_cli** | RPC client | `src/` |
| **libsqoin_zmq** | ZMQ notifications | `src/zmq/` |

### Embedded Libraries

| Library | Purpose | Location |
|---------|---------|----------|
| **LevelDB** | Key-value database | `src/leveldb/` |
| **secp256k1** | Elliptic curve crypto | `src/secp256k1/` |
| **univalue** | JSON parser | `src/univalue/` |
| **minisketch** | Set reconciliation | `src/minisketch/` |
| **CRC32C** | CRC32C checksums | `src/crc32c/` |

### Library Dependencies

```
sqoin → libsqoin_node
sqoind → libsqoin_node + libsqoin_wallet
sqoin-cli → libsqoin_cli
sqoin-qt → libsqoin_node + libsqoinqt + libsqoin_wallet

libsqoin_node → libsqoin_kernel + libsqoin_common
libsqoin_wallet → libsqoin_common
libsqoin_common → libsqoin_consensus + libsqoin_util
libsqoin_kernel → libsqoin_consensus + libsqoin_util
libsqoin_util → libsqoin_crypto
```

---

## Executables

### Main Executables

| Executable | Purpose | Source |
|------------|---------|--------|
| **sqoin** | Wrapper command | `src/sqoin.cpp` |
| **sqoind** | Daemon (full node) | `src/sqoind.cpp` |
| **sqoin-qt** | GUI client | `src/qt/sqoin.cpp` |
| **sqoin-cli** | RPC client | `src/sqoin-cli.cpp` |
| **sqoin-tx** | Transaction utility | `src/sqoin-tx.cpp` |
| **sqoin-wallet** | Wallet utility | `src/sqoin-wallet.cpp` |
| **sqoin-util** | General utility | `src/sqoin-util.cpp` |

### Test Executables

| Executable | Purpose |
|------------|---------|
| **test_sqoin** | Unit tests |
| **bench_sqoin** | Benchmarks |
| **sqoin-chainstate** | Chainstate utility (experimental) |

---

## Key Classes and Functions

### Blockchain Core

**File**: `src/chain.h`, `src/chain.cpp`

```cpp
class CBlockIndex {
    // Block metadata
    uint256 hashBlock;           // Block hash
    CBlockIndex* pprev;          // Previous block
    int nHeight;                 // Block height
    int64_t nTime;              // Block timestamp
    uint32_t nBits;             // Proof-of-work target
    
    // Functions
    CBlockHeader GetBlockHeader() const;
    uint256 GetBlockHash() const;
};

class CChain {
    std::vector<CBlockIndex*> vChain;  // Active chain
    
    CBlockIndex* Tip() const;          // Chain tip
    int Height() const;                // Chain height
    CBlockIndex* operator[](int nHeight) const;
};
```

### UTXO Management

**File**: `src/coins.h`, `src/coins.cpp`

```cpp
class Coin {
    CTxOut out;              // Transaction output
    uint32_t nHeight;        // Height at which tx was included
    bool fCoinBase;          // Is coinbase output
};

class CCoinsView {
    virtual bool GetCoin(const COutPoint &outpoint, Coin &coin) const;
    virtual bool HaveCoin(const COutPoint &outpoint) const;
    virtual bool BatchWrite(CCoinsMap &mapCoins, const uint256 &hashBlock);
};
```

### Transaction Primitives

**File**: `src/primitives/transaction.h`

```cpp
class COutPoint {
    uint256 hash;     // Transaction hash
    uint32_t n;       // Output index
};

class CTxIn {
    COutPoint prevout;       // Previous output
    CScript scriptSig;       // Signature script
    uint32_t nSequence;      // Sequence number
    CScriptWitness scriptWitness;  // Witness data
};

class CTxOut {
    CAmount nValue;          // Amount
    CScript scriptPubKey;    // Public key script
};

class CTransaction {
    int32_t nVersion;
    std::vector<CTxIn> vin;
    std::vector<CTxOut> vout;
    uint32_t nLockTime;
    
    uint256 GetHash() const;
    CAmount GetValueOut() const;
};
```

### Block Primitives

**File**: `src/primitives/block.h`

```cpp
class CBlockHeader {
    int32_t nVersion;
    uint256 hashPrevBlock;
    uint256 hashMerkleRoot;
    uint32_t nTime;
    uint32_t nBits;
    uint32_t nNonce;
    
    uint256 GetHash() const;
};

class CBlock : public CBlockHeader {
    std::vector<CTransactionRef> vtx;
};
```

### Network Protocol

**File**: `src/protocol.h`

```cpp
class CMessageHeader {
    uint8_t pchMessageStart[4];  // Network magic bytes
    char pchCommand[12];          // Command name
    uint32_t nMessageSize;        // Payload size
    uint32_t nChecksum;           // Checksum
};

class CAddress {
    uint64_t nServices;           // Node services
    CService ip;                  // IP address and port
    int64_t nTime;                // Last seen time
};
```

---

## Module Dependencies

### Dependency Graph

```
┌─────────────────┐
│    sqoin.cpp    │  (Main wrapper)
└────────┬────────┘
         │
    ┌────┴────┬──────────┐
    │         │          │
┌───▼───┐ ┌──▼──┐  ┌────▼────┐
│sqoind │ │sqoin│  │sqoin-cli│
│  .cpp │ │-qt  │  │  .cpp   │
└───┬───┘ └──┬──┘  └────┬────┘
    │        │           │
    │   ┌────┴───────────┘
    │   │
┌───▼───▼───────────────────┐
│   libsqoin_node           │
│   - net.cpp               │
│   - net_processing.cpp    │
│   - validation.cpp        │
│   - txmempool.cpp         │
└───────┬───────────────────┘
        │
    ┌───┴────┬────────────┐
    │        │            │
┌───▼────┐ ┌─▼──────┐ ┌──▼──────┐
│ kernel │ │ common │ │ wallet  │
└───┬────┘ └───┬────┘ └────┬────┘
    │          │            │
    └──────┬───┴────────────┘
           │
    ┌──────┴──────┐
    │             │
┌───▼──────┐  ┌──▼────┐
│consensus │  │ util  │
└───┬──────┘  └───┬───┘
    │             │
    └──────┬──────┘
           │
       ┌───▼───┐
       │ crypto│
       └───────┘
```

### Inter-Module Communication

**Interfaces Layer** (`src/interfaces/`):
- Provides abstract interfaces between major components
- Prevents tight coupling between node, wallet, and GUI
- Enables multiprocess architecture

**Key Interfaces**:
- `interfaces::Chain` - Blockchain access for wallet
- `interfaces::Node` - Node control for GUI
- `interfaces::Wallet` - Wallet access for GUI

---

## Build System

### CMake Configuration

**Main File**: `CMakeLists.txt`

**Key Variables**:
```cmake
CLIENT_NAME = "Sqoin Core"
CLIENT_VERSION = 30.99.0
PROJECT_NAME = SqoinCore
```

**Build Options**:
```cmake
BUILD_DAEMON          # Build sqoind
BUILD_GUI             # Build sqoin-qt
BUILD_CLI             # Build sqoin-cli
BUILD_WALLET          # Enable wallet
BUILD_TESTS           # Build tests
ENABLE_WALLET         # Wallet support
WITH_ZMQ              # ZMQ notifications
```

### Build Process

```bash
# Configure
cmake -B build -DCMAKE_BUILD_TYPE=Release

# Build
cmake --build build -j$(nproc)

# Test
ctest --test-dir build

# Install
cmake --install build
```

### Dependencies

**Required**:
- Boost >= 1.73.0
- libevent >= 2.1.8
- SQLite >= 3.32.0

**Optional**:
- Qt 6 >= 6.5.0 (for GUI)
- ZeroMQ >= 4.0.0
- MiniUPnPc >= 2.2.2

---

## Testing Framework

### Unit Tests (`src/test/`)

**Framework**: Boost.Test

**Key Test Files**:
- `transaction_tests.cpp` - Transaction validation
- `script_tests.cpp` - Script execution
- `validation_tests.cpp` - Blockchain validation
- `mempool_tests.cpp` - Mempool logic
- `wallet_tests.cpp` - Wallet operations

**Run Tests**:
```bash
./build/src/test/test_sqoin
```

### Functional Tests (`test/functional/`)

**Framework**: Python

**Key Tests**:
- `wallet_basic.py` - Wallet functionality
- `p2p_segwit.py` - SegWit protocol
- `rpc_blockchain.py` - RPC interface
- `feature_rbf.py` - Replace-by-fee

**Run Tests**:
```bash
./build/test/functional/test_runner.py
```

### Fuzz Tests (`src/test/fuzz/`)

**Purpose**: Discover crashes and undefined behavior

**Targets**:
- `transaction` - Transaction deserialization
- `block` - Block processing
- `script` - Script execution
- `process_message` - Network messages

---

## Data Flow

### Transaction Flow

```
User → Wallet
  ↓
Create Transaction (coinselection.cpp)
  ↓
Sign Transaction (spend.cpp)
  ↓
Send to Mempool (validation.cpp)
  ↓
Broadcast to Network (net_processing.cpp)
  ↓
Peers Validate & Relay
  ↓
Miner Includes in Block
  ↓
Block Propagation
  ↓
Validation & Chain Update (validation.cpp)
```

### Block Validation Flow

```
Receive Block (net_processing.cpp)
  ↓
CheckBlock() - Basic checks
  ↓
AcceptBlock() - Accept to block index
  ↓
ConnectBlock() - Apply to chain state
  ↓
Update UTXO set (coins.cpp)
  ↓
ActivateBestChain() - Switch if needed
  ↓
Notify Wallet & GUI
```

---

## Key Algorithms

### Coin Selection

**Location**: `src/wallet/coinselection.cpp`

**Algorithms**:
1. **Branch and Bound** - Exact match finding
2. **Knapsack Solver** - Approximate solution
3. **Lowest Larger** - Simple selection
4. **Coin Grinder** - Long-term optimization

### Fee Estimation

**Location**: `src/policy/fees/`

**Algorithm**: Statistical analysis of block confirmation times

### Proof-of-Work

**Location**: `src/pow.cpp`

**Algorithm**: SHA-256d double hash with difficulty adjustment

---

## Configuration Files

### sqoin.conf

**Location**: `~/.sqoin/sqoin.conf`

**Common Options**:
```ini
# Network
listen=1
port=8333

# RPC
rpcuser=username
rpcpassword=password
rpcport=8332

# Wallet
wallet=default
```

### Data Directory Structure

```
~/.sqoin/
├── sqoin.conf           # Configuration
├── blocks/              # Block data
├── chainstate/          # UTXO database
├── wallets/             # Wallet files
├── debug.log            # Log file
├── peers.dat            # Peer database
└── mempool.dat          # Mempool backup
```

---

## Network Protocol

### Message Types

| Message | Purpose |
|---------|---------|
| `version` | Handshake |
| `verack` | Handshake acknowledgment |
| `inv` | Inventory announcement |
| `getdata` | Request data |
| `tx` | Transaction data |
| `block` | Block data |
| `headers` | Block headers |
| `ping/pong` | Keep-alive |
| `addr` | Address sharing |

### Connection Lifecycle

```
1. Connect TCP socket
2. Send version message
3. Receive version message
4. Send verack
5. Receive verack
6. Connection established
7. Exchange data
8. Disconnect
```

---

## Security Considerations

### Cryptographic Functions

- **SHA-256** - Block and transaction hashing
- **RIPEMD-160** - Address generation
- **ECDSA** - Transaction signatures (secp256k1)
- **Schnorr** - Taproot signatures

### Validation Checks

1. **Script verification** - All inputs validated
2. **Signature verification** - ECDSA/Schnorr
3. **Double-spend prevention** - UTXO tracking
4. **Proof-of-work verification** - Block hashing
5. **Merkle tree verification** - Transaction inclusion

---

## Performance Optimizations

### Database

- **LevelDB** - Fast key-value storage
- **Caching** - In-memory UTXO cache
- **Batching** - Batch database writes

### Network

- **Compact Blocks** - Bandwidth reduction
- **Headers-first sync** - Faster initial sync
- **Erlay** - Efficient transaction relay

### Script Execution

- **Signature cache** - Caches verified signatures
- **Script cache** - Caches executed scripts

---

## Future Enhancements

### Planned Features
- Enhanced privacy features
- Improved scalability
- Advanced smart contract capabilities
- Layer 2 integration support
- Enhanced GUI features
- Mobile wallet support

### Development Roadmap
- Version 31.0: Stability improvements
- Version 32.0: Performance optimizations
- Version 33.0: New features rollout

---

## References

### Internal Documentation
- `/docs/CONTRIBUTING.md` - Contributing guidelines
- `/docs/INSTALL.md` - Installation instructions
- `/docs/SECURITY.md` - Security policies
- `/doc/` - Technical documentation

### External Resources
- Sqoin Website: https://sqoincore.org
- GitHub Repository: https://github.com/sqoin/sqoin
- Developer Forum: (TBD)

---

**Last Updated**: 2025-11-11  
**Version**: 1.0  
**Maintained by**: Sqoin Core Developers
