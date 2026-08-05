# NFT Minting Platform

A full-stack, multi-chain-ready NFT minting platform built end to end: a gas-optimized Solidity contract, a Rust indexer with wallet-based auth, a TheGraph subgraph, and a Next.js frontend, all wired together and running against a live, verified testnet deployment.

This repo is the front door. It doesn't contain application code, it orchestrates the four services below and documents how they fit together.

**Live contract:** [`0x1D24FE1860F4E670aFd65C1B93118A4B4F5c0f54`](https://sepolia.etherscan.io/address/0x1d24fe1860f4e670afd65c1b93118a4b4f5c0f54) on Sepolia, verified

---

## The four repos

| Repo | What it is | Stack |
|---|---|---|
| [`nft-mint-platform`](https://github.com/sivajisj/nft-mint-platform) | ERC-721A minting contract: allowlist, royalties, reveal, reentrancy guards | Solidity, Foundry, OpenZeppelin |
| [`nft-indexer`](https://github.com/sivajisj/nft-indexer) | Async backend that watches the chain, indexes events, serves the API, handles wallet auth | Rust, Axum, SQLx, Postgres, alloy |
| [`nft-minting-subgraph`](https://github.com/sivajisj/nft-minting-subgraph) | TheGraph subgraph for public, GraphQL-queryable event data | AssemblyScript, TheGraph |
| [`nft-frontend`](https://github.com/sivajisj/nft-frontend) | The actual minting app, wallet connect, live mint, owned-tokens gallery, sign-in | Next.js, wagmi, RainbowKit, TypeScript |

---

## Why it's shaped like this

Four services, four repos, on purpose. Each one has a different toolchain, a different release cadence, and a different reason to exist:

- The **contract** rarely changes after deployment. It's the source of truth others read from.
- The **indexer** needs to run continuously, forever, watching new blocks. It's a long-lived service, not a script.
- The **subgraph** is a second, independent way to query the same on-chain data, publicly, without trusting my backend.
- The **frontend** changes the most often and deploys independently of everything else.

Keeping them separate means each repo's CI only tests what's relevant to it, and each can be understood on its own without wading through unrelated code. The tradeoff is real too: changes that span two services (say, a new contract event the indexer needs to know about) require two PRs instead of one atomic commit. For a project this size, the separation was worth it.

## How the pieces actually talk to each other

```
                     ┌─────────────────────┐
                     │   Sepolia testnet    │
                     │  NFTMintingPlatform   │
                     │   (ERC-721A, verified)│
                     └──────────┬───────────┘
                                │ emits Transfer events
                 ┌──────────────┴───────────────┐
                 ▼                               ▼
      ┌────────────────────┐          ┌──────────────────────┐
      │   nft-indexer       │          │  nft-minting-subgraph │
      │  Rust / Axum         │          │  TheGraph (local)      │
      │  - watches events     │          │  - schema + mappings    │
      │  - 12-block confirm    │          │  - GraphQL endpoint       │
      │  - Postgres storage      │          └──────────────────────┘
      │  - SIWE auth               │
      │  - REST API                   │
      └──────────────┬────────────────┘
                      │ HTTP / JSON
                      ▼
           ┌────────────────────────┐
           │     nft-frontend          │
           │  Next.js + wagmi            │
           │  - connect wallet              │
           │  - live mint tx                  │
           │  - owned tokens gallery            │
           │  - sign the ledger (SIWE)             │
           └────────────────────────────────────────┘
```

The indexer and the subgraph are two independent, parallel ways of reading the same on-chain truth, one gives me a real backend to build auth and custom queries on, the other gives anyone a trustless, public GraphQL endpoint without needing to run my server at all.

## Real, verifiable artifacts

Not claims, links you can click:

- **Contract on Etherscan:** [sepolia.etherscan.io/address/0x1d24...c0f54](https://sepolia.etherscan.io/address/0x1d24fe1860f4e670afd65c1b93118a4b4f5c0f54), source code, verified, readable
- **CI pipeline:** [nft-indexer/actions](https://github.com/sivajisj/nft-indexer/actions), green on every push, fmt + clippy + build/test
- **128,000 randomized mint calls, zero invariant violations**, Foundry invariant testing against the deployed contract's logic

## Running the whole stack locally

You need all four repos cloned as siblings:

```bash
mkdir nft && cd nft
git clone https://github.com/sivajisj/nft-mint-platform.git
git clone https://github.com/sivajisj/nft-indexer.git
git clone https://github.com/sivajisj/nft-minting-subgraph.git
git clone https://github.com/sivajisj/nft-frontend.git
git clone https://github.com/sivajisj/nft-infra.git
cd nft-infra
```

### 1. Backend: Postgres + indexer, via Docker

```bash
cp .env.example .env
# fill in RPC_URL (an Alchemy/Infura Sepolia endpoint) and POSTGRES_PASSWORD
docker compose up --build
```

This starts Postgres (with a healthcheck gate) and the indexer, which begins watching Sepolia from the contract's deployment block and exposes the API on `:4000`.

### 2. Frontend

See [`nft-frontend`'s README](https://github.com/sivajisj/nft-frontend#readme) for the full setup, in short: `npm install`, fill in `.env.local`, `npm run dev`.

### `.env` for this repo (docker-compose)

```
POSTGRES_PASSWORD=devpassword
RPC_URL=https://eth-sepolia.g.alchemy.com/v2/your-key-here
```

## Engineering decisions worth knowing about

**Why ERC-721A over standard ERC-721.** Batch minting writes ownership once per transaction instead of once per token, a real ~40% gas reduction on multi-token mints. The tradeoff: `ownerOf()` lookups walk backward through unwritten slots, so individual transfers cost slightly more. For a collection where minting happens in batches and transfers happen one at a time, that's the right trade.

**Why Rust for the indexer instead of a Node.js/TypeScript indexer.** Partly a skills decision, partly a real one: an async, typed backend with SQLx's compile-time-checked queries catches a whole class of bugs (wrong column type, malformed query) before the code ever runs, not at 3am when a query silently returns the wrong shape.

**Why both a custom indexer and a subgraph.** They're not redundant. The subgraph is trustless and public, anyone can query it without trusting my server. The indexer is where custom logic lives, wallet auth, confirmation-depth gating, anything that isn't "just show me the event log."

**Why 12-block confirmation depth, not more, not less.** It's a risk/UX tradeoff, not a fixed rule. 12 blocks (~24s on a fast chain) is enough that spontaneous reorgs are vanishingly unlikely, while still keeping the wait tolerable for a mint confirmation. A high-value DeFi protocol would reasonably push that number much higher.

**Why SIWE instead of email/password.** The wallet already proves identity cryptographically. Building a second identity system on top would be redundant and worse, it would mean storing and protecting credentials I don't actually need.

**A known, honest gap.** No third-party security audit, that costs real money a solo project doesn't have. What's there instead: Slither + manual review, 128k-call invariant fuzzing, and a documented list of what a real audit would additionally check. Said plainly in an interview, this is a stronger answer than pretending the gap doesn't exist.

## About

Built by [Sivaji Gadidala](https://sivajibuilds.netlify.app) as a from-scratch, self-directed deep dive into Rust systems programming and Solidity security, not a tutorial clone. Every piece, the reentrancy guard, the confirmation-depth logic, the SIWE flow, the ERC-721A gas tradeoff, was built with an explanation of *why* it works, not just that it does.

- Email: [sivajigsivajig703@gmail.com](mailto:sivajigsivajig703@gmail.com)
- GitHub: [github.com/sivajisj](https://github.com/sivajisj)
- LinkedIn: [linkedin.com/in/sivaji-gadidala-b712ba221](https://linkedin.com/in/sivaji-gadidala-b712ba221)
- Portfolio: [sivajibuilds.netlify.app](https://sivajibuilds.netlify.app)
