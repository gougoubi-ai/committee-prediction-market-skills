# committee-prediction-market-skills

`committee-prediction-market-skills` is a committee-based prediction market contract skill repository for AI Agents, built around **GouGouBiMarketFactory**, **GouGouBiMarketProposal**, and **GouGouBiMarketCondition**. It provides unified call conventions and state-machine documentation so Agents can safely and reliably create proposals and conditions, run activation/settlement/dispute voting, and perform CPMM trading and redemption under natural-language instructions.

---

## About this repository

This repository contains the Skill definition and documentation for the **committee-based prediction market contracts**, with these goals:

- **Unified specification**: define contract addresses, core method signatures, parameter meanings, and return-value descriptions.
- **Lower integration barrier**: provide reusable call templates for scripts, frontends, and workflows (e.g. OpenClaw).
- **Serve Agents**: enable various AI Agents (regardless of framework) to use the committee-based prediction market with minimal configuration.

Core descriptions and usage conventions are in `SKILLS.md`; read it fully before integrating.

---

## Use cases

Use this Skill when you need to perform the following on the **committee-based prediction market**:

- **Create proposal**: submit a market-creation proposal and stake via the Factory.
- **Committee and conditions**: join a proposal committee, create conditions, vote on condition activation.
- **Settlement and dispute**: committee settlement votes, initiate disputes, supreme committee arbitration.
- **Trading and redemption**: buy/sell YES/NO and add liquidity when a condition is ACTIVE; redeem and claim LP after the condition is SETTLED.

Whether you use scripts (Node.js/TS), a frontend DApp, or a workflow engine (e.g. OpenClaw), follow the conventions in `SKILLS.md` and use libraries such as `ethers` / `viem` / `web3` to interact with the three contracts above.

---

## Directory overview

- **`contracts/contracts/GouGouBiMarketFactory.sol`**: factory contract — create proposals, committee/supreme committee staking, dispute arbitration, reward claiming.
- **`contracts/contracts/GouGouBiMarketProposal.sol`**: proposal contract (one clone per proposal) — condition creation, activation/settlement voting, dispute handling, state transitions.
- **`contracts/contracts/GouGouBiMarketCondition.sol`**: condition contract (one clone per condition) — trading, liquidity, redemption.
- **`SKILLS.md`**: Skill metadata, call conventions, and examples (main entry document for AI Agent integration).
- Other scripts or config files: added as needed (e.g. deployment scripts, test examples).

The exact file layout may change over time; refer to the latest contents in the repository.

---

## Contribution guide

Contributions and PRs to improve this Skill are welcome, including but not limited to:

- Adding or correcting documentation (parameter descriptions, code examples, edge cases).
- Adding examples in different languages/frameworks (e.g. Python, Go, different frontend stacks).
- Improving test cases or local development scripts.

Recommended workflow:

1. **Create a branch**
   ```bash
   git checkout -b feature/<your-change>
   ```
2. **Complete your changes and pass local tests.**
3. **Open a Pull Request**, explaining the motivation and impact of your changes.

---

## Disclaimer

This repository and its Skill configurations, contract examples, and documentation are for technical research and reference only.  
They do **not** constitute any investment, financial, or trading advice, nor any guarantee of returns. On-chain operations carry significant risk, including but not limited to contract risk, private key/mnemonic exposure, and market volatility.

Before using any content from this repository with real funds, you should:

- Fully understand the contract logic and call risks;
- Verify the flow with small amounts and in test environments;
- Make prudent decisions based on your own situation and bear all consequences yourself.

The maintainers of this repository do not assume legal responsibility for any loss or liability arising from the use of this repository.
