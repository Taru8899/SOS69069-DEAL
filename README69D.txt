```markdown
# SOS69069 DEAL (69D)

Reputation-based deal confirmation on top of SOS69069 PRESENCE (69P).

**Name:** SOS69069 DEAL  
**Symbol:** 69D  
**Network:** Ethereum Mainnet  
**Compiler:** 0.8.36

---

## Purpose

SOS69069 DEAL records the lifecycle of a deal that starts from a Presence entry:

1. Accept an offer  
2. Record each party’s part  
3. Confirm the deal  

On confirmation, the contract mints **1 SOS** to the original Presence poster.

The contract does not hold funds. Settlement of ETH, goods, or services happens outside the contract. Reputation is the enforcement layer.

---

## Dependencies

| Contract | Role |
|----------|------|
| SOS (`0x61af906f53Eb927790055AC8eA99916a01873c15`) | Mints 1 SOS via `pushTo` |
| SOS69069 PRESENCE (69P) | Source of offer data via `actionOf[poster][typeId]` |

Presence address is set once in the constructor (not hardcoded).

---

## Core flow

### 1. Accept offer

```solidity
acceptOffer(address poster, uint8 typeId)
```

- `poster` — address that posted the Presence  
- `typeId` — type in range **1–9**  

Requirements:

- `poster != msg.sender`
- `actionOf[poster][typeId]` is non-empty
- No active deal already exists for that `(poster, typeId)`

Effect:

- Creates a new deal (`status = Open`)
- Snapshots the action string at accept time
- Increments `accepts[msg.sender]`
- Emits `DealAccepted`

### 2. Do my part

```solidity
doMyPart(uint256 dealId, bytes32 externalTxHash)
```

Callable only by the poster or the seeker of that deal.

- Stores `externalTxHash` (optional proof; may be `bytes32(0)`)
- Sets `status = PartDone`
- Increments `partsDone[msg.sender]`
- Emits `PartDone`

### 3. Confirm deal

```solidity
confirmDeal(uint256 dealId)
```

Callable only by the opposite party after the other side has recorded their part.

- Poster may confirm only if seeker has called `doMyPart`
- Seeker may confirm only if poster has called `doMyPart`
- Sets `status = Confirmed`
- Increments `confirmations[msg.sender]`
- Calls `sos.pushTo(poster)` → mints **1 SOS** to the Presence poster
- Clears the active slot for `(poster, typeId)`
- Emits `DealConfirmed`

### 4. Cancel (optional)

```solidity
cancelDeal(uint256 dealId)
```

- Only while `status == Open`
- Only poster or seeker
- Sets `status = Cancelled`
- Clears the active slot

---

## Deal status

| Status | Meaning |
|--------|---------|
| Open | Accepted; waiting for parts |
| PartDone | At least one party recorded `doMyPart` |
| Confirmed | Counterparty confirmed; 1 SOS minted to poster |
| Cancelled | Cancelled while still Open |

---

## Reputation

Per address:

| Field | Meaning |
|-------|---------|
| `accepts` | Number of offers accepted |
| `partsDone` | Number of `doMyPart` calls |
| `confirmations` | Number of deals confirmed |

Score:

```text
reputationScore = 2 × confirmations − accepts
```

A score near zero or positive means accepts and confirmations are balanced.

---

## Storage model

- Deals are sequential: `nextDealId` starts at 1  
- One active deal per `(poster, typeId)` at a time  
- On confirm or cancel, that slot is freed  
- `getDeal(dealId)` returns the full deal struct  
- `getReputation(user)` and `reputationScore(user)` are view helpers  

---

## What this contract does not do

- Does not transfer ETH or tokens between parties  
- Does not escrow funds  
- Does not verify that an external tx hash is valid  
- Does not change Presence data  

It only records acceptance, parts, confirmation, and reputation, and mints 1 SOS to the poster on successful confirm.

---

## Deploy

Constructor argument:

```text
presence_ = <SOS69069 PRESENCE address>
```

Example (first 69P deployment):

```text
0x40c24f56E5a492b3EB26c287E5cA2604BD29E1BF
```

No ETH is accepted. `receive` and `fallback` revert.
```