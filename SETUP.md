# Avalanche CLI Setup Guide (KITE Mainnet)

## Prerequisites

- Go 1.24+ installed
- GCC (Linux) or Clang (macOS) installed (CGO is required)
- Git with SSH access to `git@github.com:gokite-ai/avalanche-cli.git`

## 1. Clone and Build

```bash
# Clone the repo
git clone git@github.com:gokite-ai/avalanche-cli.git
cd avalanche-cli

# Checkout the staking branch
git checkout staking

# Build with version (output binary: bin/avalanche)
./scripts/build.sh

# Verify the build
./bin/avalanche --version
# Expected output should show version 1.9.6

# (Optional) Add to PATH for convenience
export PATH="$PWD/bin:$PATH"
# Or copy to a directory already in PATH:
# cp bin/avalanche /usr/local/bin/avalanche
```

## 2. Place Subnet Config Files

The CLI reads subnet configs from `~/.avalanche-cli/subnets/<subnet-name>/`.

For the KITE mainnet subnet, you need to create the directory and place two files:

```bash
mkdir -p ~/.avalanche-cli/subnets/kite
```

Copy the following two files into `~/.avalanche-cli/subnets/kite/`:

- `sidecar.json` — subnet metadata, RPC endpoints, validator manager config
- `genesis.json` — chain genesis configuration

> Ask Lyon for these files. They are located in the repo at the `kite` subnet directory
> or can be copied from an existing setup at `~/.avalanche-cli/subnets/kite/`.

Verify the files are in place:

```bash
ls ~/.avalanche-cli/subnets/kite/
# Expected output: genesis.json  sidecar.json
```

## 3. Import P-Chain Sender Key

The `pchain-sender` key is used to pay for P-Chain transaction fees and L1 gas fees.

```bash
# Import the private key (Lyon will provide the hex private key)
avalanche key import pchain-sender --private-key <PRIVATE_KEY_HEX>
```

Replace `<PRIVATE_KEY_HEX>` with the actual private key provided to you (64-char hex string, no `0x` prefix).

Verify the key was imported:

```bash
avalanche key list --keys pchain-sender --mainnet
```

You should see the key with its P-Chain and C-Chain addresses and balances.

## 4. Verify Setup

Run these commands to verify everything is working:

```bash
# List configured blockchains
avalanche blockchain list

# Should show "kite" in the list with Type "Subnet-EVM"

# Describe the kite blockchain
avalanche blockchain describe kite --mainnet

# Check key balance
avalanche key list --keys pchain-sender --mainnet
```

## 5. Common Operations

### Remove a Validator

```bash
avalanche blockchain removeValidator kite \
  --mainnet \
  --key pchain-sender \
  --node-id <NODE_ID>
```

### Add a Validator (PoS)

#### Generate transaction

```bash
avalanche blockchain addValidator kite \
  --mainnet \
  --key pchain-sender \
  --node-id <NODE_ID> \
  --bls-public-key <BLS_PUBLIC_KEY> \
  --bls-proof-of-possession <BLS_POP> \
  --weight 1000000 \
  --balance 1 \
  --staking-period 336h \
  --delegation-fee 100 \
  --rewards-recipient <EVM_ADDRESS> \
  --remaining-balance-owner <P_CHAIN_ADDRESS> \
  --disable-owner <P_CHAIN_ADDRESS> \
  --validator-owner <EVM_ADDRESS> \
  --external-evm-signature
```

#### Complete registration

avalanche blockchain addValidator kite \
  --mainnet \
  --key pchain-sender \
  --node-id <NODE_ID> \
  --bls-public-key <BLS_PUBLIC_KEY> \
  --bls-proof-of-possession <BLS_POP> \
  --weight 1000000 \
  --balance 1 \
  --staking-period 336h \
  --delegation-fee 100 \
  --rewards-recipient <EVM_ADDRESS> \
  --remaining-balance-owner <P_CHAIN_ADDRESS> \
  --disable-owner <P_CHAIN_ADDRESS> \
  --validator-owner <EVM_ADDRESS> \
  --external-evm-signature
  --initiate-tx-hash TX_HASH

### Query Current Validators

```bash
curl -s -X POST -H "Content-Type: application/json" \
  -d '{"jsonrpc":"2.0","id":1,"method":"validators.getCurrentValidators","params":{}}' \
  "http://k8s-mainnet-gokiteva-abccd78129-2ed2ee9a9cbcf926.elb.us-east-1.amazonaws.com:9650/ext/bc/3USaEfTcoUhHxpKXvpAG916UKCUEyjrtkg2hBArBG3JyDP7my/validators" | jq
```

## Notes

- The `--staking-period` flag uses Go duration format. Use `h` for hours (e.g. `336h` = 2 weeks). `d` and `w` are NOT supported.
- The P-Chain sender key needs sufficient AVAX balance for validator balance deposits and tx fees.
- The validator owner EVM address needs sufficient KITE tokens on the L1 chain for gas fees.
- The ELB endpoint (`k8s-mainnet-gokiteva-...:9650`) requires IP whitelisting. Contact Paul/infra team to add your IP if needed.
