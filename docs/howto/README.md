# How To Use Cast: a Trustless DEX on Bitcoin with Charms

**TL;DR**: This guide demonstrates how to create, cancel, and fill orders on a fully trustless decentralized exchange
running natively on Bitcoin. We'll trade [BRO](https://bro.charms.dev) tokens for sats—no bridges, no wrapped tokens, no custodians—just ZK proofs enforcing trade logic directly on the Bitcoin blockchain.

---

## What is Charms Cast?

[Charms](https://charms.dev) brings programmable assets to Bitcoin through zero-knowledge proofs. **Charms Cast** is a
decentralized exchange built on this protocol, enabling peer-to-peer trading of Charms tokens without intermediaries.

Key features:

- **Sell Orders (Ask)**: Offer tokens in exchange for BTC
- **Buy Orders (Bid)**: Offer BTC in exchange for tokens
- **Partial Fills**: Orders can be filled incrementally across multiple transactions
- **All-or-None**: Orders that must be filled completely or not at all
- **Cancellation**: Makers can cancel their orders with a cryptographic signature

Every operation is verified by a ZK proof (called a "spell") embedded in the Bitcoin transaction itself.

---

## Prerequisites

### Tools You'll Need

1. **Charms CLI** — For proving and verifying spells
  - [github.com/CharmsDev/charms](https://github.com/CharmsDev/charms)
  - [docs.charms.dev](https://docs.charms.dev)

2. **scrolls-nonce** — Computes deterministic nonces for Scrolls addresses
  - [github.com/CharmsDev/scrolls-nonce](https://github.com/CharmsDev/scrolls-nonce)

3. **sign-txs** — Signs transactions using your Bitcoin Core wallet
  - [github.com/CharmsDev/sign-txs](https://github.com/CharmsDev/sign-txs)
  - Works with local `bitcoin-cli` and (optionally) bitcoind in a Docker container (via `--bitcoind-container` flag or
    `BITCOIND_CONTAINER` env var)

4. **cancel-msg** — Generates and signs order cancellation messages
  - [github.com/CharmsDev/cancel-msg](https://github.com/CharmsDev/cancel-msg)

5. **Bitcoin Core** — With a wallet containing your signing keys

6. **Cast Contract Binary** — Download `charms-cast-v0.2.0.wasm` from:
  - [github.com/CharmsDev/cast-releases/releases/tag/v0.2.0](https://github.com/CharmsDev/cast-releases/releases/tag/v0.2.0)

### Operator Parameters

The DEX operator provides signed parameters that authorize trading. These include fee configuration and the contract app
ID:

```yaml
params:
  - fulfill:
      app: b/00000...00000/a471d3fcc436ae7cbc0e0c82a68cdc8e003ee21ef819e1acf834e11c43ce47d8
      fee_config:
        taker_fee: 50        # 0.50% of order value
        matcher_fee: 2000    # 20% of bid-ask spread
        fee_address: bc1qxxxjm06n50uugxewxe5r5w5tskqwq4gkwrm0al
      min_value: 10000       # Minimum order value: 10,000 sats
  - 20f0aa694f7382a64cce14ebffa9f62428d4655c4f53826f1b745b556205edda...  # Operator signature
```

---

## Understanding Scrolls Addresses

DEX orders live at special **Scrolls addresses**. These are Bitcoin addresses controlled by the Scrolls signing service,
which only signs transactions that satisfy the smart contract rules (verified via ZK proof).

### Generating a Scrolls Address

Each Scrolls address requires a deterministic 64-bit nonce derived from a funding UTXO:

```bash
# Step 1: Compute the nonce
# Usage: scrolls-nonce <funding_utxo_id> <output_index>
scrolls-nonce "2efcde0d11cca5c23a415de6cb917e6da5b747975911e7855eb29d0737a68873:2" 0
# Output: 2592674606473911827

# Step 2: Get the Scrolls address from the Scrolls API
curl "https://scrolls-v9.charms.dev/main/address/2592674606473911827"
# Output: "bc1qzhdm5ee85jk7xwsue2ewnp9eh886mqkg3sfsem"
```

The nonce is computed as: `sha256("{funding_utxo_id}:{output_index}")` → first 8 bytes as little-endian u64.

### The prev_txs Parameter

When proving a spell, you must provide the raw hex of transactions that created any UTXOs being spent. This lets the
prover verify the charms attached to those UTXOs:

```bash
# Fetch a transaction's raw hex
curl -s "https://mempool.space/api/tx/TXID/hex" > prev_txs.txt
```

---

## The Proving Workflow

Every DEX operation follows this workflow:

1. **Write the spell** — A YAML file describing inputs, outputs, and contract state
2. **Mock prove** — Test with `--mock` flag (executes instantly, verifies logic)
3. **Real prove** — Remove `--mock` to generate the actual ZK proof
4. **Sign** — Use `sign-txs` for your wallet inputs, Scrolls API for order inputs
5. **Broadcast** — Submit the fully signed transaction

---

## Creating a Sell Order

Let's create an order to sell 50 [BRO](https://bro.charms.dev) tokens for 10,000 sats (200 sats per BRO). BRO is a Charms token used throughout this guide to demonstrate DEX operations.

### The Spell Structure

```yaml
version: 9

apps:
  $CAST: b/0000000000000000000000000000000000000000000000000000000000000000/a471d3fcc436ae7cbc0e0c82a68cdc8e003ee21ef819e1acf834e11c43ce47d8
  $BRO: t/3d7fe7e4cea6121947af73d70e5119bebd8aa5b7edfe74bfaf6e779a1847bd9b/c975d4e0c292fb95efbda5c13312d6ac1d8b5aeff7f0f1e5578645a2da70ff5f

private_inputs:
  $CAST:
    params:
      - fulfill:
          app: b/0000000000000000000000000000000000000000000000000000000000000000/a471d3fcc436ae7cbc0e0c82a68cdc8e003ee21ef819e1acf834e11c43ce47d8
          fee_config:
            taker_fee: 50
            matcher_fee: 2000
            fee_address: bc1qxxxjm06n50uugxewxe5r5w5tskqwq4gkwrm0al
          min_value: 10000
      - 20f0aa694f7382a64cce14ebffa9f62428d4655c4f53826f1b745b556205edda317b64649a68f214e3fd533929875e5576d6f9293cfda7ea80924ffeb16bbf07ea
    edit_orders:
      cancel: {}

ins:
  - utxo_id: <your_token_utxo>
    charms:
      $BRO: 5000000000  # 50 BRO (8 decimals)

outs:
  # Order output at Scrolls address
  - address: bc1q<scrolls_address>
    coin: 300  # Dust amount
    charms:
      $BRO: 5000000000
      $CAST:
        maker: bc1q<your_payment_address>
        exec_type: all_or_none
        side: ask
        price: [1, 500000]  # 1 sat per 500,000 units = 200 sats/BRO
        amount: 10000       # Total BTC requested
        quantity: 5000000000
        asset:
          token: t/3d7fe7e4cea6121947af73d70e5119bebd8aa5b7edfe74bfaf6e779a1847bd9b/c975d4e0c292fb95efbda5c13312d6ac1d8b5aeff7f0f1e5578645a2da70ff5f
```

### Build, Sign, and Broadcast

```bash
# Prove the spell
charms spell prove \
  --spell create-order.yaml \
  --app-bins charms-cast-v0.2.0.wasm \
  --prev-txs "$(cat prev_txs.txt)" \
  --funding-utxo "TXID:VOUT" \
  --funding-utxo-value 50000 \
  --change-address "bc1q..." \
  --fee-rate 2 > tx_to_sign.json

# Sign with your wallet
sign-txs tx_to_sign.json > tx_signed.json

# Broadcast
curl -X POST "https://mempool.space/api/tx" -d "$(jq -r '.[0].bitcoin' tx_signed.json)"
```

---

## Canceling and Replacing an Order

To modify an order, you cancel the existing one and create a replacement atomically in a single transaction.

### Generating the Cancellation Signature

The maker must sign a message authorizing the cancellation. The message format is:

```
{utxo_id} {outputs_hash}
```

Where `outputs_hash` is the double-SHA256 of the serialized spell outputs (both charms data and native coin outputs).

Use the **cancel-msg** tool to generate and sign this message:

```bash
# Step 1: Generate the cancellation message from your spell
cancel-msg message cancel-order.yaml "TXID:VOUT"
# Output: TXID:VOUT 9f86d081884c...

# Step 2: Sign the message with your private key
cancel-msg sign "TXID:VOUT 9f86d081884c..." \
  --xprv "xprv9s21ZrQH143K..." \
  --path "0/0"
# Output: 1fff1122...  (hex signature)
```

The resulting signature goes in your spell's `private_inputs`.

### Complete Spell

This spell cancels an existing 50 BRO order, combines it with 49 BRO from another UTXO, and creates a new 99 BRO order
with partial fills enabled at a new price of 250 sats/BRO:

```yaml
version: 9

apps:
  $CAST: b/0000000000000000000000000000000000000000000000000000000000000000/a471d3fcc436ae7cbc0e0c82a68cdc8e003ee21ef819e1acf834e11c43ce47d8
  $BRO: t/3d7fe7e4cea6121947af73d70e5119bebd8aa5b7edfe74bfaf6e779a1847bd9b/c975d4e0c292fb95efbda5c13312d6ac1d8b5aeff7f0f1e5578645a2da70ff5f

private_inputs:
  $CAST:
    params:
      - fulfill:
          app: b/0000000000000000000000000000000000000000000000000000000000000000/a471d3fcc436ae7cbc0e0c82a68cdc8e003ee21ef819e1acf834e11c43ce47d8
          fee_config:
            taker_fee: 50
            matcher_fee: 2000
            fee_address: bc1qxxxjm06n50uugxewxe5r5w5tskqwq4gkwrm0al
          min_value: 10000
      - 20f0aa694f7382a64cce14ebffa9f62428d4655c4f53826f1b745b556205edda317b64649a68f214e3fd533929875e5576d6f9293cfda7ea80924ffeb16bbf07ea
    edit_orders:
      cancel:
        # Cancellation signature for input 0
        0: 1fff1122edd917b6d2fe7917a8e1e5febe87cb266159a6412c2bf4cefdc6d6f49f2f60a0c56e1e7dd6dbec62cca75196da2bf0ddf55e9593fc1ae1512d09eed283

ins:
  # Input 0: The order to cancel (50 BRO)
  - utxo_id: 2efcde0d11cca5c23a415de6cb917e6da5b747975911e7855eb29d0737a68873:0
    charms:
      $BRO: 5000000000
      $CAST:
        maker: bc1qvlxzrdx5q8e56vpf45hae85kg8vjrze9qceada
        exec_type: all_or_none
        side: ask
        price: [1, 500000]
        amount: 10000
        quantity: 5000000000
        asset:
          token: t/3d7fe7e4cea6121947af73d70e5119bebd8aa5b7edfe74bfaf6e779a1847bd9b/c975d4e0c292fb95efbda5c13312d6ac1d8b5aeff7f0f1e5578645a2da70ff5f

  # Input 1: Additional BRO tokens (49 BRO)
  - utxo_id: b0751a9e34e4cc0663b6361dcd7b9acc69a97e7d35e245b270d67dfef78741e1:1
    charms:
      $BRO: 4900000000

outs:
  # Output 0: New order at fresh Scrolls address
  # 99 BRO total (50 + 49), allowing partial fills
  - address: bc1qzhdm5ee85jk7xwsue2ewnp9eh886mqkg3sfsem
    coin: 300
    charms:
      $BRO: 9900000000  # 99 BRO
      $CAST:
        maker: bc1qn4z8s2f43fkdf2eys0pt23nsm668w3rx472n2s
        exec_type:
          partial: {}  # Now allows partial fills
        side: ask
        price: [1, 400000]  # 250 sats per BRO
        amount: 24750       # 99 BRO × 250 sats
        quantity: 9900000000
        asset:
          token: t/3d7fe7e4cea6121947af73d70e5119bebd8aa5b7edfe74bfaf6e779a1847bd9b/c975d4e0c292fb95efbda5c13312d6ac1d8b5aeff7f0f1e5578645a2da70ff5f
```

### Signing via Scrolls API

The order input is at a Scrolls address, so it needs the Scrolls signature:

```bash
curl -X POST "https://scrolls-v9.charms.dev/main/sign" \
  -H "Content-Type: application/json" \
  -d '{
    "tx_to_sign": "<tx_hex>",
    "prev_txs": ["<prev_tx_hex>"],
    "sign_inputs": [{"index": 0, "nonce": 2592674606473911827}]
  }'
```

---

## Partially Filling an Order

A taker can partially fill an order with `exec_type: partial`. The unfilled portion remains as a new order.

### Key Points

1. The **remainder order** must reference the original via `exec_type.partial.from`
2. Same maker, side, asset, and price as original
3. Reduced quantity and amount proportionally
4. Must stay at the **same Scrolls address** as the original

### Fee Calculation

```
Taker fee = filled_amount × taker_fee / 10000
```

For a 7,500 sat fill with `taker_fee: 50` (0.50%):

```
Fee = 7500 × 50 / 10000 = 37 sats
```

### Complete Spell

This spell partially fills the 99 BRO order. The taker buys 30 BRO for 7,500 sats, leaving a remainder order for 69 BRO:

```yaml
version: 9

apps:
  $CAST: b/0000000000000000000000000000000000000000000000000000000000000000/a471d3fcc436ae7cbc0e0c82a68cdc8e003ee21ef819e1acf834e11c43ce47d8
  $BRO: t/3d7fe7e4cea6121947af73d70e5119bebd8aa5b7edfe74bfaf6e779a1847bd9b/c975d4e0c292fb95efbda5c13312d6ac1d8b5aeff7f0f1e5578645a2da70ff5f

private_inputs:
  $CAST:
    params:
      - fulfill:
          app: b/0000000000000000000000000000000000000000000000000000000000000000/a471d3fcc436ae7cbc0e0c82a68cdc8e003ee21ef819e1acf834e11c43ce47d8
          fee_config:
            taker_fee: 50
            matcher_fee: 2000
            fee_address: bc1qxxxjm06n50uugxewxe5r5w5tskqwq4gkwrm0al
          min_value: 10000
      - 20f0aa694f7382a64cce14ebffa9f62428d4655c4f53826f1b745b556205edda317b64649a68f214e3fd533929875e5576d6f9293cfda7ea80924ffeb16bbf07ea

ins:
  # Input 0: The sell order to partially fill
  - utxo_id: 75681f0f1d8adacfee1506330560b28b7fe11bc666f8d6e791b19d79a5c9291c:0
    charms:
      $BRO: 9900000000  # 99 BRO
      $CAST:
        maker: bc1qn4z8s2f43fkdf2eys0pt23nsm668w3rx472n2s
        exec_type:
          partial: {}
        side: ask
        price: [1, 400000]  # 250 sats per BRO
        amount: 24750
        quantity: 9900000000
        asset:
          token: t/3d7fe7e4cea6121947af73d70e5119bebd8aa5b7edfe74bfaf6e779a1847bd9b/c975d4e0c292fb95efbda5c13312d6ac1d8b5aeff7f0f1e5578645a2da70ff5f

  # Input 1: Taker's funding UTXO (provides sats for payment and fees)
  - utxo_id: 75681f0f1d8adacfee1506330560b28b7fe11bc666f8d6e791b19d79a5c9291c:3

outs:
  # Output 0: Taker receives 30 BRO
  - address: bc1qasm6srsv48tp2gqs6ja9quuml0ztaeztarx2x0
    coin: 300
    charms:
      $BRO: 3000000000  # 30 BRO

  # Output 1: Remainder order - 69 BRO still for sale
  # Must stay at the SAME Scrolls address as the original order
  - address: bc1qzhdm5ee85jk7xwsue2ewnp9eh886mqkg3sfsem
    coin: 300
    charms:
      $BRO: 6900000000  # 69 BRO
      $CAST:
        maker: bc1qn4z8s2f43fkdf2eys0pt23nsm668w3rx472n2s
        exec_type:
          partial:
            from: 75681f0f1d8adacfee1506330560b28b7fe11bc666f8d6e791b19d79a5c9291c:0
        side: ask
        price: [1, 400000]  # Same price
        amount: 17250       # 69 BRO × 250 sats
        quantity: 6900000000
        asset:
          token: t/3d7fe7e4cea6121947af73d70e5119bebd8aa5b7edfe74bfaf6e779a1847bd9b/c975d4e0c292fb95efbda5c13312d6ac1d8b5aeff7f0f1e5578645a2da70ff5f

  # Output 2: Maker receives BTC for the filled portion
  # 30 BRO at 250 sats/BRO = 7,500 sats + 300 dust offset
  - address: bc1qn4z8s2f43fkdf2eys0pt23nsm668w3rx472n2s
    coin: 7800

  # Output 3: Fee output
  # Taker fee: 7,500 × 50 / 10,000 = 37 sats
  # Scrolls protocol fee: ~960 sats
  # Total: ~1,000 sats
  - address: bc1qxxxjm06n50uugxewxe5r5w5tskqwq4gkwrm0al
    coin: 1000
```

---

## Live Example: Complete Workflow

We executed this entire workflow on Bitcoin mainnet. Here are the actual transactions:

| Step | Description                                                     | Transaction                                                                                                |
|------|-----------------------------------------------------------------|------------------------------------------------------------------------------------------------------------|
| 1    | Create sell order (50 BRO @ 200 sats/BRO)                       | [b0751a9e...e1](https://mempool.space/tx/b0751a9e34e4cc0663b6361dcd7b9acc69a97e7d35e245b270d67dfef78741e1) |
| 2    | Contract upgrade (v0.1 → v0.2)                                  | [2efcde0d...73](https://mempool.space/tx/2efcde0d11cca5c23a415de6cb917e6da5b747975911e7855eb29d0737a68873) |
| 3    | Cancel & replace (99 BRO @ 250 sats/BRO, partial fills enabled) | [75681f0f...1c](https://mempool.space/tx/75681f0f1d8adacfee1506330560b28b7fe11bc666f8d6e791b19d79a5c9291c) |
| 4    | Partial fill (buy 30 BRO, 69 BRO remaining)                     | [5b4ad8e5...0b](https://mempool.space/tx/5b4ad8e5b435f5989707438ee90ae95059f0e7fbb016ba4ff36961806ec5d50b) |

---

## Contract Validation Rules

The Cast contract enforces these invariants:

**For Ask (Sell) Orders:**

- `amount = ceil(quantity × price)`
- Token output must match `order.quantity`
- Minimum coin output: 300 sats (dust limit)

**For Bid (Buy) Orders:**

- `quantity = ceil(amount / price)`
- Coin output ≥ `amount + 300` sats

**For Remainder Orders:**

- Must reference original via `exec_type.partial.from`
- Same maker, side, asset, and price
- Must stay at the same Scrolls address

**For Cancellations:**

- Valid signature from maker over `{utxo_id} {outputs_hash}`

---

## Resources

- **Charms Protocol**: [charms.dev](https://charms.dev)
- **Charms CLI**: [github.com/CharmsDev/charms](https://github.com/CharmsDev/charms)
- **scrolls-nonce**: [github.com/CharmsDev/scrolls-nonce](https://github.com/CharmsDev/scrolls-nonce)
- **sign-txs**: [github.com/CharmsDev/sign-txs](https://github.com/CharmsDev/sign-txs)
- **cancel-msg**: [github.com/CharmsDev/cancel-msg](https://github.com/CharmsDev/cancel-msg)
- **Cast Releases**: [github.com/CharmsDev/cast-releases/releases](https://github.com/CharmsDev/cast-releases/releases)
- **Scrolls API**: [scrolls-v9.charms.dev](https://scrolls-v9.charms.dev)
- **BRO Token**: [bro.charms.dev](https://bro.charms.dev)

---

*This guide demonstrates real, production transactions on Bitcoin mainnet. The Charms Cast DEX is live and operational.*
