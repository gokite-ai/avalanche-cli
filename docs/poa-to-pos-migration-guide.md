# PoA to PoS Migration Guide (KiteStakingManager)

This guide describes the complete process for migrating an Avalanche L1 blockchain from Proof of Authority (PoA) to Proof of Stake (PoS) mode using a custom `KiteStakingManager`.

## Table of Contents

- [Overview](#overview)
- [Configuration](#configuration)
- [Prerequisites](#prerequisites)
- [Preparation: Setup Validator Multi-Sigs](#preparation-setup-validator-multi-sigs)
- [Migration Steps](#migration-steps)
  - [Step 1: Deploy and Initialize KiteStakingManager](#step-1-deploy-and-initialize-kitestakingmanager)
  - [Step 2: Transfer ValidatorManager Ownership](#step-2-transfer-validatormanager-ownership)
  - [Step 3: Daily Validator Migration](#step-3-daily-validator-migration-remove-poa--add-pos)
- [Multi-Sig Workflow](#multi-sig-workflow-summary)
- [Permission Summary](#permission-summary)
- [Troubleshooting](#troubleshooting)

---

## Overview

The migration from PoA to PoS involves:

1. **Prepare**: Setup 11 validator multi-sigs (3/5 each, 55 unique signers)
2. **Deploy & Initialize**: Deploy and initialize custom `KiteStakingManager` contract
3. **Transfer**: Transfer ValidatorManager ownership (via existing multi-sig)
4. **Migrate**: Remove and re-add validators **one by one** (due to churn limit)

### ⚠️ Important: Churn Limit

Due to the current churn settings, **only 1 validator can be removed and 1 can be added per day**.

**Migration Strategy**:
- Each day: Remove 1 PoA validator → Add it back as PoS validator
- Total migration time: **11 days** (one validator per day)
- This ensures network stability

---

## Configuration

### Staking Parameters

| Parameter | Value | Description |
|-----------|-------|-------------|
| `minStakeAmount` | 1,000,000 KITE | Minimum stake per validator |
| `maxStakeAmount` | 10,000,000 KITE | Maximum stake per validator |
| `minStakeDuration` | 2 weeks (1,209,600 seconds) | Minimum lock period |

### Validator Multi-Sig Setup

| Item | Count | Details |
|------|-------|---------|
| Total Validators | 11 | |
| Multi-sig Type | 3/5 | 3 signatures required out of 5 |
| Signers per Validator | 5 | Unique signers |
| Total Unique Signers | 55 | 11 × 5 = 55 |

### Token Minting

| Item | Value |
|------|-------|
| NativeMinter Admin | `0x7700D95B5250045400FE531f8886D9B02eA50801` (multi-sig) |
| Total KITE to Mint | 11,000,000 KITE (11M) |
| Distribution | 1M KITE per validator |

**Flow**:
```
NativeMinter.mintNativeCoin(0x7700..., 11M KITE)
     ↓
0x7700... (admin multi-sig) holds 11M KITE
     ↓
Transfer 1M KITE to each validator's Safe during registration
```

---

## Prerequisites

- Avalanche CLI installed and configured
- Access to the **existing ValidatorManager owner multi-sig** (for ownership transfer)
- Access to the **NativeMinter admin multi-sig** (`0x7700D95B5250045400FE531f8886D9B02eA50801`)
- 55 unique EOA addresses for validator multi-sigs
- Multi-sig wallet factory (e.g., Safe) deployed on L1
- Node BLS keys for all 11 validators
- **P-Chain addresses** for each validator (see below)
- Sufficient funds:
  - Contract deployment gas fees
  - P-Chain validator balance (AVAX)
  - 11M KITE minted for staking (1M per validator)

### P-Chain Addresses for Validators

Each validator needs **2 P-Chain addresses** for P-Chain-level controls:

| Address | Purpose | Description |
|---------|---------|-------------|
| **Remaining Balance Owner** | Receive leftover AVAX | When validator exits, remaining P-Chain balance (from `--balance`) is refunded to this address |
| **Disable Owner** | Emergency disable control | Can disable validator via P-Chain `DisableL1ValidatorTx` (recoverable via `IncreaseL1ValidatorBalanceTx`) |

> 💡 **Tip**: Since these are P-Chain addresses with limited, recoverable permissions, **single-sig (EOA) is acceptable**. Multi-sig is not required for these addresses.

**Prepare for each validator**:
```
Validator 1:  remaining_balance_owner_1 (P-...), disable_owner_1 (P-...)
Validator 2:  remaining_balance_owner_2 (P-...), disable_owner_2 (P-...)
...
Validator 11: remaining_balance_owner_11 (P-...), disable_owner_11 (P-...)
```

Use different P chain address for each validator.

---

## Preparation: Setup Validator Multi-Sigs

Before migration, create 11 Safe multi-sig wallets (3/5) for each validator.

### Step 0.1: Prepare 55 Unique Signers

Generate or collect 55 unique EOA addresses:

```
Validator 1:  signer_1_1, signer_1_2, signer_1_3, signer_1_4, signer_1_5
Validator 2:  signer_2_1, signer_2_2, signer_2_3, signer_2_4, signer_2_5
...
Validator 11: signer_11_1, signer_11_2, signer_11_3, signer_11_4, signer_11_5
```

### Step 0.2: Deploy Safe Multi-Sigs

For each validator, deploy a Safe wallet:

```bash
# Using Safe SDK or Safe UI
# Threshold: 3
# Owners: 5 addresses

# Example using cast to interact with SafeProxyFactory
# (Replace with actual Safe deployment on your L1)
```

**Record all 11 Safe addresses**:

| Validator | Node ID | Safe Address (Owner) |
|-----------|---------|---------------------|
| 1 | NodeID-... | 0xSafe1... |
| 2 | NodeID-... | 0xSafe2... |
| ... | ... | ... |
| 11 | NodeID-... | 0xSafe11... |

### Step 0.3: Fund Multi-Sigs with Gas

Each Safe needs gas funds for signing transactions:

```bash
# Transfer gas funds to each Safe
# Recommended: 0.1 - 1 native token per Safe for gas
```

### Step 0.4: Mint 11M KITE via NativeMinter

Mint staking tokens to the NativeMinter admin multi-sig.

**NativeMinter Precompile Address**: `0x0200000000000000000000000000000000000001`

**Admin Multi-Sig**: `0x7700D95B5250045400FE531f8886D9B02eA50801`

#### Generate Mint Calldata

```bash
# Using cast
cast calldata "mintNativeCoin(address,uint256)" \
  0x7700D95B5250045400FE531f8886D9B02eA50801 \
  11000000000000000000000000

# Result calldata: 0x...
```

#### Create Multi-Sig Transaction

1. Go to Safe UI for `0x7700D95B5250045400FE531f8886D9B02eA50801`
2. Create new transaction:
   - **To**: `0x0200000000000000000000000000000000000001` (NativeMinter)
   - **Value**: 0
   - **Data**: Calldata from above
3. Sign and execute

#### Verify Mint

```bash
# Check balance
cast balance 0x7700D95B5250045400FE531f8886D9B02eA50801 --rpc-url <L1_RPC>
# Expected: 11000000000000000000000000 (11M KITE)
```

---

## Migration Steps

### Step 1: Deploy and Initialize KiteStakingManager

Deploy and initialize your custom `KiteStakingManager` contract (done outside CLI).

```solidity
// KiteStakingManager initialization parameters
struct InitParams {
    uint256 minStakeAmount;      // 1,000,000 * 1e18 KITE
    uint256 maxStakeAmount;      // 10,000,000 * 1e18 KITE  
    uint64 minStakeDuration;     // 1,209,600 seconds (2 weeks)
    address validatorManager;    // existing ValidatorManager proxy address
    // ... other params
}

// Deploy and initialize in one transaction (or two if needed)
KiteStakingManager stakingManager = new KiteStakingManager();
stakingManager.initialize(initParams);
```

**Record the deployed address**: `KITE_STAKING_MANAGER_ADDRESS`

---

### Step 2: Transfer ValidatorManager Ownership

Transfer ownership of the ValidatorManager to KiteStakingManager.

> ⚠️ **This requires the existing ValidatorManager owner multi-sig to sign**

#### Generate Transfer Calldata

```bash
# Generate calldata for ownership transfer
cast calldata "transferOwnership(address)" <KITE_STAKING_MANAGER_ADDRESS>

# Example output: 0xf2fde38b000000000000000000000000<KITE_STAKING_MANAGER_ADDRESS>
```

#### Execute via Multi-Sig

1. Go to Safe UI for the **existing ValidatorManager owner multi-sig**
2. Create new transaction:
   - **To**: ValidatorManager proxy address
   - **Value**: 0
   - **Data**: Calldata from above
3. Collect required signatures and execute

---

### Step 3: Daily Validator Migration (Remove PoA → Add PoS)

Due to the **churn limit** (1 validator change per day), migration is done **one validator at a time**.

```
Day 1:  Remove Validator 1 (PoA) → Add Validator 1 (PoS)
Day 2:  Remove Validator 2 (PoA) → Add Validator 2 (PoS)
...
Day 11: Remove Validator 11 (PoA) → Add Validator 11 (PoS)
```

---

### Daily Migration Process (Repeat for Each Validator)

#### 4.1 Remove PoA Validator (Permissionless)

PoA validator removal is **permissionless** - no multi-sig required

```bash
# Remove the PoA validator
avalanche blockchain removeValidator <blockchainName> \
  --mainnet \
  --node-id <NODE_ID>
```

The CLI will:
1. Call `initiateValidatorRemoval` (permissionless for PoA)
2. Update weight on P-Chain via `SetL1ValidatorWeightTx`
3. Call `completeValidatorRemoval` (also permissionless)

#### 4.2 Transfer 1M KITE to Validator Safe

Before staking, transfer KITE from admin multi-sig to this validator's Safe.

**From**: `0x7700D95B5250045400FE531f8886D9B02eA50801` (NativeMinter Admin)  
**To**: Validator Safe address  
**Amount**: 1,000,000 KITE

In Safe UI for admin multi-sig:
- **To**: `<VALIDATOR_SAFE_ADDRESS>`
- **Value**: `1000000000000000000000000` (1M KITE in wei)
- **Data**: `0x` (empty for native transfer)

> 💡 Transfer tokens **one validator at a time**, before each validator's PoS registration.

#### 4.3 Add PoS Validator

Re-add the same validator with PoS staking. The validator's 3/5 multi-sig will be the staker/owner.

#### Verify Transfer

```bash
# Check validator Safe balance
cast balance <VALIDATOR_SAFE_ADDRESS> --rpc-url <L1_RPC>
# Expected: 1000000000000000000000000 (1M KITE)
```

#### 4.4 Generate Unsigned Initiate Transaction

Generate calldata for the validator's multi-sig to execute:

```bash
avalanche blockchain addValidator kite \
  --mainnet \
  --key pchain-sender \
  --node-id <NODE_ID> \
  --bls-public-key <BLS_PUBLIC_KEY> \
  --bls-proof-of-possession <BLS_POP> \
  --weight <STAKE_WEIGHT> \
  --balance 0.5 \
  --staking-period 2w \
  --delegation-fee 300 \
  --rewards-recipient <SAFE_ADDRESS> \
  --remaining-balance-owner <P_CHAIN_ADDRESS> \
  --disable-owner <P_CHAIN_ADDRESS> \
  --external-evm-signature \
  --validator-owner <SAFE_ADDRESS>
```

**CLI Output Example**:
```
External signing mode enabled for PoS validator owner
Validator owner (staker): 0xSafe1...
Required staking funds: 1,000,000 KITE

Tx Dump For Initializing Validator Registration:
0x02f902...

Calldata Dump:
0xad93936d...
```

#### 4.5 Execute via Validator Multi-Sig (3/5)

In Safe UI for the validator's Safe:

1. **Create Transaction**:
   - **To**: KiteStakingManager address
   - **Value**: `1000000000000000000000000` (1M KITE in wei)
   - **Data**: Calldata from CLI output

2. **Collect 3 of 5 Signatures**:
   - Signer 1 signs ✓
   - Signer 2 signs ✓
   - Signer 3 signs ✓

3. **Execute Transaction**
4. **Record Transaction Hash**: `0xInitiateTxHash...`

#### 4.6 Complete Registration (Permissionless)

```bash
avalanche blockchain addValidator kite \
  --mainnet \
  --key pchain-sender \
  --node-id <NODE_ID> \
  --bls-public-key <BLS_PUBLIC_KEY> \
  --bls-proof-of-possession <BLS_POP> \
  --weight <STAKE_WEIGHT> \
  --balance 0.5 \
  --staking-period 2w \
  --delegation-fee 300 \
  --rewards-recipient <SAFE_ADDRESS> \
  --remaining-balance-owner <P_CHAIN_ADDRESS> \
  --disable-owner <P_CHAIN_ADDRESS> \
  --validator-owner <SAFE_ADDRESS> \
  --initiate-tx-hash <INITIATE_TX_HASH>
```

> **No multi-sig needed for complete!** `completeValidatorRegistration` is permissionless.

---

### Validator Data Reference

Prepare this data for all 11 validators before starting migration:

| # | Node ID | BLS Public Key | BLS PoP | Safe Address | P-Chain Addr |
|---|---------|----------------|---------|--------------|--------------|
| 1 | NodeID-v1... | 0x... | 0x... | 0xSafe1... | P-... |
| 2 | NodeID-v2... | 0x... | 0x... | 0xSafe2... | P-... |
| ... | ... | ... | ... | ... | ... |
| 11 | NodeID-v11... | 0x... | 0x... | 0xSafe11... | P-... |

---

### Daily Migration Checklist

For each day's validator migration:

```
Day N: Migrate Validator N

[ ] 1. Remove PoA validator N
    avalanche blockchain removeValidator kite --mainnet --node-id <NODE_ID_N>
    
[ ] 2. Transfer 1M KITE to Validator N's Safe
    - Create tx in admin multi-sig (0x7700...)
    - To: <SAFE_N_ADDRESS>
    - Value: 1000000000000000000000000
    - Collect signatures & execute

[ ] 3. Generate initiate calldata
    avalanche blockchain addValidator kite --mainnet ... --external-evm-signature
    
[ ] 4. Execute via Validator N's Safe (3/5 multi-sig)
    - Create tx with calldata
    - Collect 3 signatures & execute
    - Record tx hash

[ ] 5. Complete registration
    avalanche blockchain addValidator kite --mainnet ... --initiate-tx-hash <TX_HASH>
    
[ ] 6. Verify validator is active
    avalanche blockchain list-validators kite --mainnet
```

**Repeat for Day 1 through Day 11**

---

### Example: Day 1 Migration Script

```bash
#!/bin/bash
# daily_migration.sh - Run this script for each validator

BLOCKCHAIN_NAME="kite"
STAKING_PERIOD="2w"  # 2 weeks minimum
DELEGATION_FEE=300
P_CHAIN_KEY="pchain-sender"

# Today's validator info (update daily)
NODE_ID="NodeID-v1..."
BLS_PUB="0x..."
BLS_POP="0x..."
SAFE_ADDR="0xSafe1..."
P_CHAIN_ADDR="P-avax1..."
WEIGHT=1000000

echo "=========================================="
echo "Day 1: Migrating validator $NODE_ID"
echo "=========================================="

# Step 1: Remove PoA validator
echo "Step 1: Removing PoA validator..."
avalanche blockchain removeValidator $BLOCKCHAIN_NAME \
  --mainnet \
  --node-id $NODE_ID

echo "✅ PoA validator removed"
echo ""

# Step 2: Transfer KITE (manual - do in Safe UI)
echo "Step 2: Transfer 1M KITE to $SAFE_ADDR"
echo "  - Create tx in admin multi-sig"
echo "  - To: $SAFE_ADDR"
echo "  - Value: 1000000000000000000000000 (1M KITE)"
echo ""
read -p "Press Enter after KITE transfer is complete..."
echo ""

# Step 3: Generate initiate calldata
echo "Step 3: Generating initiate calldata..."
avalanche blockchain addValidator $BLOCKCHAIN_NAME \
  --mainnet \
  --key $P_CHAIN_KEY \
  --node-id $NODE_ID \
  --bls-public-key $BLS_PUB \
  --bls-proof-of-possession $BLS_POP \
  --weight $WEIGHT \
  --balance 0.5 \
  --staking-period $STAKING_PERIOD \
  --delegation-fee $DELEGATION_FEE \
  --rewards-recipient $SAFE_ADDR \
  --remaining-balance-owner $P_CHAIN_ADDR \
  --disable-owner $P_CHAIN_ADDR \
  --external-evm-signature \
  --validator-owner $SAFE_ADDR

echo ""
echo "Step 4: Execute via Safe"
echo "  - Create tx in validator Safe ($SAFE_ADDR)"
echo "  - Use calldata from above"
echo "  - Collect 3/5 signatures & execute"
echo ""
read -p "Enter initiate tx hash after Safe execution: " TX_HASH
echo ""

# Step 5: Complete registration
echo "Step 5: Completing registration..."
avalanche blockchain addValidator $BLOCKCHAIN_NAME \
  --mainnet \
  --key $P_CHAIN_KEY \
  --node-id $NODE_ID \
  --bls-public-key $BLS_PUB \
  --bls-proof-of-possession $BLS_POP \
  --weight $WEIGHT \
  --balance 0.5 \
  --staking-period $STAKING_PERIOD \
  --delegation-fee $DELEGATION_FEE \
  --rewards-recipient $SAFE_ADDR \
  --remaining-balance-owner $P_CHAIN_ADDR \
  --disable-owner $P_CHAIN_ADDR \
  --validator-owner $SAFE_ADDR \
  --initiate-tx-hash $TX_HASH

echo "✅ Day 1 migration complete! Validator $NODE_ID is now PoS"
echo ""
echo "=========================================="
echo "Repeat tomorrow with next validator"
echo "=========================================="
```

---

## Multi-Sig Workflow Summary

### Flow Diagram

```
┌─────────────────────────────────────────────────────────────────┐
│                    MIGRATION WORKFLOW                            │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  PHASE 0: PREPARATION (Before Day 1)                            │
│  ├─ Create 55 unique signer EOAs                                │
│  ├─ Deploy 11 Safe wallets (3/5 each)                           │
│  ├─ Prepare P-Chain addresses for each validator                │
│  ├─ Fund each Safe with gas                                     │
│  └─ Mint 11M KITE to admin (0x7700...) ← Multi-sig              │
│                                                                  │
│  PHASE 1: DEPLOY & CONFIGURE (Before Day 1)                     │
│  ├─ Deploy and initialize KiteStakingManager                    │
│  └─ Transfer ValidatorManager ownership ← Multi-sig              │
│                                                                  │
│  PHASE 2: DAILY MIGRATION (11 Days - Churn Limit)               │
│  For each day (validator 1 to 11):                              │
│  ├─ Remove PoA validator N (permissionless)                     │
│  ├─ Transfer 1M KITE to validator N's Safe ← Admin multi-sig    │
│  ├─ CLI generates initiate calldata                             │
│  ├─ Validator N's Safe signs (3/5) ← Multi-sig                  │
│  ├─ Safe executes initiate tx (stakes KITE)                     │
│  └─ CLI completes registration (permissionless)                 │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### Multi-Sig Addresses

| Multi-Sig | Address | Purpose |
|-----------|---------|---------|
| NativeMinter Admin | `0x7700D95B5250045400FE531f8886D9B02eA50801` | Mint & distribute KITE |
| ValidatorManager Owner | (existing) | Transfer ownership |
| Validator 1-11 Safes | (to be created) | Stake & manage validators |

### Transaction Count

| Phase | Transactions | Multi-Sig | Signer |
|-------|--------------|-----------|--------|
| Mint 11M KITE | 1 | ✅ | 0x7700... |
| Transfer ownership | 1 | ✅ | Existing owner |
| Daily: Remove PoA validator | 11 (1/day) | ❌ | Permissionless |
| Daily: Distribute 1M KITE | 11 (1/day) | ✅ | 0x7700... |
| Daily: Add PoS validator (initiate) | 11 (1/day) | ✅ | Each validator Safe |
| Daily: Add PoS validator (complete) | 11 (1/day) | ❌ | Permissionless |

**Multi-sig transactions breakdown**:
- 0x7700... admin: 1 (mint) + 11 (distribute) = **12 txs**
- Existing owner: 1 (transfer) = **1 tx**
- Each validator Safe: 1 each = **11 txs**
- **Total: 24 multi-sig transactions over 11 days**

---

## Permission Summary

### Permission Table

| Operation | Function | Permission | Multi-Sig? |
|-----------|----------|------------|------------|
| Transfer ownership | `transferOwnership` | Current owner | ✅ Existing owner |
| **Add PoS Validator** | | | |
| - Initiate | `initiateValidatorRegistration` | Staker/Owner only | ✅ Validator's Safe |
| - Complete | `completeValidatorRegistration` | Permissionless | ❌ No |
| **Remove PoA Validator** | | | |
| - Initiate | `initiateValidatorRemoval` | Permissionless | ❌ No |
| - Complete | `completeValidatorRemoval` | Permissionless | ❌ No |
| **Remove PoS Validator** | | | |
| - Initiate | `initiateValidatorRemoval` | Owner only | ✅ Validator's Safe |
| - Complete | `completeValidatorRemoval` | Permissionless | ❌ No |

### Why Permissionless?

- **Complete operations**: Only verify warp message from P-Chain. Authorization proven during initiate.
- **PoA removal**: Bootstrap validators have `MinStakeDuration = 0`, no staking owner.

### PoS Validator Removal Rules

After validators are added as PoS, removal requires:

1. **Owner signature**: Only the validator's Safe (3/5) can initiate
2. **Wait period**: Cannot remove before `startTime + 2 weeks`

```bash
# Future PoS validator removal (after 2 weeks minimum stake)
avalanche blockchain removeValidator kite \
  --mainnet \
  --node-id <NODE_ID> \
  --external-evm-signature \
  --validator-manager-owner <VALIDATOR_SAFE_ADDRESS>

# After Safe signs:
avalanche blockchain removeValidator kite \
  --mainnet \
  --node-id <NODE_ID> \
  --initiate-tx-hash <SIGNED_TX_HASH>
```

---

## Troubleshooting

### Error: "doesn't have authorization to remove the validator"

**Cause**: Trying to remove a PoS validator with wrong owner address.

**Solution**: Use the correct validator's Safe address with `--validator-manager-owner`.

### Error: "can't remove validator before X"

**Cause**: Minimum stake duration (2 weeks) has not passed.

**Solution**: Wait until `startTime + 1,209,600 seconds` has elapsed.

### Error: "node X is not a L1 validator"

**Cause**: Node ID not found in validator set.

**Solution**: Verify the node ID is correct and the validator is active.

### Safe transaction requires more signatures

**Cause**: Only 2 of 5 signers approved.

**Solution**: Get 3rd signer to approve (threshold is 3/5).

### KiteStakingManager initialization failed

**Cause**: Ownership not transferred yet.

**Solution**: Ensure ValidatorManager ownership transfer completed first.

---

## Execution Checklist

### Phase 0: Preparation
```
□ 55 unique signer EOAs generated/collected
□ 11 Safe wallets deployed (3/5 each)
  □ Safe 1: __________ (Validator 1)
  □ Safe 2: __________ (Validator 2)
  □ Safe 3: __________ (Validator 3)
  □ Safe 4: __________ (Validator 4)
  □ Safe 5: __________ (Validator 5)
  □ Safe 6: __________ (Validator 6)
  □ Safe 7: __________ (Validator 7)
  □ Safe 8: __________ (Validator 8)
  □ Safe 9: __________ (Validator 9)
  □ Safe 10: __________ (Validator 10)
  □ Safe 11: __________ (Validator 11)
□ All Safes funded with gas
□ Mint 11M KITE (via 0x7700... multi-sig)
  □ NativeMinter.mintNativeCoin() calldata generated
  □ Multi-sig signed
  □ Transaction executed
  □ Balance verified: 11,000,000 KITE
```

### Phase 1: Deploy & Configure
```
□ KiteStakingManager deployed: __________
□ ValidatorManager ownership transferred
  □ Existing owner multi-sig signed
  □ Transaction executed
  □ Ownership verified
□ KiteStakingManager initialized
  □ minStakeAmount: 1,000,000 KITE ✓
  □ maxStakeAmount: 10,000,000 KITE ✓
  □ minStakeDuration: 1,209,600 seconds ✓
```

### Phase 2: Remove PoA Validators (Permissionless)
```
□ Validator 1 removed: NodeID-__________
□ Validator 2 removed: NodeID-__________
□ Validator 3 removed: NodeID-__________
□ Validator 4 removed: NodeID-__________
□ Validator 5 removed: NodeID-__________
□ Validator 6 removed: NodeID-__________
□ Validator 7 removed: NodeID-__________
□ Validator 8 removed: NodeID-__________
□ Validator 9 removed: NodeID-__________
□ Validator 10 removed: NodeID-__________
□ Validator 11 removed: NodeID-__________
```

### Phase 3: Distribute KITE (via 0x7700... multi-sig)
```
□ Safe 1 received 1M KITE - tx: __________
□ Safe 2 received 1M KITE - tx: __________
□ Safe 3 received 1M KITE - tx: __________
□ Safe 4 received 1M KITE - tx: __________
□ Safe 5 received 1M KITE - tx: __________
□ Safe 6 received 1M KITE - tx: __________
□ Safe 7 received 1M KITE - tx: __________
□ Safe 8 received 1M KITE - tx: __________
□ Safe 9 received 1M KITE - tx: __________
□ Safe 10 received 1M KITE - tx: __________
□ Safe 11 received 1M KITE - tx: __________
□ All balances verified
```

### Phase 4: Add PoS Validators
```
For each validator (1-11):
□ Validator #__
  □ Node ID: __________
  □ Safe address: __________
  □ Safe KITE balance: 1,000,000 KITE ✓
  □ Initiate calldata generated
  □ Safe signed (3/5): ____/____/____ 
  □ Initiate tx hash: __________
  □ Complete tx executed
  □ ValidationID: __________
```

### Final Verification
```
□ All 11 validators active on P-Chain
□ Total staked: 11,000,000 KITE
□ Validator set healthy
□ 0x7700... balance: 0 KITE (all distributed)
```

---

## References

- [Avalanche CLI Documentation](https://docs.avax.network/tooling/avalanche-cli)
- [ICM Contracts - Validator Manager](https://github.com/ava-labs/icm-contracts/tree/main/contracts/validator-manager)
- [Safe (Gnosis Safe) Documentation](https://docs.safe.global/)
- [Avalanche Tooling SDK](https://github.com/ava-labs/avalanche-tooling-sdk-go)
