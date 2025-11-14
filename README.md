# Part 1 – Run a Drosera Operator on Ethereum Mainnet (Docker)

### 1. Requirements

* Linux server (e.g. Ubuntu 22.04)
* Open ports: `31313/tcp` (P2P) and `31314/tcp` (HTTP)
* One **Ethereum RPC URL** (mainnet), e.g. `https://…`
* One **operator wallet** with a little ETH (at least $10) for gas (fresh EOA recommended)


---

### 2. Install Docker

```bash
curl -fsSL https://get.docker.com | sh
```

---

### 3. Register your operator (one-time)

```bash
docker run --rm ghcr.io/drosera-network/drosera-operator:latest \
  register \
  --eth-rpc-url https://YOUR_MAINNET_RPC_URL \
  --eth-private-key 0xYOUR_OPERATOR_PRIVATE_KEY
```

This writes your BLS public key into the Drosera registry on mainnet.

---

### 4. Create `docker-compose.yml`

In some directory, e.g. `~/drosera-operator`:

```bash
mkdir -p ~/drosera-operator
cd ~/drosera-operator
nano docker-compose.yml
```
Edit in docker-compose.yml:

* `https://YOUR_MAINNET_RPC_URL`
* `0xYOUR_OPERATOR_PRIVATE_KEY`
* `VPS_PUBLIC_IP`

Paste and edit:

```yaml
services:
  drosera-operator:
    image: ghcr.io/drosera-network/drosera-operator:latest
    container_name: drosera-operator
    restart: always
    ports:
      - "31313:31313"   # P2P
      - "31314:31314"   # HTTP (health)
    environment:
      - DRO__DROSERA_ADDRESS=0x01C344b8406c3237a6b9dbd06ef2832142866d87  # Drosera mainnet core
      - DRO__ETH__CHAIN_ID=1
      - DRO__ETH__RPC_URL=https://YOUR_MAINNET_RPC_URL  # EDIT THIS
      - DRO__ETH__PRIVATE_KEY=0xYOUR_OPERATOR_PRIVATE_KEY  # EDIT THIS
      - DRO__DATA_DIR=/data/.drosera/data
      - DRO__LISTEN_ADDRESS=0.0.0.0
      - DRO__NETWORK__P2P_PORT=31313
      - DRO__NETWORK__EXTERNAL_P2P_ADDRESS=VPS_PUBLIC_IP  # EDIT THIS
      - DRO__SERVER__PORT=31314
      - DRO__GAS_REIMBURSEMENT_REQUIRED=true
    volumes:
      - drosera_data:/data
    command: ["node"]
volumes:
  drosera_data: {}
```


---

### 5. Start and check health

```bash
cd ~/drosera-operator
docker compose up -d
docker compose logs -f
```

Health check (from your laptop or anywhere):

```bash
curl -s "http://YOUR_SERVER_PUBLIC_IP:31314" \
  -H "Content-Type: application/json" \
  --data '{"jsonrpc":"2.0","method":"drosera_healthCheck","params":[],"id":1}'
```

You should see `{"ok":true}` in the result.

---

# Part 2 – Deploy Vault TVL Drop Trap on Mainnet

Repo: [https://github.com/R1ghTsS/vault-tvl-drop-trap](https://github.com/R1ghTsS/vault-tvl-drop-trap)

You need a **separate “creator” wallet** (can be same as operator, but better separate) with ETH for deployment and hydration.

---

### 1. Install Foundry + Drosera CLI

```bash
curl -L https://foundry.paradigm.xyz | bash
foundryup

curl -L https://app.drosera.io/install | bash
droseraup
```

---

### 2. Clone the repo and configure the vault

```bash
git clone https://github.com/R1ghTsS/vault-tvl-drop-trap.git
cd vault-tvl-drop-trap
nano src/VaultTVLDropTrap.sol
```

In `VaultTVLDropTrap.sol`, set:

```solidity
address public constant VAULT = 0xYOUR_MAINNET_VAULT_ADDRESS;
uint256 public constant DROP_THRESHOLD_BPS = 500;  // 5% (change if you want)
uint256 public constant MIN_ABSOLUTE_DROP = 0;     // optional floor
```
Sample Vault You Can Watch
```bash
Lido stETH – 0xae7ab96520DE3A18E5e111B5EaAb095312D7fE84
Rocket Pool rETH – 0xae78736Cd615f374D3085123A210448E74Fc6393
Coinbase cbETH – 0xBe9895146f7AF43049ca1c1AE358B0541Ea49704
Aave WETH – 0x4d5F47FA6A74757f35C14fD3a6Ef8E3C9BC514E8
Aave weETH – 0xBdfa7b7893081B35Fb54027489e2Bc7A38275129
Aave wstETH – 0x0B925eD163218f6662a35e0f0371Ac234f9E9371
Aave USDT – 0x23878914EFE38d27C4D67Ab83ed1b93A74D4086a
Aave USDC – 0x98C23E9d8f34FEFb1B7BD6a91B7FF122F4e16F5c
Aave WBTC – 0x5Ee5bf7ae06D1Be5997A1A72006FE6C607eC6DE8
```
Then build:

```bash
forge build
```

---

### 3. Deploy the response contract (`VaultTVLResponse`)

This contract just emits an event when the trap fires.

```bash
cast send \
  --rpc-url https://YOUR_MAINNET_RPC_URL \
  --private-key 0xYOUR_CREATOR_PRIVATE_KEY \
  --create "$(jq -r '.bytecode.object' out/VaultTVLResponse.sol/VaultTVLResponse.json)"
```

This prints a `transactionHash` and a `contractAddress`.
The `contractAddress` is your **response contract**, e.g.:

```text
contractAddress      0xf2534e3e64453081E567ECA7d06259CFEB0afC2d
```

Write that down.

(If you want to re-check later:
`cast receipt --rpc-url https://YOUR_MAINNET_RPC_URL 0xTX_HASH contractAddress`)

---

### 4. Create `drosera.toml` for mainnet

In `~/vault-tvl-drop-trap`:

```bash
nano drosera.toml
```
Fill in:

* `https://YOUR_MAINNET_RPC_URL`
* `0xYOUR_RESPONSE_CONTRACT_ADDRESS` (from step 3)
* `0xYOUR_OPERATOR_ADDRESS` (the EOA whose private key you used for the operator)
Paste and edit:

```toml
ethereum_rpc    = "https://YOUR_MAINNET_RPC_URL"
drosera_rpc     = "https://relay.ethereum.drosera.io"
eth_chain_id    = 1
drosera_address = "0x01C344b8406c3237a6b9dbd06ef2832142866d87"

[traps]

[traps.vault_tvl_drop]
path                    = "out/VaultTVLDropTrap.sol/VaultTVLDropTrap.json"
response_contract       = "0xYOUR_RESPONSE_CONTRACT_ADDRESS"
response_function       = "handleVaultDrop(string)"
cooldown_period_blocks  = 32
min_number_of_operators = 1
max_number_of_operators = 3
block_sample_size       = 20
private_trap            = true
whitelist               = ["0xYOUR_OPERATOR_ADDRESS"]
```

---

### 5. Plan → Dryrun → Apply

From `~/vault-tvl-drop-trap`:

```bash
drosera plan
drosera dryrun
DROSERA_PRIVATE_KEY=0xYOUR_CREATOR_PRIVATE_KEY drosera apply
```

`apply` will deploy the trap and create the Trap Config on mainnet.
It prints a **Trap Config address** — call it `0xTRAP_CONFIG_ADDRESS`. Save it.

---

### 6. *OPTIONAL*: Hydrate (fund rewards) and Bloom Boost (fund gas) 

```bash
DROSERA_PRIVATE_KEY=0xYOUR_CREATOR_PRIVATE_KEY \
  drosera hydrate --trap-address 0xTRAP_CONFIG_ADDRESS --dro-amount 1500

DROSERA_PRIVATE_KEY=0xYOUR_CREATOR_PRIVATE_KEY \
  drosera bloomboost --trap-address 0xTRAP_CONFIG_ADDRESS --eth-amount 0.05
```

Adjust DRO and ETH amounts as you like.

---

### 7. Opt your Operator into the trap

Run this on the **operator server** (where Docker is running):

```bash
cd ~/drosera-operator
docker run --rm ghcr.io/drosera-network/drosera-operator:latest \
  optin \
  --eth-rpc-url https://YOUR_MAINNET_RPC_URL \
  --eth-private-key 0xYOUR_OPERATOR_PRIVATE_KEY \
  --trap-config-address 0xTRAP_CONFIG_ADDRESS
```

Then:

```bash
cd ~/drosera-operator
docker compose logs -f
```

You should see something like:

```text
Number of opted in traps: 1
Trap Enzyme Service started ...
Trap Attestation Service started ...
Trap Submission Service started ...
```
