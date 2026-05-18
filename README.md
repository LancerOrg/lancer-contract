# lancer-contracts

> The trust layer of Lancer — Soroban smart contracts that hold client funds, enforce milestone agreements, and execute community arbitration with zero human custody.

---

## What This Is

`lancer-contracts` contains the two Soroban smart contracts that make Lancer fundamentally different from every traditional freelance platform.

On Upwork, Upwork holds your money. Their team decides disputes. Their algorithm flags accounts. Their rules change without notice.

On Lancer, a smart contract holds the money. Community arbitrators decide disputes. Rules are encoded in Rust and deployed to a blockchain nobody controls. This repo is where that trustlessness is built.

Both contracts are written in Rust using the Soroban SDK and deployed on Stellar Testnet for the wave. Mainnet deployment follows production launch.

---

## The Two Contracts

### Contract 1: Milestone Escrow (`milestone_escrow`)

The core trust engine of every Lancer contract.

When a client awards a contract, the `lancer-api` deploys an instance of this contract with the agreed milestones, amounts, client address, and freelancer address encoded at initialization. The client then funds each milestone into the contract — the freelancer can see the funds locked on-chain before starting a single line of work.

When the freelancer submits a milestone and the client approves, the contract releases that milestone's USDC directly to the freelancer's Stellar wallet. No Lancer wallet is involved at any point. No admin can intervene. The release is automatic on approval.

If the client does not approve and raises a dispute, the milestone funds freeze inside the contract. They stay frozen until the arbitration contract sends a resolution signal — then the escrow releases to whoever won.

**The guarantee this gives a freelancer:** If the funds are locked in the contract, they exist. They cannot be moved by the client, by Lancer, or by anyone except the contract's own logic.

---

### Contract 2: Arbitration (`arbitration`)

The community court that resolves disputes between clients and freelancers.

When a dispute is raised on a milestone, the `lancer-api` opens a case in this contract. Three arbitrators are randomly selected from the pool of verified, high-reputation Lancer users. Both parties submit evidence hashes — references to files stored off-chain, making the evidence tamper-evident without bloating the blockchain.

Arbitrators review the case through the Lancer interface and cast their votes on-chain. When the majority vote is recorded, the arbitration contract calls `execute_dispute` on the escrow contract, which releases funds to the winning party automatically.

Arbitrators earn a fee for resolving cases, paid from the dispute activation fees both parties submitted when opening the case. This incentivises thorough, honest arbitration.

**The guarantee this gives both parties:** No Lancer employee decides your dispute. Three independent community members do, their votes are on-chain, and the outcome is executed by code — not by a support ticket that may never be answered.

---

## Tech Stack

| Layer | Technology |
|---|---|
| Language | Rust |
| Smart Contract SDK | Soroban SDK |
| Network | Stellar Testnet → Mainnet at launch |
| Build Tool | Cargo |
| Deployment | Stellar CLI |
| Testing | Soroban native Rust test utilities |
| Explorer | Stellar Expert / Stellar Laboratory |

---

## Project Structure

```
lancer-contracts/
├── contracts/
│   ├── milestone_escrow/
│   │   ├── src/
│   │   │   ├── lib.rs            # contract entry point and interface
│   │   │   ├── storage.rs        # contract state definitions
│   │   │   ├── escrow.rs         # milestone funding and release logic
│   │   │   └── dispute.rs        # dispute freeze and execution logic
│   │   ├── Cargo.toml
│   │   └── README.md
│   └── arbitration/
│       ├── src/
│       │   ├── lib.rs            # contract entry point and interface
│       │   ├── storage.rs        # case state definitions
│       │   ├── arbitration.rs    # case management and voting logic
│       │   ├── selection.rs      # arbitrator random selection logic
│       │   └── resolution.rs     # majority vote and escrow call logic
│       ├── Cargo.toml
│       └── README.md
├── tests/
│   ├── milestone_escrow_test.rs  # full escrow lifecycle tests
│   └── arbitration_test.rs       # dispute and voting tests
├── scripts/
│   ├── deploy_testnet.sh         # deploy both contracts to testnet
│   └── invoke_example.sh         # example Stellar CLI invocations
├── Cargo.toml                    # workspace config
└── .stellar/                     # Stellar CLI network config
```

---

## Contract Interfaces

### Milestone Escrow

```rust
// Deploy a new contract instance for an awarded contract
fn initialize(
    env: Env,
    client: Address,
    freelancer: Address,
    milestone_titles: Vec<String>,
    milestone_amounts: Vec<i128>,   // amounts in USDC stroops
) -> ()

// Client funds a specific milestone into escrow
fn fund_milestone(
    env: Env,
    milestone_index: u32,
    amount: i128,
) -> ()

// Freelancer marks a milestone as submitted for review
fn submit_milestone(
    env: Env,
    milestone_index: u32,
) -> ()

// Client approves submitted milestone — releases USDC to freelancer
fn approve_milestone(
    env: Env,
    milestone_index: u32,
) -> ()

// Either party raises a dispute — freezes milestone funds
fn raise_dispute(
    env: Env,
    milestone_index: u32,
    raised_by: Address,
) -> BytesN<32>     // returns dispute_id

// Called by arbitration contract only — executes final payout
fn execute_dispute_resolution(
    env: Env,
    milestone_index: u32,
    winner: Address,
) -> ()

// Query full contract state
fn get_state(
    env: Env,
) -> ContractState   // milestones, amounts, statuses, dispute flags

// Query a specific milestone
fn get_milestone(
    env: Env,
    milestone_index: u32,
) -> MilestoneState  // amount, status, submitted_at, approved_at
```

### Arbitration

```rust
// Open a new arbitration case
fn open_case(
    env: Env,
    escrow_contract: Address,
    milestone_index: u32,
    claimant: Address,
    respondent: Address,
) -> BytesN<32>      // returns case_id

// Assign three arbitrators to a case
fn assign_arbitrators(
    env: Env,
    case_id: BytesN<32>,
    arbitrators: Vec<Address>,  // randomly selected by lancer-api
) -> ()

// Party submits evidence — IPFS hash of uploaded files
fn submit_evidence(
    env: Env,
    case_id: BytesN<32>,
    party: Address,
    evidence_hash: BytesN<32>,
) -> ()

// Arbitrator casts their vote
fn cast_vote(
    env: Env,
    case_id: BytesN<32>,
    arbitrator: Address,
    vote_for: Address,          // claimant or respondent address
) -> ()

// Finalize case when majority vote reached — triggers escrow payout
fn finalize_case(
    env: Env,
    case_id: BytesN<32>,
) -> ()

// Query case state
fn get_case(
    env: Env,
    case_id: BytesN<32>,
) -> CaseState       // votes, evidence, status, arbitrators, outcome
```

---

## Getting Started

### Prerequisites

Install Rust:
```bash
curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh
```

Add Soroban target:
```bash
rustup target add wasm32-unknown-unknown
```

Install Stellar CLI:
```bash
cargo install --locked stellar-cli --features opt
```

### Build

```bash
# Build all contracts
cargo build --target wasm32-unknown-unknown --release

# Run all tests
cargo test
```

### Deploy to Testnet

```bash
# Fund a testnet account first
stellar keys generate --global deployer --network testnet
stellar keys fund deployer --network testnet

# Deploy both contracts
chmod +x scripts/deploy_testnet.sh
./scripts/deploy_testnet.sh
```

The script outputs two contract IDs. Add them to `lancer-api/.env`:
```env
ESCROW_CONTRACT_ID=<output from deploy script>
ARBITRATION_CONTRACT_ID=<output from deploy script>
```

---

## Security Principles

**No unilateral withdrawal**
Neither contract has any function that allows a single address — including the contract deployer or any Lancer admin — to withdraw funds without the proper state machine having executed. Funds move only on approval or arbitration resolution.

**Arbitration contract is the only dispute executor**
The `execute_dispute_resolution` function on the escrow contract checks that the caller is the deployed arbitration contract address. No other caller can execute a dispute outcome. This is enforced at the contract level, not the API level.

**Evidence is tamper-evident**
Evidence is stored as cryptographic hashes on-chain, not as files. The actual files live off-chain, but their hash is immutable on Stellar. Any modification to a submitted file changes its hash and breaks the chain of evidence.

**All state transitions emit events**
Every fund, submit, approve, dispute, vote, and resolution emits a Stellar contract event. The `lancer-api` streams these events from Horizon to keep its internal database in sync. Full auditability without relying on the API as the source of truth.

**Re-entrancy protection**
All state writes execute before any token transfers throughout both contracts. Transfer-last pattern is enforced to prevent re-entrancy attacks on fund release functions.

**Overflow-safe arithmetic**
All arithmetic uses Soroban's checked math primitives. Silent overflow on fund calculations is impossible.

---

## Test Coverage

Each contract has tests covering the full lifecycle and edge cases:

**Milestone Escrow Tests**
- Happy path: fund → submit → approve → release for all milestones
- Partial approval: some milestones approved, others disputed
- Dispute freeze: funds stay locked until arbitration resolves
- Dispute resolution: claimant wins, respondent wins
- Unauthorized approve: non-client cannot call approve
- Unauthorized resolution: non-arbitration-contract cannot call execute_dispute_resolution
- Double funding: second fund call on already-funded milestone

**Arbitration Tests**
- Happy path: open → assign → evidence → majority vote → finalize → escrow executes
- Tie vote: three arbitrators, two vote one way
- Evidence after deadline: rejected
- Non-arbitrator vote: rejected
- Finalize before majority: rejected
- Arbitrator self-dealing: arbitrator is party in dispute — rejected at assignment

```bash
# Run all tests with output
cargo test -- --nocapture

# Run specific contract tests
cargo test milestone_escrow -- --nocapture
cargo test arbitration -- --nocapture
```

---

## How This Connects to lancer-api

The backend orchestrates contract interactions. The flow:

1. Client awards contract → `lancer-api` calls `milestone_escrow::initialize()` → stores contract address in PostgreSQL
2. Client funds milestone → `lancer-api` calls `milestone_escrow::fund_milestone()` → frontend shows milestone as funded
3. Freelancer submits → `lancer-api` calls `milestone_escrow::submit_milestone()` → client notified
4. Client approves → `lancer-api` calls `milestone_escrow::approve_milestone()` → USDC releases to freelancer instantly
5. Dispute raised → `lancer-api` calls `arbitration::open_case()` → assigns arbitrators → both parties notified
6. Vote cast → `lancer-api` calls `arbitration::cast_vote()` → checks for majority → calls `finalize_case()` if reached
7. Case finalized → `arbitration` contract calls `milestone_escrow::execute_dispute_resolution()` → funds released to winner

The API orchestrates. The contracts enforce.

---

## Contributing

If you are a Rust developer comfortable with Soroban — this is your home in the Lancer wave.

Before writing any code:
- Read the [Soroban SDK docs](https://developers.stellar.org/docs/build/smart-contracts/overview) fully
- Understand both contract interfaces and how they interact
- Read the security principles section above — these are non-negotiable

Contribution rules:
- Every new function needs a test before the PR is opened — no exceptions for contract code
- No admin escape hatches — any function that allows a single address to withdraw funds arbitrarily will not be merged regardless of the justification
- All state transitions must emit events
- Coordinate with the `lancer-api` team before changing any function signature — the backend depends on these interfaces

---

## Resources

- [Soroban Smart Contracts Overview](https://developers.stellar.org/docs/build/smart-contracts/overview)
- [Soroban SDK Reference](https://docs.rs/soroban-sdk/latest/soroban_sdk/)
- [Stellar CLI Reference](https://developers.stellar.org/docs/tools/stellar-cli)
- [Stellar Laboratory — Testnet](https://laboratory.stellar.org)
- [Stellar Expert Explorer](https://stellar.expert)
- [Soroban by Example](https://soroban.stellar.org/docs/basic-tutorials/hello-world)

---

## Related Repos

| Repo | Description |
|---|---|
| [lancer-api](https://github.com/LancerOrg/lancer-api) | Node.js backend — orchestrates all contract calls |
| [lancer-web](https://github.com/LancerOrg/lancer-web) | Next.js frontend — the marketplace interface |

---

## License

MIT — build freely, give credit.

---

*Built in Nigeria. For African talent. On Stellar.*