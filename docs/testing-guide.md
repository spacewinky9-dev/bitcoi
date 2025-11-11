# Sqoin Core - Complete Testing Guide

## Installation, Transaction Creation, Validation, and Spending

This guide provides step-by-step instructions for installing Sqoin Core in a VM, creating transactions, validating them, and spending coins - all based on the consensus logic documented in `consensus-logic.md`.

---

## Table of Contents

1. [VM Setup and Installation](#1-vm-setup-and-installation)
2. [Understanding the Build](#2-understanding-the-build)
3. [Starting Regtest Network](#3-starting-regtest-network)
4. [Mining Blocks](#4-mining-blocks)
5. [Creating Transactions](#5-creating-transactions)
6. [Transaction Validation](#6-transaction-validation)
7. [Spending Coins](#7-spending-coins)
8. [Understanding UTXO](#8-understanding-utxo)
9. [Advanced Testing](#9-advanced-testing)
10. [Troubleshooting](#10-troubleshooting)

---

## 1. VM Setup and Installation

### 1.1 System Requirements

**Minimum VM Configuration**:
- OS: Ubuntu 22.04 LTS or later
- RAM: 4 GB minimum (8 GB recommended)
- Disk: 20 GB free space
- CPU: 2 cores minimum

### 1.2 Install Dependencies

```bash
# Update system
sudo apt update && sudo apt upgrade -y

# Install build dependencies
sudo apt install -y \
    build-essential \
    cmake \
    git \
    libboost-all-dev \
    libevent-dev \
    libsqlite3-dev \
    pkg-config \
    python3

# Optional: Install Qt for GUI (optional)
sudo apt install -y \
    qtbase5-dev \
    qttools5-dev \
    qttools5-dev-tools
```

### 1.3 Clone and Build Sqoin Core

```bash
# Clone the repository
cd ~
git clone https://github.com/spacewinky9-dev/bitcoi.git sqoin
cd sqoin

# Create build directory
mkdir build
cd build

# Configure with CMake
cmake .. \
    -DCMAKE_BUILD_TYPE=Release \
    -DBUILD_GUI=OFF \
    -DBUILD_TESTS=ON

# Build (this will take 15-30 minutes)
make -j$(nproc)

# Install binaries
sudo make install

# Verify installation
sqoind --version
sqoin-cli --version
```

**Expected Output**:
```
Sqoin Core version v28.99.0-...
Copyright (C) 2009-present The Sqoin Core developers
```

### 1.4 Create Data Directory

```bash
# Create Sqoin data directory
mkdir -p ~/.sqoin

# Create configuration file
cat > ~/.sqoin/sqoin.conf << EOF
# Sqoin Core configuration for testing

# Run on regtest network (for testing)
regtest=1

# RPC settings
server=1
rpcuser=sqointest
rpcpassword=sqointest123
rpcallowip=127.0.0.1

# Enable transaction indexing (useful for testing)
txindex=1

# Logging (optional)
debug=mempool
debug=validation

# Wallet settings
wallet=testwallet
EOF
```

---

## 2. Understanding the Build

### 2.1 Executables Compiled

After building, you have these executables:

1. **sqoind** - The Sqoin daemon (server)
2. **sqoin-cli** - Command-line interface to interact with sqoind
3. **sqoin-tx** - Transaction manipulation tool
4. **sqoin-wallet** - Wallet management tool
5. **sqoin-util** - Utility functions
6. **sqoin** - GUI wallet (if Qt enabled)

### 2.2 Key Concepts from consensus-logic.md

**Understanding from the documentation**:

1. **UTXO Model**: 
   - Coins exist as Unspent Transaction Outputs
   - Each output can only be spent once
   - Spending = referencing previous output as input

2. **Transaction Structure**:
   ```
   Transaction {
       inputs: [
           {prevout: {txid, vout}, scriptSig, witness}
       ]
       outputs: [
           {value, scriptPubKey}
       ]
       locktime, version
   }
   ```

3. **Validation Rules** (from consensus-logic.md):
   - All inputs must reference existing UTXOs
   - Input values ≥ output values
   - Scripts must verify correctly
   - No double-spending

---

## 3. Starting Regtest Network

### 3.1 What is Regtest?

**Regtest** (Regression Test Mode):
- Private local blockchain for testing
- You can mine blocks instantly
- No network connections needed
- Perfect for learning and testing

### 3.2 Start Sqoin Daemon

```bash
# Start sqoind in background
sqoind -daemon -regtest

# Wait a few seconds for startup
sleep 3

# Check if running
sqoin-cli -regtest getblockchaininfo
```

**Expected Output**:
```json
{
  "chain": "regtest",
  "blocks": 0,
  "headers": 0,
  "bestblockhash": "0f9188f13cb7b2c71f2a335e3a4fc328bf5beb436012afca590b1a11466e2206",
  "difficulty": 4.656542373906925e-10,
  "verificationprogress": 1,
  "chainwork": "0000000000000000000000000000000000000000000000000000000000000002"
}
```

**Understanding the Output**:
- `chain`: "regtest" confirms we're on test network
- `blocks`: 0 means only genesis block exists
- `difficulty`: Very low (easy to mine)
- `bestblockhash`: Genesis block hash

---

## 4. Mining Blocks

### 4.1 Why Mine?

**From consensus-logic.md**:
- Coinbase outputs need 100 confirmations before spendable (COINBASE_MATURITY)
- Mining creates new coins as block subsidy
- Initial subsidy: 50 SQOIN per block

### 4.2 Create a Wallet

```bash
# Create new wallet
sqoin-cli -regtest createwallet "testwallet"

# Generate a receiving address
ADDRESS=$(sqoin-cli -regtest getnewaddress)
echo "Your address: $ADDRESS"
```

### 4.3 Mine Blocks

```bash
# Mine 101 blocks to our address
# (Need 101 to spend coinbase: 1 to create + 100 confirmations)
sqoin-cli -regtest generatetoaddress 101 $ADDRESS
```

**Understanding What Happened**:

From consensus-logic.md, each block:
1. Contains coinbase transaction (50 SQOIN reward)
2. Requires proof-of-work (easy in regtest)
3. Updates UTXO set
4. Increments chain height

**Verify**:
```bash
# Check balance (should show 50 × 101 = 5050 SQOIN)
sqoin-cli -regtest getbalance

# Check blockchain info
sqoin-cli -regtest getblockchaininfo

# List unspent outputs
sqoin-cli -regtest listunspent
```

**Expected Balance**: 5050.00000000 SQOIN

---

## 5. Creating Transactions

### 5.1 Understanding Transaction Creation

**Process** (based on consensus-logic.md):
```
1. Select UTXOs (coins to spend)
2. Create outputs (recipients)
3. Calculate fee
4. Sign inputs
5. Broadcast to network
```

### 5.2 Simple Transaction (Using sqoin-cli)

```bash
# Generate a new address to send to
RECIPIENT=$(sqoin-cli -regtest getnewaddress)
echo "Recipient address: $RECIPIENT"

# Send 10 SQOIN
TXID=$(sqoin-cli -regtest sendtoaddress $RECIPIENT 10)
echo "Transaction ID: $TXID"

# View transaction details
sqoin-cli -regtest gettransaction $TXID

# View raw transaction (hexadecimal)
RAW_TX=$(sqoin-cli -regtest getrawtransaction $TXID)
echo "Raw transaction: $RAW_TX"

# Decode transaction to see structure
sqoin-cli -regtest decoderawtransaction $RAW_TX
```

**Understanding the Decoded Transaction**:
```json
{
  "txid": "...",
  "version": 2,
  "locktime": 0,
  "vin": [
    {
      "txid": "previous_tx_id",
      "vout": 0,
      "scriptSig": {"asm": "...", "hex": "..."},
      "sequence": 4294967295
    }
  ],
  "vout": [
    {
      "value": 10.00000000,
      "n": 0,
      "scriptPubKey": {
        "asm": "OP_DUP OP_HASH160 ... OP_EQUALVERIFY OP_CHECKSIG",
        "type": "pubkeyhash",
        "address": "..."
      }
    },
    {
      "value": 39.99990000,  # Change output
      "n": 1,
      "scriptPubKey": {...}
    }
  ]
}
```

**Key Points** (from consensus-logic.md):
- `vin`: Inputs (spending previous outputs)
- `vout`: New outputs being created
- Change output created automatically
- Fee = input_total - output_total = 0.00010000 SQOIN

### 5.3 Manual Transaction Creation

**Advanced: Create transaction manually**

```bash
# Step 1: Get a UTXO to spend
UTXO=$(sqoin-cli -regtest listunspent | jq -r '.[0]')
UTXO_TXID=$(echo $UTXO | jq -r '.txid')
UTXO_VOUT=$(echo $UTXO | jq -r '.vout')
UTXO_AMOUNT=$(echo $UTXO | jq -r '.amount')

echo "Spending UTXO:"
echo "  TXID: $UTXO_TXID"
echo "  Vout: $UTXO_VOUT"
echo "  Amount: $UTXO_AMOUNT"

# Step 2: Create recipient address
RECIPIENT=$(sqoin-cli -regtest getnewaddress)
CHANGE_ADDR=$(sqoin-cli -regtest getnewaddress)

# Step 3: Calculate amounts
SEND_AMOUNT=5.0
FEE=0.0001
CHANGE=$(echo "$UTXO_AMOUNT - $SEND_AMOUNT - $FEE" | bc)

# Step 4: Create raw transaction
RAW_TX=$(sqoin-cli -regtest createrawtransaction \
  "[{\"txid\":\"$UTXO_TXID\",\"vout\":$UTXO_VOUT}]" \
  "{\"$RECIPIENT\":$SEND_AMOUNT,\"$CHANGE_ADDR\":$CHANGE}")

echo "Raw unsigned transaction: $RAW_TX"

# Step 5: Sign transaction
SIGNED_TX=$(sqoin-cli -regtest signrawtransactionwithwallet $RAW_TX | jq -r '.hex')

echo "Signed transaction: $SIGNED_TX"

# Step 6: Send transaction
TXID=$(sqoin-cli -regtest sendrawtransaction $SIGNED_TX)
echo "Transaction sent! TXID: $TXID"
```

---

## 6. Transaction Validation

### 6.1 Understanding Validation (from consensus-logic.md)

**Validation Stages**:
1. **CheckTransaction()**: Basic checks
2. **AcceptToMemoryPool()**: Mempool admission
3. **ConnectBlock()**: Apply to chain state (when mined)

### 6.2 Check Transaction in Mempool

```bash
# View transaction in mempool
sqoin-cli -regtest getmempoolentry $TXID

# View all mempool transactions
sqoin-cli -regtest getrawmempool true
```

**Understanding Mempool Entry**:
```json
{
  "vsize": 191,           # Virtual size (for fee calculation)
  "fee": 0.00010000,      # Fee paid
  "modifiedfee": 0.00010000,
  "time": 1234567890,     # Entry time
  "height": 101,          # Block height when entered
  "depends": [],          # Parent transactions
  "spentby": [],          # Child transactions
  "bip125-replaceable": false
}
```

### 6.3 Validation Tests

**Test 1: Invalid Transaction (Negative Output)**
```bash
# This will fail validation
sqoin-cli -regtest createrawtransaction \
  "[{\"txid\":\"$UTXO_TXID\",\"vout\":$UTXO_VOUT}]" \
  "{\"$RECIPIENT\":-1.0}"

# Error: Amount out of range
```

**Test 2: Double Spend**
```bash
# Try to spend same UTXO twice (will fail)
RAW_TX1=$(sqoin-cli -regtest createrawtransaction \
  "[{\"txid\":\"$UTXO_TXID\",\"vout\":$UTXO_VOUT}]" \
  "{\"$RECIPIENT\":1.0}")

SIGNED_TX1=$(sqoin-cli -regtest signrawtransactionwithwallet $RAW_TX1 | jq -r '.hex')
TXID1=$(sqoin-cli -regtest sendrawtransaction $SIGNED_TX1)

# Try to spend same UTXO again
RAW_TX2=$(sqoin-cli -regtest createrawtransaction \
  "[{\"txid\":\"$UTXO_TXID\",\"vout\":$UTXO_VOUT}]" \
  "{\"$RECIPIENT\":2.0}")

SIGNED_TX2=$(sqoin-cli -regtest signrawtransactionwithwallet $RAW_TX2 | jq -r '.hex')
sqoin-cli -regtest sendrawtransaction $SIGNED_TX2

# Error: bad-txns-inputs-missingorspent
```

**Test 3: Insufficient Funds**
```bash
# Try to spend more than input value (will fail)
sqoin-cli -regtest createrawtransaction \
  "[{\"txid\":\"$UTXO_TXID\",\"vout\":$UTXO_VOUT}]" \
  "{\"$RECIPIENT\":100.0}"

# After signing and sending:
# Error: bad-txns-in-belowout
```

---

## 7. Spending Coins

### 7.1 Confirm Transaction

```bash
# Mine a block to confirm the transaction
sqoin-cli -regtest generatetoaddress 1 $ADDRESS

# Check confirmations
sqoin-cli -regtest gettransaction $TXID | jq '.confirmations'

# Output: 1
```

**Understanding Confirmations** (from consensus-logic.md):
- 0 confirmations: In mempool, not in block
- 1 confirmation: In latest block
- 6 confirmations: Standard for "safe" (industry practice)
- 100 confirmations: Required for coinbase (COINBASE_MATURITY)

### 7.2 Spend the Received Coins

```bash
# List UTXOs for recipient address
sqoin-cli -regtest listunspent 1 9999999 "[\"$RECIPIENT\"]"

# Create new transaction spending the received coins
NEW_RECIPIENT=$(sqoin-cli -regtest getnewaddress)

sqoin-cli -regtest sendtoaddress $NEW_RECIPIENT 5.0
```

### 7.3 UTXO Lifecycle

**Tracking UTXO State**:

```bash
# Before spending: UTXO exists
sqoin-cli -regtest gettxout $TXID 0

# Output shows UTXO details

# After spending: UTXO is spent
sqoin-cli -regtest gettxout $TXID 0

# Output: null (UTXO spent)
```

---

## 8. Understanding UTXO

### 8.1 UTXO Model Explained

**From consensus-logic.md**:

```
UTXO Set = All unspent transaction outputs
         = All spendable coins in the system

Each UTXO contains:
- value: Amount in satoshis
- scriptPubKey: Spending condition
- height: Block height when created
- coinbase flag: Is it from mining?
```

### 8.2 Query UTXO Set

```bash
# Get total UTXO set statistics
sqoin-cli -regtest gettxoutsetinfo

# List all UTXOs for your wallet
sqoin-cli -regtest listunspent

# Get specific UTXO details
sqoin-cli -regtest gettxout $TXID $VOUT
```

**Understanding gettxoutsetinfo**:
```json
{
  "height": 102,
  "bestblock": "...",
  "transactions": 102,
  "txouts": 150,
  "total_amount": 5100.00000000,
  "total_unspendable_amount": 0,
  "block_info": {...}
}
```

- `txouts`: Total number of UTXOs
- `total_amount`: Total spendable coins

### 8.3 UTXO Creation and Destruction

**Watch UTXO changes**:

```bash
# Count UTXOs before
BEFORE=$(sqoin-cli -regtest gettxoutsetinfo | jq '.txouts')

# Create transaction (2 outputs: recipient + change)
sqoin-cli -regtest sendtoaddress $RECIPIENT 1.0

# Mine block
sqoin-cli -regtest generatetoaddress 1 $ADDRESS

# Count UTXOs after
AFTER=$(sqoin-cli -regtest gettxoutsetinfo | jq '.txouts')

echo "UTXOs before: $BEFORE"
echo "UTXOs after: $AFTER"
echo "Change: $(($AFTER - $BEFORE))"

# Typically: +2 new outputs, -1 spent input, +1 coinbase = net +2
```

---

## 9. Advanced Testing

### 9.1 Test Time Locks (nLockTime)

**From consensus-logic.md**: Transactions can be time-locked

```bash
# Get current block height
HEIGHT=$(sqoin-cli -regtest getblockcount)

# Create transaction locked until future block
FUTURE_HEIGHT=$((HEIGHT + 10))

# Get UTXO
UTXO=$(sqoin-cli -regtest listunspent | jq -r '.[0]')
UTXO_TXID=$(echo $UTXO | jq -r '.txid')
UTXO_VOUT=$(echo $UTXO | jq -r '.vout')

# Create raw transaction with locktime
RAW_TX=$(sqoin-cli -regtest createrawtransaction \
  "[{\"txid\":\"$UTXO_TXID\",\"vout\":$UTXO_VOUT,\"sequence\":4294967294}]" \
  "{\"$RECIPIENT\":1.0}" \
  $FUTURE_HEIGHT)

# Sign transaction
SIGNED_TX=$(sqoin-cli -regtest signrawtransactionwithwallet $RAW_TX | jq -r '.hex')

# Try to send (will fail - locktime not reached)
sqoin-cli -regtest sendrawtransaction $SIGNED_TX
# Error: non-final

# Mine blocks to reach locktime
sqoin-cli -regtest generatetoaddress 10 $ADDRESS

# Now it works
sqoin-cli -regtest sendrawtransaction $SIGNED_TX
```

### 9.2 Test Replace-By-Fee (RBF)

**From consensus-logic.md**: RBF allows replacing unconfirmed transactions

```bash
# Create transaction with RBF enabled (sequence < 0xfffffffe)
UTXO=$(sqoin-cli -regtest listunspent | jq -r '.[0]')
UTXO_TXID=$(echo $UTXO | jq -r '.txid')
UTXO_VOUT=$(echo $UTXO | jq -r '.vout')

# Original transaction (low fee)
RAW_TX=$(sqoin-cli -regtest createrawtransaction \
  "[{\"txid\":\"$UTXO_TXID\",\"vout\":$UTXO_VOUT,\"sequence\":4294967293}]" \
  "{\"$RECIPIENT\":1.0}")

SIGNED_TX=$(sqoin-cli -regtest signrawtransactionwithwallet $RAW_TX | jq -r '.hex')
TXID=$(sqoin-cli -regtest sendrawtransaction $SIGNED_TX)

echo "Original TXID: $TXID"

# Replacement transaction (higher fee)
RAW_TX2=$(sqoin-cli -regtest createrawtransaction \
  "[{\"txid\":\"$UTXO_TXID\",\"vout\":$UTXO_VOUT,\"sequence\":4294967293}]" \
  "{\"$RECIPIENT\":0.99}")  # Less to recipient = higher fee

SIGNED_TX2=$(sqoin-cli -regtest signrawtransactionwithwallet $RAW_TX2 | jq -r '.hex')
TXID2=$(sqoin-cli -regtest sendrawtransaction $SIGNED_TX2)

echo "Replacement TXID: $TXID2"

# Check mempool - should only see replacement
sqoin-cli -regtest getrawmempool
```

### 9.3 Test Script Types

**P2PKH (Pay-to-PubKey-Hash)** - Default

```bash
# Generate legacy address
ADDR=$(sqoin-cli -regtest getnewaddress "" "legacy")
echo "P2PKH address: $ADDR"

# Send to it
sqoin-cli -regtest sendtoaddress $ADDR 1.0

# Decode the transaction to see scriptPubKey
TXID=$(sqoin-cli -regtest sendtoaddress $ADDR 1.0)
RAW=$(sqoin-cli -regtest getrawtransaction $TXID)
sqoin-cli -regtest decoderawtransaction $RAW | jq '.vout[0].scriptPubKey'

# Output shows: OP_DUP OP_HASH160 <pubkeyhash> OP_EQUALVERIFY OP_CHECKSIG
```

**P2SH (Pay-to-Script-Hash)**

```bash
# Generate P2SH address
ADDR=$(sqoin-cli -regtest getnewaddress "" "p2sh-segwit")
echo "P2SH address: $ADDR"

# Send to it
TXID=$(sqoin-cli -regtest sendtoaddress $ADDR 1.0)
RAW=$(sqoin-cli -regtest getrawtransaction $TXID)
sqoin-cli -regtest decoderawtransaction $RAW | jq '.vout[0].scriptPubKey'

# Output shows: OP_HASH160 <scripthash> OP_EQUAL
```

**P2WPKH (Pay-to-Witness-PubKey-Hash)** - Native SegWit

```bash
# Generate SegWit address
ADDR=$(sqoin-cli -regtest getnewaddress "" "bech32")
echo "P2WPKH address: $ADDR"

# Send to it
TXID=$(sqoin-cli -regtest sendtoaddress $ADDR 1.0)
RAW=$(sqoin-cli -regtest getrawtransaction $TXID)
sqoin-cli -regtest decoderawtransaction $RAW | jq '.vout[0].scriptPubKey'

# Output shows: OP_0 <20-byte-hash>
```

---

## 10. Troubleshooting

### 10.1 Common Issues

**Issue 1: "Cannot connect to daemon"**
```bash
# Check if sqoind is running
ps aux | grep sqoind

# Check RPC settings
cat ~/.sqoin/sqoin.conf

# Try connecting with explicit credentials
sqoin-cli -regtest -rpcuser=sqointest -rpcpassword=sqointest123 getblockcount
```

**Issue 2: "Insufficient funds"**
```bash
# Check actual balance
sqoin-cli -regtest getbalance

# List available UTXOs
sqoin-cli -regtest listunspent

# Check if coinbase matured (need 100 confirmations)
sqoin-cli -regtest getblockcount
```

**Issue 3: "Transaction too large"**
```bash
# Check transaction size
RAW=$(sqoin-cli -regtest getrawtransaction $TXID)
SIZE=$(echo -n $RAW | wc -c)
echo "Transaction size: $((SIZE / 2)) bytes"

# From consensus-logic.md: MAX_BLOCK_WEIGHT = 4,000,000
# Single tx must be < 4MB
```

### 10.2 Debug Commands

```bash
# View debug log
tail -f ~/.sqoin/regtest/debug.log

# Check mempool details
sqoin-cli -regtest getmempoolinfo

# Get network info
sqoin-cli -regtest getnetworkinfo

# Verify blockchain
sqoin-cli -regtest verifychain 2 100
```

### 10.3 Reset and Start Fresh

```bash
# Stop daemon
sqoin-cli -regtest stop

# Wait for clean shutdown
sleep 5

# Remove regtest data (keeps wallet backups)
rm -rf ~/.sqoin/regtest/blocks
rm -rf ~/.sqoin/regtest/chainstate
rm -rf ~/.sqoin/regtest/mempool.dat

# Restart
sqoind -daemon -regtest
```

---

## 11. Testing Checklist

### 11.1 Basic Tests ✓

- [ ] Install and build Sqoin Core
- [ ] Start regtest network
- [ ] Mine 101 blocks
- [ ] Check balance
- [ ] Create simple transaction
- [ ] Decode and understand transaction structure
- [ ] Confirm transaction by mining block
- [ ] Spend confirmed coins

### 11.2 Advanced Tests ✓

- [ ] Create manual transaction
- [ ] Test invalid transactions (negative amounts, double-spend)
- [ ] Test time-locked transactions
- [ ] Test RBF (Replace-By-Fee)
- [ ] Test different address types (P2PKH, P2SH, P2WPKH)
- [ ] Monitor UTXO set changes
- [ ] Verify transaction in mempool

### 11.3 Validation Understanding ✓

- [ ] Understand CheckTransaction() checks
- [ ] Understand UTXO verification
- [ ] Understand script execution
- [ ] Understand coinbase maturity (100 blocks)
- [ ] Understand confirmation depths

---

## 12. Summary

### What You've Learned

**From consensus-logic.md Implementation**:

1. **UTXO Model**: Coins are outputs, spent by referencing as inputs
2. **Transaction Validation**: 9 checks ensure no double-spend, proper values
3. **Block Rewards**: 50 SQOIN per block, halving every 210,000 blocks
4. **Coinbase Maturity**: Must wait 100 confirmations before spending
5. **Script Types**: P2PKH, P2SH, P2WPKH all validated differently
6. **Time Locks**: Transactions can be locked by height or time
7. **Mempool**: Transactions wait here before block inclusion

### Next Steps

For production deployment:
1. Review `docs/changes.md` for network customization
2. Create new genesis block
3. Change magic bytes and ports
4. Setup DNS seeds
5. Test on private testnet before mainnet

### Key RPC Commands Reference

```bash
# Blockchain
getblockchaininfo    # Chain status
getblockcount        # Current height
getblock <hash>      # Block details
getblockhash <height> # Get block hash

# Wallet
getnewaddress        # Generate address
getbalance           # Check balance
sendtoaddress        # Send coins
listtransactions     # List transactions
listunspent          # List UTXOs

# Transactions
gettransaction <txid>      # Get tx details
getrawtransaction <txid>   # Get raw tx
decoderawtransaction <hex> # Decode tx
createrawtransaction       # Create tx
signrawtransactionwithwallet # Sign tx
sendrawtransaction         # Broadcast tx

# Mining (regtest only)
generatetoaddress <n> <addr> # Mine n blocks

# Mempool
getmempoolinfo       # Mempool stats
getrawmempool        # List mempool txs
getmempoolentry      # Mempool tx details

# UTXO
gettxout <txid> <n>  # Check if UTXO exists
gettxoutsetinfo      # UTXO set statistics

# Network
getnetworkinfo       # Network status
getpeerinfo          # Connected peers
```

---

**Documentation References**:
- `docs/consensus-logic.md` - Complete consensus implementation
- `docs/current-strategy.md` - Current implementation details
- `docs/changes.md` - How to modify parameters
- `docs/codebase.md` - Architecture overview

**For Support**: Check debug.log file for detailed error messages

---

*This guide is for testing and educational purposes only. For production deployment, conduct thorough security audits and testing.*
