---
name: committee-prediction-market-skills
display_name: Committee-based Prediction Market Skill
description: >
  Call guide and state-machine conventions for the committee-based prediction market built from
  GouGouBiMarketFactory, GouGouBiMarketProposal, and GouGouBiMarketCondition, including how to
  create proposals and conditions (with `skills` metadata), run activation/settlement/dispute voting,
  execute supreme-committee arbitration, apply committee/supreme penalties, distribute governance
  rewards, and perform CPMM trading and redemption. Use this skill when an Agent needs to handle
  operations like "create proposal", "committee", "condition", "activation vote", "settlement vote",
  "dispute", "supreme committee", "governance reward", "redeem", or "buy YES/NO", and when the next
  call depends on the condition status.
author: frank
version: 0.1.1
language: en
tags:
  - evm
  - prediction-market
  - committee
  - contract-call
  - subgraph
---

# Committee-based prediction market (Agent call conventions)

## 1. Deployment information

- **Network**: BSC mainnet (Binance Smart Chain, `chainId = 56`)
- **Factory contract file**: `GouGouBiMarketFactory.sol`
- **Factory contract address**: `0xa334AcEFc50AE494610678FBf7e2A35Ca05Ff336`
- **Staking token (DOGE / 狗狗币)**: `0xb05678Ed0c9559955559DE864829a0c8AF574444` — DOGE is 狗狗币 (Dogecoin). All proposal/supreme committee stakes and vote locks use this token (18 decimals). Caller must `approve` the Factory (or transfer DOGE) before `proposeMarketCreation` and `stakeForProposalCommittee`.
- **Proposal / Condition contracts**: created as minimal clones by the Factory

Assumptions when calling:
- You are already connected to a BSC mainnet RPC (for example `https://bsc-dataseed.binance.org`)
- Account balance and gas settings are handled by the caller

## 2. When to use this Skill

- User or workflow requests: **create proposal**, **join committee**, **create condition**, **activation vote**, **settlement vote**, **initiate dispute**, **supreme committee arbitration**, **redeem**, **buy YES/NO**, **add liquidity**.
- Next step depends on **condition status**: read **Proposal.conditionStatus(conditionContract)** first, then decide which contract and method to call.
- Writing or debugging scripts, OpenClaw workflows, or frontends that involve call order and preconditions for the three contracts above.
- Subgraphs or indexers that treat **Proposal** as the authoritative source for status and results.
- **Graph queries**: Query indexed data (proposals, conditions, trades, votes, disputes, etc.) via the Subgraph endpoint; see section below «Subgraph (Graph) support».

## 3. Contract paths

| Role   | Contract file                                           | Description |
|--------|---------------------------------------------------------|-------------|
| Factory | `GouGouBiMarketFactory.sol`   | Create proposals, committee/supreme committee staking, dispute arbitration, reward claiming |
| Proposal | `GouGouBiMarketProposal.sol`  | One clone per proposal; condition creation, activation/settlement voting, dispute, state transitions |
| Condition | `GouGouBiMarketCondition.sol` | One clone per condition; trading, liquidity, redeem |

**Convention**: Factory / Proposal / Condition addresses are supplied by upper-layer config (e.g. OpenClaw parameters, environment variables).

---

## 3.1 `skills` field conventions (Proposal / Condition)

- **Proposal.skills (`proposalSkills`)**:  
  Filled via `Factory.proposeMarketCreation(..., skills, ...)`. Recommended as natural‑language or structured hints for **how this overall proposal/market should be queried and settled**, for example:
  - “Use Beijing time; settle by Binance BTCUSDT daily close; call the on‑chain view contract to fetch the latest price before submitting settlement votes.”
  - `{"type":"sports","league":"NBA","source":"https://example.com/api","settlement_rule":"final score only"}`
- **Condition.skills**:  
  Filled via `Proposal.createConditions(..., _rules, _skills, _isNormalized)`. Describes **how to query/execute this specific condition**, for example:
  - "Read oracle XXX with key Y; treat value > 0 as YES."
  - "Scrape the official announcement page and look for keywords 'extend/cancel' to decide a NO outcome."

When Agents create proposals or conditions, if the upper layer already knows which data source / query logic to use, it should prefer encoding that information into `skills`, so downstream Agents can parse it directly to generate scripts or external API calls.

---

## 4. Minimum limits and constants (from contracts)

All stake and lock amounts below are in **DOGE（狗狗币 / Dogecoin）** (18 decimals). Values are taken from the deployed Factory and Proposal contracts.

### 4.1 Factory constants

| Constant | Value | Meaning |
|----------|--------|--------|
| **MIN_STAKE_AMOUNT** | `10 * 1e18` (10 DOGE) | Minimum stake for `proposeMarketCreation(..., stakeAmount)` and minimum per `stakeForProposalCommittee(proposal, amount)` when committee has &lt; 3 members. |
| **COMMITTEE_MIN_SIZE** | `3` | Minimum committee size for **governance thresholds** such as activation voting (2/3 of committee) and some supreme-governance rules; **note**: creating conditions now only requires “caller is a committee member”, not that the committee has ≥ 3 members. |
| **Committee stake when full** | `amount >= avgStake * 2` | When the committee already has ≥ 3 members, a new joiner must stake at least **twice the current average stake** per member. Use `factory.getProposalCommitteeMinStakeToJoin(proposal)` to get `(minStakeToJoin, isFull)`. |
| **SETTLEMENT_BOND_MIN** | `10 * 1e18` (10 DOGE) | Minimum `lockAmount` for `conditionSettlementVoteLock(conditionContract, result, lockAmount)` and `voteOnConditionResult(..., lockAmount)`. |
| **DISPUTE_BOND_MIN** | `10 * 1e18` (10 DOGE) | Minimum `lockAmount` for `initiateDispute(conditionContract, evidence, lockAmount)`. |
| **SUPREME_GOVERNANCE_MIN_LOCK_AMOUNT** | `10 * 1e18` (10 DOGE) | Minimum `lockAmount` for `voteConditionDisputeSupreme(conditionContract, result, lockAmount)`. |
| **SUPREME_MIN_STAKE_AMOUNT** | `100 * 1e18` (100 DOGE) | Minimum stake for `stakeForSupremeCommittee(amount)`. |
| **GOVERNANCE_SIZE** | `21` | Number of supreme committee governance members (top 21 by stake) who can call `voteConditionDisputeSupreme`. |
| **SETTLEMENT_PENALTY_WINDOW_SECONDS** | `600` (10 min) | After `conditionDeadline + 600`, if the committee never voted (no settlement bonds), anyone may call `penalizeProposalCommitteeForNoVote` to slash 10% of each committee member’s stake and extend the vote timeout. |
| **DISPUTER_RATIO_DENOMINATOR / SUPREME_RATIO_NUMERATOR** | `10 / 3` | When disputes lose, 30% of dispute/committee penalties go to supreme governance (pro‑rata by stake), 70% to committee side or disputers as defined in Factory. |

### 4.2 Proposal time windows

| Constant | Value | Meaning |
|----------|--------|--------|
| **SETTLEMENT_VOTE_WINDOW_SECONDS** | `600` (10 min) | Settlement vote is allowed until `conditionDeadline + 600`. After that, if there are votes, status can move to DISPUTED. |
| **DISPUTE_WINDOW_SECONDS** | `600` (10 min) | Dispute can be initiated until `conditionDeadline + SETTLEMENT_VOTE_WINDOW_SECONDS + DISPUTE_WINDOW_SECONDS`. After that, `checkConditionStatus` can set SETTLED or DISPUTED_INITIATED. |
| **Activation threshold** | 2/3 | Condition activation: `voteOnConditionActivation(conditionContract, approve)` — need 2/3 of committee to approve to reach ACTIVE, or 2/3 to reject for REJECTED. |
| **Dispute → DISPUTED_INITIATED** | Total dispute bonds ≥ 1/10 of winning side’s settlement bonds | After the dispute window ends, if `disputeInitiatorTotalBondAmount >= (settlementYesTotalBonds or settlementNoTotalBonds, depending on committee result) / 10`, status becomes DISPUTED_INITIATED; otherwise SETTLED (dispute bonds released). |

### 4.3 Condition fees (GouGouBiMarketCondition)

| Constant | Value | Meaning |
|----------|--------|--------|
| **TRADING_FEE_BPS** | `30` (0.30%) | Buy fee (buyYes / buyNo). |
| **SELL_FEE_BPS** | `100` (1%) | Sell fee (sellYes / sellNo). |
| **COMMITTEE_REDEEM_FEE_BPS** | `100` (1%) | Share of redeem pool to committee winners. |
| **SUPREME_COMMITTEE_REDEEM_FEE_BPS** | `30` (0.30%) | Share of redeem pool to supreme committee. |
| **LpType** | `0` = FEE (稳健), `1` = RISK (风险) | For `addLiquidity(amount, lpType)`. |

---

## Flow overview (steps + decision points)

```
[1] Factory.proposeMarketCreation(...)  → get proposalAddress
[2] Factory.stakeForProposalCommittee(proposalAddress, amount)  → optional, grows committee and stake-weight but no longer required for creating conditions
[3] Proposal.createConditions(...)  → get conditionContract from events/conditionContracts
[4] Proposal.voteOnConditionActivation(conditionContract, approve)  → 2/3 approve → status=ACTIVE(1)
     │
     ▼
[5] If status==ACTIVE and tradeDeadline>now:
    Condition.addLiquidity(amount, lpType) / buyYes / buyNo / sellYes / sellNo / swap*
    If deadline passed: go to [6]
[6] Committee Proposal.conditionSettlementVoteLock or voteOnConditionResult
    Anyone Proposal.checkConditionStatus(conditionContract)
    ├─ Past settlement window with votes → status=DISPUTED(4)
    └─ Still ACTIVE → continue [5] or wait
[7] If status==DISPUTED, within dispute window:
    Proposal.initiateDispute(conditionContract, evidence, lockAmount)
    Anyone checkConditionStatus(conditionContract)
    ├─ Past dispute window, no dispute → SETTLED(7)
    ├─ Past dispute window, dispute stake≥1/10 → DISPUTED_INITIATED(5)
    └─ Otherwise keep waiting
[8] If status==DISPUTED_INITIATED:
    Factory.voteConditionDisputeSupreme(conditionContract, result, lockAmount)
    When satisfied, internally calls Proposal.resolveConditionByFactory → SETTLED(7)
[9] If status==SETTLED:
    Condition.redeem() / Condition.claimLp()
    Claim committee/supreme committee rewards on Factory
```

**Decision points**:  
- Trading/liquidity: `Proposal.conditionStatus(cond)==1` and `Condition.tradeDeadline()>block.timestamp`.  
- Settlement vote window: `conditionDeadline + 600` (10 min).  
- Dispute window: `conditionDeadline + 600 + 600` (20 min after condition deadline).  
- Committee no‑vote penalty: after `conditionDeadline + 600` with **no settlement votes recorded**, anyone may call `Factory.penalizeProposalCommitteeForNoVote(proposal, conditionContract)` to slash committee stakes and extend the vote window.  
- Status authority: always use **Proposal.conditionStatus(conditionContract)**; do not rely on Condition alone.

---

## Condition status (authority on Proposal)

| Enum | Name               | Allowed actions |
|------|--------------------|-----------------|
| 0    | CREATED            | Activation vote only; 2/3 approve→ACTIVE, reject→REJECTED |
| 1    | ACTIVE             | Trading, add/remove LP, committee settlement vote; past settlement window + votes→DISPUTED |
| 2    | REJECTED           | No longer used |
| 4    | DISPUTED           | initiateDispute; after dispute window checkConditionStatus → SETTLED or DISPUTED_INITIATED |
| 5    | DISPUTED_INITIATED | Supreme committee voteConditionDisputeSupreme; when satisfied → SETTLED |
| 7    | SETTLED            | redeem, claimLp; committee/supreme committee claim rewards |

---

## Common Agent tasks (contract.method + constraints)

| Task              | Contract  | Method | Constraints |
|-------------------|----------|--------|-------------|
| Create proposal   | Factory  | proposeMarketCreation(liquidityToken, proposalName, deadline, imageUrl, rules, timezone, language, groupUrl, skills, tags, stakeAmount) | stakeAmount ≥ 10 DOGE (1e19); `skills` is arbitrary metadata string; approve Factory for DOGE first; returns proposalAddress |
| Committee expand  | Factory  | stakeForProposalCommittee(proposal, amount) | amount ≥ 10 DOGE; if committee already ≥3, amount ≥ 2× avg stake (use getProposalCommitteeMinStakeToJoin) |
| Stake supreme committee | Factory | stakeForSupremeCommittee(amount) | amount ≥ 100 DOGE; increases caller’s supreme stake and may put them into top‑21 governance set |
| Claim governance rewards | Factory | claimGovernanceReward() | Anyone with accumulated penalty/committee rewards can call; pays out DOGE and condition‑token rewards, reverts if no reward |
| Penalize inactive committee | Factory | penalizeProposalCommitteeForNoVote(proposal, conditionContract) | After `conditionDeadline + 600`, status==ACTIVE and **no settlement votes**; slashes 10% of each committee member’s stake and distributes to supreme governance; extends settlement window |
| Create condition  | Proposal | createConditions(conditionName, deadline, tradeDeadline, conditionImageUrl, rules, skills, isNormalized) | Caller must be **proposal committee member**; no minimum committee size requirement; `skills` is arbitrary metadata describing the condition |
| Activation vote   | Proposal | voteOnConditionActivation(conditionContract, approve) | status==CREATED; 2/3 approves → ACTIVE, 2/3 rejects → REJECTED |
| Read status       | Proposal | conditionStatus(conditionContract) | Authoritative; read first then decide next call |
| Advance status    | Proposal | checkConditionStatus(conditionContract) | Anyone can call; updates status by time and votes, may transition to DISPUTED / DISPUTED_INITIATED / SETTLED and handle dispute‑bond releases and fee distribution |
| Settlement vote   | Proposal | conditionSettlementVoteLock(conditionContract, result, lockAmount) or voteOnConditionResult(conditionContract, result, evidence, lockAmount) | result: 1=YES, 2=NO; lockAmount ≥ 10 DOGE; within conditionDeadline+600; cannot switch side once bonded on YES/NO |
| Initiate dispute  | Proposal | initiateDispute(conditionContract, evidence, lockAmount) | status==DISPUTED; lockAmount ≥ 10 DOGE; within dispute window (deadline+1200); multiple initiators allowed and summed |
| Supreme committee vote | Factory | voteConditionDisputeSupreme(conditionContract, result, lockAmount) | Caller in top‑21 governance; result 1=YES, 2=NO; lockAmount ≥ 10 DOGE; moves DISPUTED_INITIATED → SETTLED and triggers committee/disputer penalties and rewards |
| Add liquidity     | Condition | addLiquidity(amount, lpType) | status==ACTIVE, tradeDeadline>now; lpType 0=FEE, 1=RISK |
| Buy/sell/swap      | Condition | buyYes(tokenIn, minYesOut)/buyNo/sellYes/sellNo/swapYesForNo/swapNoForYes | status==ACTIVE, tradeDeadline>now |
| Redeem             | Condition | redeem() / claimLp() | status==SETTLED |

---

## Pre-call checklist

- Use **Proposal.conditionStatus(conditionContract)** for status; do not read Condition only.
- Trading/liquidity: **Condition.tradeDeadline() > block.timestamp** and status==ACTIVE.
- Settlement window: `conditionDeadline + 600` (10 min); dispute window: +600 (10 min) after that.
- Minimum amounts: proposal/committee stake ≥ 10 DOGE (1e19 wei); when committee has ≥3 members, new joiner needs ≥ 2× average stake. Settlement/dispute/supreme lock ≥ 10 DOGE each.
- Create condition, activation, settlement vote: **proposal committee members** only; dispute arbitration: **supreme governance members** (top 21 by stake); redeem/claimLp: only after **SETTLED**.
- Staking token: DOGE at `factory.dogeToken()`; approve Factory before proposeMarketCreation and stakeForProposalCommittee.
 - Supreme governance: stake ≥ 100 DOGE via `stakeForSupremeCommittee`, then read top‑21 via `getSupremeCommitteeMembers` and call `claimGovernanceReward` to withdraw penalty and winner rewards when available.

---

## ABI source and common fragments

**Full ABI** (recommended for production/subgraph):  
- Factory: `subgraph/abis/GouGouBiMarketFactory.json`  
- Proposal: `subgraph/abis/GouGouBiMarketProposal.json`  
- Condition: `subgraph/abis/GouGouBiMarketCondition.json`

Below is a **minimal ABI** of commonly used methods and key events for Agents; use with ethers/viem (deduplicate when merging with full ABI).

### GouGouBiMarketFactory (common)

```json
[
  {"type":"function","name":"proposeMarketCreation","inputs":[{"name":"liquidityToken","type":"address"},{"name":"proposalName","type":"string"},{"name":"deadline","type":"uint256"},{"name":"imageUrl","type":"string"},{"name":"rules","type":"string"},{"name":"timezone","type":"string"},{"name":"language","type":"string"},{"name":"groupUrl","type":"string"},{"name":"skills","type":"string"},{"name":"tags","type":"string[]"},{"name":"stakeAmount","type":"uint256"}],"outputs":[{"name":"proposalAddress","type":"address"}],"stateMutability":"nonpayable"},
  {"type":"function","name":"stakeForSupremeCommittee","inputs":[{"name":"amount","type":"uint256"}],"outputs":[],"stateMutability":"nonpayable"},
  {"type":"function","name":"claimGovernanceReward","inputs":[],"outputs":[],"stateMutability":"nonpayable"},
  {"type":"function","name":"penalizeProposalCommitteeForNoVote","inputs":[{"name":"proposal","type":"address"},{"name":"conditionContract","type":"address"}],"outputs":[],"stateMutability":"nonpayable"},
  {"type":"function","name":"stakeForProposalCommittee","inputs":[{"name":"proposal","type":"address"},{"name":"amount","type":"uint256"}],"outputs":[],"stateMutability":"nonpayable"},
  {"type":"function","name":"getProposalCommitteeMembers","inputs":[{"name":"proposal","type":"address"}],"outputs":[{"name":"","type":"address[]"}],"stateMutability":"view"},
  {"type":"function","name":"getSupremeCommitteeMembers","inputs":[],"outputs":[{"name":"","type":"address[]"}],"stateMutability":"view"},
  {"type":"function","name":"voteConditionDisputeSupreme","inputs":[{"name":"conditionContract","type":"address"},{"name":"result","type":"uint8"},{"name":"amount","type":"uint256"}],"outputs":[],"stateMutability":"nonpayable"},
  {"type":"function","name":"MIN_STAKE_AMOUNT","inputs":[],"outputs":[{"type":"uint256"}],"stateMutability":"view"},
  {"type":"function","name":"COMMITTEE_MIN_SIZE","inputs":[],"outputs":[{"type":"uint256"}],"stateMutability":"view"},
  {"type":"function","name":"getProposalCommitteeMinStakeToJoin","inputs":[{"name":"proposal","type":"address"}],"outputs":[{"name":"minStakeToJoin","type":"uint256"},{"name":"isFull","type":"bool"}],"stateMutability":"view"},
  {"type":"function","name":"dogeToken","inputs":[],"outputs":[{"type":"address"}],"stateMutability":"view"},
  {"type":"event","name":"ProposalCreatedWithStake","inputs":[{"indexed":true,"name":"proposer","type":"address"},{"indexed":true,"name":"proposal","type":"address"},{"indexed":false,"name":"stakeAmount","type":"uint256"}]}
]
```

### GouGouBiMarketProposal (common)

```json
[
  {"type":"function","name":"createConditions","inputs":[{"name":"_conditionName","type":"string"},{"name":"_deadline","type":"uint256"},{"name":"_tradeDeadline","type":"uint256"},{"name":"_conditionImageUrl","type":"string"},{"name":"_rules","type":"string"},{"name":"_skills","type":"string"},{"name":"_isNormalized","type":"bool"}],"outputs":[],"stateMutability":"nonpayable"},
  {"type":"function","name":"voteOnConditionActivation","inputs":[{"name":"conditionContract","type":"address"},{"name":"approve","type":"bool"}],"outputs":[],"stateMutability":"nonpayable"},
  {"type":"function","name":"conditionStatus","inputs":[{"name":"conditionContract","type":"address"}],"outputs":[{"name":"","type":"uint8"}],"stateMutability":"view"},
  {"type":"function","name":"conditionResult","inputs":[{"name":"conditionContract","type":"address"}],"outputs":[{"name":"","type":"uint8"}],"stateMutability":"view"},
  {"type":"function","name":"conditionDeadline","inputs":[{"name":"conditionContract","type":"address"}],"outputs":[{"name":"","type":"uint256"}],"stateMutability":"view"},
  {"type":"function","name":"checkConditionStatus","inputs":[{"name":"conditionContract","type":"address"}],"outputs":[],"stateMutability":"nonpayable"},
  {"type":"function","name":"conditionSettlementVoteLock","inputs":[{"name":"conditionContract","type":"address"},{"name":"result","type":"uint8"},{"name":"lockAmount","type":"uint256"}],"outputs":[],"stateMutability":"nonpayable"},
  {"type":"function","name":"voteOnConditionResult","inputs":[{"name":"conditionContract","type":"address"},{"name":"result","type":"uint8"},{"name":"evidence","type":"string"},{"name":"lockAmount","type":"uint256"}],"outputs":[],"stateMutability":"nonpayable"},
  {"type":"function","name":"initiateDispute","inputs":[{"name":"conditionContract","type":"address"},{"name":"evidence","type":"string"},{"name":"lockAmount","type":"uint256"}],"outputs":[],"stateMutability":"payable"},
  {"type":"function","name":"conditionContracts","inputs":[{"name":"","type":"uint256"}],"outputs":[{"name":"","type":"address"}],"stateMutability":"view"},
  {"type":"event","name":"ConditionCreated","inputs":[{"indexed":true,"name":"conditionContract","type":"address"},{"indexed":false,"name":"conditionName","type":"string"},{"indexed":false,"name":"deadline","type":"uint256"},{"indexed":false,"name":"tradeDeadline","type":"uint256"},{"indexed":false,"name":"yesToken","type":"address"},{"indexed":false,"name":"noToken","type":"address"}]},
  {"type":"event","name":"ConditionStatusUpdated","inputs":[{"indexed":true,"name":"conditionContract","type":"address"},{"indexed":false,"name":"status","type":"uint8"}]}
]
```

### GouGouBiMarketCondition (common)

```json
[
  {"type":"function","name":"addLiquidity","inputs":[{"name":"amount","type":"uint256"},{"name":"lpType","type":"uint8"}],"outputs":[],"stateMutability":"payable"},
  {"type":"function","name":"buyYes","inputs":[{"name":"tokenIn","type":"uint256"},{"name":"minYesOut","type":"uint256"}],"outputs":[],"stateMutability":"payable"},
  {"type":"function","name":"buyNo","inputs":[{"name":"tokenIn","type":"uint256"},{"name":"minNoOut","type":"uint256"}],"outputs":[],"stateMutability":"payable"},
  {"type":"function","name":"sellYes","inputs":[{"name":"amountIn","type":"uint256"},{"name":"minAmountOut","type":"uint256"}],"outputs":[],"stateMutability":"nonpayable"},
  {"type":"function","name":"sellNo","inputs":[{"name":"amountIn","type":"uint256"},{"name":"minAmountOut","type":"uint256"}],"outputs":[],"stateMutability":"nonpayable"},
  {"type":"function","name":"swapYesForNo","inputs":[{"name":"amountIn","type":"uint256"},{"name":"minAmountOut","type":"uint256"}],"outputs":[],"stateMutability":"nonpayable"},
  {"type":"function","name":"swapNoForYes","inputs":[{"name":"amountIn","type":"uint256"},{"name":"minAmountOut","type":"uint256"}],"outputs":[],"stateMutability":"nonpayable"},
  {"type":"function","name":"redeem","inputs":[],"outputs":[],"stateMutability":"nonpayable"},
  {"type":"function","name":"claimLp","inputs":[],"outputs":[],"stateMutability":"nonpayable"},
  {"type":"function","name":"tradeDeadline","inputs":[],"outputs":[{"name":"","type":"uint256"}],"stateMutability":"view"},
  {"type":"function","name":"liquidityToken","inputs":[],"outputs":[{"name":"","type":"address"}],"stateMutability":"view"},
  {"type":"function","name":"yesToken","inputs":[],"outputs":[{"name":"","type":"address"}],"stateMutability":"view"},
  {"type":"function","name":"noToken","inputs":[],"outputs":[{"name":"","type":"address"}],"stateMutability":"view"},
  {"type":"event","name":"BuyYes","inputs":[{"indexed":true,"name":"user","type":"address"},{"indexed":false,"name":"tokenIn","type":"uint256"},{"indexed":false,"name":"totalYesOut","type":"uint256"}]},
  {"type":"event","name":"Redeem","inputs":[{"indexed":true,"name":"user","type":"address"},{"indexed":false,"name":"userShare","type":"uint256"},{"indexed":false,"name":"totalShare","type":"uint256"},{"indexed":false,"name":"payout","type":"uint256"}]}
]
```

Usage examples (ethers v6):  
- Read status: `const status = await proposalContract.conditionStatus(conditionAddress)`.  
- Create proposal: `const tx = await factoryContract.proposeMarketCreation(...); const rec = await tx.wait();` get `proposalAddress` from events or return value.  
- After creating condition, get `conditionContract` from event `ConditionCreated` or `proposalContract.conditionContracts(index)`.

---

## Subgraph (Graph) support

For querying indexed on-chain data as an alternative or supplement to direct RPC calls. Currently in **dev** stage.

### Endpoint

| Environment | URL |
|-------------|-----|
| **Dev** | `https://api.goldsky.com/api/public/project_cmh1u6lj9zwam01yi0vum6rww/subgraphs/gougoubi-subgraph/dev/gn` |

### Schema and entities

Entity definitions are in `schema.graphql`. Main entities include:

| Entity | Description |
|--------|-------------|
| **Proposal** | Proposal: id, proposer, name, deadline, conditions, status, tags, skills, etc. |
| **Condition** | Condition: id, proposal, status, deadline, tradeDeadline, yesToken, noToken, trades, votes, disputes, etc. |
| **Trade** | Trade: condition, user, type (BuyYes/BuyNo/SellYes/SellNo/SwapYesForNo/SwapNoForYes), tokenIn, tokenOut, price, etc. |
| **Vote** | Vote: condition, voter, voteType (Activation/Settlement), approve, result, etc. |
| **Dispute** | Dispute: condition, initiator, evidence, bondAmount, etc. |
| **ProposalCommittee** / **ProposalCommitteeMember** | Proposal committee and members |
| **SupremeCommittee** / **GovernanceCommittee** | Supreme committee and governance committee |
| **PriceRecord** / **Candle** | Price records and candlestick data |
| **LiquidityEvent** | Liquidity events |
| **TokenHolder** | Token holders |
| **UserConditionProfit** | User condition profit summary |

### Use cases

- Filter proposals and conditions by status, tags, or language
- Query trade history, price trends, and candlestick data
- Query committee members, votes, and disputes
- Query user positions and profits
- Build frontend lists, charts, and statistics

### Example queries (GraphQL)

```graphql
# Query active proposals
query ActiveProposals {
  proposals(where: { status: "ACTIVE" }, first: 20, orderBy: createdAt, orderDirection: desc) {
    id
    name
    proposer
    deadline
    conditionCount
    tags
    conditions { id name status deadline tradeDeadline }
  }
}

# Query trades for a condition
query ConditionTrades($conditionId: ID!) {
  condition(id: $conditionId) {
    id name status
    trades(first: 50, orderBy: timestamp, orderDirection: desc) {
      id user type tokenIn tokenOut price timestamp
    }
  }
}
```

---

## Relation to other Skills

- **market-configurable-skills**: GouGouBiMarket**Configurable** and its Factory use **oracle auto-settlement** and rounds; that is a different market from this skill’s **committee + dispute + supreme committee** human adjudication. Do not mix contracts or status semantics.
- This skill applies only to **GouGouBiMarketFactory / GouGouBiMarketProposal / GouGouBiMarketCondition**.
