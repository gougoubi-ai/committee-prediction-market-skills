---
name: committee-market-skills
description: Committee-based prediction market for GouGouBiMarketFactory, GouGouBiMarketProposal, and GouGouBiMarketCondition: call conventions and state machine for creating proposals and conditions, activation/settlement/dispute voting, and CPMM trading and redemption. For use by OpenClaw, scripts, or workflows when calling these contracts — load this skill when the user or task involves "create proposal", "committee", "condition", "activation vote", "settlement vote", "dispute", "supreme committee", "redeem", "buy YES/NO", or when the next call depends on condition status.
---

# Committee-based prediction market (Agent call conventions)

## When to use this Skill

- User or workflow requests: **create proposal**, **join committee**, **create condition**, **activation vote**, **settlement vote**, **initiate dispute**, **supreme committee arbitration**, **redeem**, **buy YES/NO**, **add liquidity**.
- Next step depends on **condition status**: read **Proposal.conditionStatus(conditionContract)** first, then decide which contract and method to call.
- Writing or debugging scripts, OpenClaw workflows, or frontends that involve call order and preconditions for the three contracts above.
- Subgraphs or indexers that treat **Proposal** as the authoritative source for status and results.

**Contract paths**:

| Role   | Contract file                                           | Description |
|--------|---------------------------------------------------------|-------------|
| Factory | `contracts/contracts/GouGouBiMarketFactory.sol`        | Create proposals, committee/supreme committee staking, dispute arbitration, reward claiming |
| Proposal | `contracts/contracts/GouGouBiMarketProposal.sol`        | One clone per proposal; condition creation, activation/settlement voting, dispute, state transitions |
| Condition | `contracts/contracts/GouGouBiMarketCondition.sol`      | One clone per condition; trading, liquidity, redeem |

**Convention**: Factory / Proposal / Condition addresses are supplied by upper-layer config (e.g. OpenClaw parameters, environment variables).

---

## Flow overview (steps + decision points)

```
[1] Factory.proposeMarketCreation(...)  → get proposalAddress
[2] Factory.stakeForProposalCommittee(proposalAddress, amount)  → at least 3 members
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
- Settlement vote window: `conditionDeadline + SETTLEMENT_VOTE_WINDOW_SECONDS`.  
- Dispute window: above + `DISPUTE_WINDOW_SECONDS`.  
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
| Create proposal   | Factory  | proposeMarketCreation(liquidityToken, proposalName, deadline, imageUrl, rules, timezone, language, groupUrl, tags, stakeAmount) | stakeAmount≥MIN_STAKE_AMOUNT; returns proposalAddress |
| Committee expand  | Factory  | stakeForProposalCommittee(proposal, amount) | At least 3 members before creating conditions |
| Create condition  | Proposal | createConditions(conditionName, deadline, tradeDeadline, conditionImageUrl, rules, isNormalized) | Caller must be proposal committee member; committee≥3 |
| Activation vote   | Proposal | voteOnConditionActivation(conditionContract, approve) | status==CREATED; 2/3 decides result |
| Read status       | Proposal | conditionStatus(conditionContract) | Authoritative; read first then decide next call |
| Advance status    | Proposal | checkConditionStatus(conditionContract) | Anyone can call; updates status by time and votes |
| Settlement vote   | Proposal | conditionSettlementVoteLock(conditionContract, result, lockAmount) or voteOnConditionResult(conditionContract, result, evidence, lockAmount) | result: 1=YES, 2=NO; within settlement window |
| Initiate dispute  | Proposal | initiateDispute(conditionContract, evidence, lockAmount) | status==DISPUTED and within dispute window |
| Supreme committee vote | Factory | voteConditionDisputeSupreme(conditionContract, result, lockAmount) | When satisfied, internally → SETTLED |
| Add liquidity     | Condition | addLiquidity(amount, lpType) | status==ACTIVE, tradeDeadline>now; lpType 0=FEE 1=RISK |
| Buy/sell/swap      | Condition | buyYes(tokenIn, minYesOut)/buyNo/sellYes/sellNo/swapYesForNo/swapNoForYes | status==ACTIVE, tradeDeadline>now |
| Redeem             | Condition | redeem() / claimLp() | status==SETTLED |

---

## Pre-call checklist

- Use **Proposal.conditionStatus(conditionContract)** for status; do not read Condition only.
- Trading/liquidity: **Condition.tradeDeadline() > block.timestamp** and status==ACTIVE.
- Settlement window: deadline + SETTLEMENT_VOTE_WINDOW_SECONDS; dispute window: that + DISPUTE_WINDOW_SECONDS.
- Create condition, activation, settlement vote: **proposal committee members** only; dispute arbitration: **supreme committee**; redeem/claimLp: only after **SETTLED**.

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
  {"type":"function","name":"proposeMarketCreation","inputs":[{"name":"liquidityToken","type":"address"},{"name":"proposalName","type":"string"},{"name":"deadline","type":"uint256"},{"name":"imageUrl","type":"string"},{"name":"rules","type":"string"},{"name":"timezone","type":"string"},{"name":"language","type":"string"},{"name":"groupUrl","type":"string"},{"name":"tags","type":"string[]"},{"name":"stakeAmount","type":"uint256"}],"outputs":[{"name":"proposalAddress","type":"address"}],"stateMutability":"nonpayable"},
  {"type":"function","name":"stakeForProposalCommittee","inputs":[{"name":"proposal","type":"address"},{"name":"amount","type":"uint256"}],"outputs":[],"stateMutability":"nonpayable"},
  {"type":"function","name":"getProposalCommitteeMembers","inputs":[{"name":"proposal","type":"address"}],"outputs":[{"name":"","type":"address[]"}],"stateMutability":"view"},
  {"type":"function","name":"voteConditionDisputeSupreme","inputs":[{"name":"conditionContract","type":"address"},{"name":"result","type":"uint8"},{"name":"amount","type":"uint256"}],"outputs":[],"stateMutability":"nonpayable"},
  {"type":"function","name":"MIN_STAKE_AMOUNT","inputs":[],"outputs":[{"type":"uint256"}],"stateMutability":"view"},
  {"type":"function","name":"COMMITTEE_MIN_SIZE","inputs":[],"outputs":[{"type":"uint256"}],"stateMutability":"view"},
  {"type":"event","name":"ProposalCreatedWithStake","inputs":[{"indexed":true,"name":"proposer","type":"address"},{"indexed":true,"name":"proposal","type":"address"},{"indexed":false,"name":"stakeAmount","type":"uint256"}]}
]
```

### GouGouBiMarketProposal (common)

```json
[
  {"type":"function","name":"createConditions","inputs":[{"name":"_conditionName","type":"string"},{"name":"_deadline","type":"uint256"},{"name":"_tradeDeadline","type":"uint256"},{"name":"_conditionImageUrl","type":"string"},{"name":"_rules","type":"string"},{"name":"_isNormalized","type":"bool"}],"outputs":[],"stateMutability":"nonpayable"},
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

## Relation to other Skills

- **market-configurable-skills**: GouGouBiMarket**Configurable** and its Factory use **oracle auto-settlement** and rounds; that is a different market from this skill’s **committee + dispute + supreme committee** human adjudication. Do not mix contracts or status semantics.
- This skill applies only to **GouGouBiMarketFactory / GouGouBiMarketProposal / GouGouBiMarketCondition**.
