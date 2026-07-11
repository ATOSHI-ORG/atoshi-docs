# Staking — validator economics

Atoshi L1 is a Cosmos SDK **Proof-of-Stake** network secured by a capped
validator set running CometBFT consensus. This chapter covers who secures
the chain, how they are paid, and the staking parameters. The numerical
reward model (block reward, halving, the locked 80%) lives in
[Block rewards](../economics/04-block-rewards.md); the release mechanics
are in [Release schedule](../economics/02-release-schedule.md).

## Participant tiers

Participation scales from a full validator down to a phone:

- **Validators** — propose and sign blocks, run consensus. Earn the liquid 20% (commission + their own stake share) and accrue the locked 80% by voting power. Active set capped at 100.
- **Delegators** — bond ATOS to a validator and share in rewards minus commission. No hardware, any amount.
- **Full nodes** — replicate complete chain state and serve queries to wallets and apps. No stake required. (See [Node operation](../reference/node-operation.md).)
- **Light / mobile clients** — the app-layer Power-of-Device engagement tier; the on-ramp for the wider community. This is an ecosystem-product concept, distinct from L1 consensus.

![Tiered validator and staking model, and how validators are paid](../assets/whitepaper/validator-staking-tiers.png)

## Staking parameters

The network launches with conservative, standard Cosmos staking parameters. Final values are confirmed by governance before genesis.

| Parameter | Value |
|---|---|
| Community tax | 2% (funds the on-chain treasury) |
| Default commission | 10% (5% suggested minimum) |
| Unbonding period | 21 days |
| Maximum active validators | 100 |

The **2% community tax** is skimmed from the immediate block-reward stream before distribution and funds the governance-controlled treasury. The **21-day unbonding** period is the standard Cosmos safety window during which unbonding stake is still slashable and cannot be transferred.

## Validator reward: two ATOS streams

Every block mints **19,819 ATOS**, split into two streams (see [Block rewards](../economics/04-block-rewards.md) for the full model):

- **Immediate — 20% (3,963.8 ATOS/block).** Sent to the fee collector, reduced by the 2% community tax, then paid via the Cosmos SDK `x/distribution` module to validators (commission) and their delegators (stake share). Liquid and withdrawable.
- **Locked — 80% (15,855.2 ATOS/block).** Allocated to validators in proportion to voting power, recorded as `MinerLockedBalance.locked_amount`. **Not** paid to delegators. Becomes spendable only when a conditional tier release fires — tying validator income to sustained long-term value rather than short-term selling.

![Where each block reward goes and what a validator earns](../assets/whitepaper/validator-economics.png)

The benchmark figures in the diagram (self-stake, network stake, ROI) are an illustration of how the model behaves under stated assumptions, not a promised return.

## Slashing and liveness

Validators must stay online and signing. Missing too many blocks in the signing window leads to jailing; recover with `atoshid tx slashing unjail` once the downtime window clears. Double-signing (equivocation) is penalized more severely. Delegation collateral bonded to a slashed validator shares the penalty pro-rata; energy-delegation collateral held in the energy module is bank-isolated and not exposed to staking slashing (see [Energy](./01-energy.md)).

## Related

- [Block rewards](../economics/04-block-rewards.md) — the full reward/halving model
- [Release schedule](../economics/02-release-schedule.md) — when locked ATOS unlocks
- [Node operation](../reference/node-operation.md) — running a validator or full node
- [Governance](./09-governance.md) — how staking params are changed

---

*Last reviewed: 2026-07-12*
