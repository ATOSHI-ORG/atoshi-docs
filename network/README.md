# Atoshi network files

Canonical files needed to join an Atoshi L1 network. See the
[Node operation guide](../reference/node-operation.md) for the full join /
validator walkthrough.

## Testnet — `atoshi_88288-1`

| File | Purpose |
|---|---|
| [`testnet/genesis.json`](./testnet/genesis.json) | Canonical genesis. Must be byte-identical on every node. |
| [`testnet/genesis.sha256`](./testnet/genesis.sha256) | Checksum — verify after download. |

- **Chain ID:** `atoshi_88288-1` (EVM `88288`)
- **genesis.json sha256:** `fadb200084370a0af8adb9d16bfdf7c490f2a8be9a01ee6215bc75e28048f48a`
- **Archive P2P peer:** `186ccd0611db23ef434454a8c03437b932abb139@tm-rpc-testnet.atoshi.org:26656`
- **Min gas price:** `1000000000aatos` (1 gwei)

Download + verify:

```bash
curl -sL "https://raw.githubusercontent.com/ATOSHI-ORG/atoshi-docs/main/network/testnet/genesis.json" -o genesis.json
curl -sL "https://raw.githubusercontent.com/ATOSHI-ORG/atoshi-docs/main/network/testnet/genesis.sha256" | shasum -a 256 -c
```

## Mainnet — `atoshi_88188-1`

_Reserved. Genesis, checksum, and archive peer id will be published here at mainnet launch._
