# Atoshi Documentation

This repository is the canonical technical documentation for the **Atoshi Chain** — a dual-layer blockchain that combines a Cosmos SDK L1 (with native EVM compatibility) and a Polygon CDK zkEVM L2, designed around a holding-based energy model that makes everyday transactions feel free for committed holders.

If you are looking for:

| Audience | Start here |
|---|---|
| First-time reader | [00-overview.md](./00-overview.md) — what Atoshi is, who it's for, what's novel |
| Builder / developer | [reference/api-guide.md](./reference/api-guide.md) — REST + JSON-RPC reference |
| Wallet integrator | [reference/wallet-integration.md](./reference/wallet-integration.md) |
| Researcher / analyst | [economics/](./economics/) and [modules/](./modules/) |
| Operator / validator | [architecture/](./architecture/) and [reference/node-operation.md](./reference/node-operation.md) |

## Table of contents

### Foundation
- [00 — Overview](./00-overview.md) — mission, design goals, what we built
- [01 — High-level architecture](./architecture/01-overview.md) — L1/L2/bridge mental model

### Architecture
- [Cosmos SDK L1 with EVM](./architecture/02-cosmos-l1.md)
- [Polygon CDK zkEVM L2](./architecture/03-polygon-l2.md)
- [Bridge between L1 and L2](./architecture/04-bridge.md)
- [Account model and key management](./architecture/05-accounts.md)

### Modules (each is a self-contained chapter)
- [Energy — gas via holding, not paying](./modules/01-energy.md)
- [Tokenomics — supply, release, halving](./modules/02-tokenomics.md)
- [Oracle — on-chain ATOS/USD price](./modules/03-oracle.md)
- [Bank — ATOS native transfers](./modules/04-bank.md)
- [EVM — Ethereum-compatible execution](./modules/05-evm.md)
- [Bridge — cross-layer asset movement](./modules/06-bridge.md)
- [Privacy — shielded transactions on L2](./modules/07-privacy.md)
- [Staking — validator economics](./modules/08-staking.md)
- [Governance — on-chain proposals with case studies](./modules/09-governance.md)

### Economics
- [Supply and circulation](./economics/01-supply.md)
- [Release schedule and halving curves](./economics/02-release-schedule.md)
- [Gas and energy economics](./economics/03-gas-fee.md)
- [Block rewards and validator income](./economics/04-block-rewards.md)
- [Burn and recycle mechanisms](./economics/05-burn-recycle.md)

### Privacy deep-dive
- [Threat model and design goals](./privacy/01-threat-model.md)
- [Shield pool — deposit, transfer, withdraw](./privacy/02-shield-pool.md)
- [ZK circuits and proving system](./privacy/03-zk-circuits.md)
- [Energy settlement for private txs](./privacy/04-energy-settlement.md)

### Reference
- [REST + JSON-RPC API reference](./reference/api-guide.md)
- [Wallet integration handbook](./reference/wallet-integration.md)
- [Node operation guide](./reference/node-operation.md)
- [Glossary](./reference/glossary.md)
- [FAQ](./reference/faq.md)

## Versioning

These docs track the `main` branch of `atoshi-chain`. When the chain releases a major version, this repo is tagged with the matching version. Older docs live in the corresponding tag.

## Contributing

Found something inaccurate, outdated, or unclear? Open a PR — every page has a `Last reviewed:` date in the footer; please update it when you change content. For larger structural changes, file an issue first.

## License

Documentation released under CC-BY-SA 4.0. Code snippets embedded in docs are MIT.
