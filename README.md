# 🗳️ Private Voting DApp — Midnight Network

> Anonymous multi-poll governance with ZK voter identity protection and admin-controlled polls, built on the [Midnight Network](https://midnight.network).

[![Built on Midnight](https://img.shields.io/badge/Built%20on-Midnight%20Network-6C3CE1?style=flat-square)](https://midnight.network)
[![Language](https://img.shields.io/badge/Contract-Compact%200.23-blue?style=flat-square)](https://docs.midnight.network/compact)
[![Network](https://img.shields.io/badge/Network-Preprod%20Testnet-orange?style=flat-square)](https://faucet.preprod.midnight.network)
[![License](https://img.shields.io/badge/License-Apache%202.0-green?style=flat-square)](./LICENSE)

---

## What is this?

A fully privacy-preserving voting application on the Midnight blockchain. Anyone can cast votes on active polls — **what they vote for is visible, but who voted is permanently hidden** using Zero Knowledge (ZK) proofs.

The admin deploys the contract, creates polls, and can close them. Voters connect with their wallet seed, pick an option, and submit a ZK-proven anonymous vote. No identity, no tracking, no leaks.

---

## How the Privacy Works

Midnight's Compact language compiles smart contract logic into ZK circuits. When you vote:

1. Your wallet's `localSecretKey()` is used locally to derive a **voter commitment** — a one-way cryptographic hash
2. That commitment (not your key or address) is stored on-chain to prevent double-voting
3. A ZK proof is generated locally by the proof server, proving you followed the rules without revealing *who* you are
4. The proof is submitted to Preprod — your identity never leaves your machine

```
Voter identity  →  [ZK Circuit]  →  Commitment stored on-chain
(stays private)       (local)        (mathematically unlinkable)
```

---

## Project Structure

```
private-voting-dapp/
├── contracts/
│   └── voting.compact          # Compact smart contract (ZK circuits)
├── src/
│   ├── deploy.ts               # Deploy contract to Preprod
│   ├── cli.ts                  # Interactive CLI (admin + voter)
│   └── voting-api.ts           # Reusable TypeScript API layer
├── frontend/
│   └── index.html              # Single-file HTML/JS UI (no framework)
├── docker-compose.yml          # Proof server config
├── package.json
├── tsconfig.json
└── deployment.json             # Auto-generated after deploy
```

---

## Prerequisites

| Tool | Version | Purpose |
|------|---------|---------|
| [Node.js](https://nodejs.org) | v22+ | Runtime |
| [Docker Desktop](https://www.docker.com/products/docker-desktop/) | Latest | Proof server |
| [Compact compiler](https://docs.midnight.network/getting-started/installation) | Latest | Contract compilation |
| Linux or macOS | — | WSL2 works on Windows |

> ⚠️ **Windows native is not supported.** Use WSL2.

Install the Compact compiler:

```bash
curl --proto '=https' --tlsv1.2 -LsSf \
  https://github.com/midnightntwrk/compact/releases/latest/download/compact-installer.sh | sh

source ~/.bashrc   # or ~/.zshrc
compact --version  # verify
```

---

## Setup

### 1. Clone the repo

```bash
git clone https://github.com/YOUR_USERNAME/private-voting-dapp.git
cd private-voting-dapp
```

### 2. Install dependencies

```bash
npm install
```

### 3. Start the proof server

The proof server generates ZK proofs locally. It must be running before deploying or voting.

```bash
npm run proof-server
# or manually:
docker compose up -d
```

Verify it's healthy:
```bash
curl http://localhost:6300/health
# → {"status":"ok"}
```

### 4. Compile the contract

```bash
npm run compile
```

This outputs compiled artifacts to `contracts/managed/voting/`.

### 5. Deploy to Preprod

```bash
npm run deploy
```

On first run, you'll be prompted to create a new wallet. The CLI will:
- Generate a wallet seed — **save this somewhere safe**
- Display your wallet address
- Wait for you to fund it from the [Preprod faucet](https://faucet.preprod.midnight.network)
- Auto-generate tDUST for transaction fees
- Deploy the contract and save the address to `deployment.json`

---

## Fund Your Wallet

After running `npm run deploy`, copy your **unshielded address** from the output and visit:

👉 **[https://faucet.preprod.midnight.network](https://faucet.preprod.midnight.network)**

Paste your address and request tNIGHT tokens. Wait 1–2 minutes for confirmation, then the deploy script resumes automatically.

---

## Usage

### CLI (Admin + Voter)

```bash
npm run cli
```

**If your wallet is the admin** (the deployer), you'll see:
```
1. Create new poll
2. Close a poll
3. View all polls
4. Exit
```

**All other wallets** see the voter menu:
```
1. View all polls
2. Cast a vote
3. Exit
```

#### Create a poll (admin)
Enter a question and 2–4 options. The poll goes live immediately on-chain.

#### Cast a vote
Select an open poll, pick your option. The CLI:
1. Generates your voter commitment locally
2. Checks you haven't already voted on this poll
3. Creates a ZK proof via the proof server
4. Submits the transaction to Preprod
5. Displays the updated tally

#### Close a poll (admin)
Marks a poll as closed. Votes can no longer be cast on it.

---

### Frontend UI

Open `frontend/index.html` directly in your browser — no build step needed.

```bash
# macOS
open frontend/index.html

# Linux / WSL2
explorer.exe frontend/index.html
```

Enter your contract address and wallet seed from `deployment.json` to connect. The UI polls for updates every 10 seconds and shows live vote tallies.

> 🔒 Your wallet seed is held **in memory only** — it is never stored in the browser or sent anywhere.

---

## Smart Contract Overview

The contract is written in **Compact** (`contracts/voting.compact`), Midnight's ZK smart contract language.

### Public on-chain state
| Field | Type | Description |
|-------|------|-------------|
| `admin` | `Bytes<32>` | Cryptographic commitment of the deployer — gates admin actions |
| `pollCount` | `Counter` | Total polls created |
| `polls` | `Map<Field, PollData>` | All poll data indexed by poll number |

### Each poll stores
- Question text, option labels (up to 4), option vote counts
- `isOpen` — whether voting is active
- A Merkle tree of voter commitments (prevents double-voting without revealing identities)

### Circuits (contract functions)
| Circuit | Who can call | What it does |
|---------|-------------|--------------|
| `createPoll(...)` | Admin only | Creates a new poll with 2–4 options |
| `castVote(pollIndex, optionIndex)` | Anyone | Casts an anonymous vote via ZK proof |
| `closePoll(pollIndex)` | Admin only | Closes a poll permanently |

### Privacy guarantee
The `localSecretKey()` witness is **never disclosed, never sent on-chain, never included in any proof**. It only exists on the voter's local machine during circuit execution.

---

## npm Scripts

| Script | What it does |
|--------|-------------|
| `npm run proof-server` | Starts Docker proof server on port 6300 |
| `npm run compile` | Compiles `voting.compact` → `contracts/managed/voting/` |
| `npm run deploy` | Deploys contract to Preprod, saves `deployment.json` |
| `npm run cli` | Starts interactive CLI |
| `npm run setup` | Does all of the above in sequence |

---

## Troubleshooting

**`compact: command not found`**
Re-source your shell after installing the Compact toolchain:
```bash
source ~/.bashrc
```

**`curl: (7) Failed to connect to localhost port 6300`**
The proof server isn't running. Start it:
```bash
docker compose up -d
```

**`No such file or directory: deployment.json`**
You haven't deployed yet. Run `npm run deploy` first.

**`failed assert: already voted`**
Your wallet has already voted on this poll. Each wallet can vote once per poll.

**`failed assert: board is occupied` / admin assertion fails**
Your wallet seed doesn't match the admin. Only the deployer wallet can create and close polls.

**WSL2: file not found errors with Windows paths**
Keep your project inside WSL's filesystem (`~/projects/...`), not on `/mnt/c/...`. Cross-filesystem I/O is slow and can cause path issues.

---

## Architecture Diagram

```
┌─────────────────────────────────────────────┐
│              Frontend (index.html)           │
│         Vanilla JS — no framework            │
└──────────────────┬──────────────────────────┘
                   │ calls
┌──────────────────▼──────────────────────────┐
│           voting-api.ts (TypeScript)         │
│    deploy · vote · createPoll · getPolls     │
└────────┬────────────────────┬───────────────┘
         │                    │
┌────────▼────────┐  ┌────────▼────────────────┐
│  Proof Server   │  │   Midnight Preprod        │
│  localhost:6300 │  │   (Indexer + Node)        │
│  (Docker)       │  │                           │
│  Generates ZK   │  │  voting.compact deployed  │
│  proofs locally │  │  Ledger state on-chain    │
└─────────────────┘  └───────────────────────────┘
```

---

## Contributing

PRs welcome. If you find a bug or want to extend the contract (e.g. time-locked polls, token-gated voting), open an issue first so we can discuss the approach.

Before submitting, make sure:
- The contract compiles: `npm run compile`
- At least one vote transaction works end-to-end on Preprod
- No wallet seeds or private keys are committed to the repo

---

## Resources

- [Midnight Docs](https://docs.midnight.network)
- [Compact Language Reference](https://docs.midnight.network/compact)
- [Bulletin Board Example](https://docs.midnight.network/examples/dapps/bboard) — the ZK pattern this project builds on
- [Preprod Faucet](https://faucet.preprod.midnight.network)
- [Midnight Discord](https://discord.com/invite/midnightnetwork)

---

## License

Apache 2.0 — see [LICENSE](./LICENSE)

---

*Built on [Midnight Network](https://midnight.network) — programmable privacy for the real world.*  