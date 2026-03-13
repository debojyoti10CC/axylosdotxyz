# AgentDNS — Full Technical Documentation

**Project**: AgentDNS — Agent Mesh + OpenClaw + HTTP 402 (x402) + Elsa execution

**Short description**: AgentDNS is a decentralized agent economy prototype that lets autonomous AI agents discover peers over a UDP P2P mesh, negotiate services, pay for API calls using HTTP 402 micropayments (x402), and settle agreements on-chain using the Elsa execution stack.

**Primary stack**: Node.js, Express, ethers.js, axios, UDP sockets, OpenClaw integration (mocked LLM), Elsa X402 client.

**Related projects/services referenced once**: entity["software","HeyElsa","AI crypto agent platform"] — execution/payment layer used for on-chain settlement and x402 requests.

---

## Table of Contents

1. Overview
2. Goals & Use Cases
3. High-Level Architecture
4. Components (Detailed)

   * Elsa X402 Payment Client
   * P2P Mesh & Agent DNS
   * Negotiation Protocol (NEGO state machine)
   * OpenClaw Agent Integration
   * REST API & CLI
5. Installation & Local Setup
6. Environment Variables
7. Running the System (Examples)
8. API Reference (endpoints)
9. Negotiation & Payment Flow (sequence diagrams)
10. Security Considerations
11. Demo Plan & Suggested Scripts (for hackathons like entity["event","ETHMumbai","Ethereum hackathon in Mumbai"])
12. Testing & Troubleshooting
13. Roadmap & Future Enhancements
14. Contributing
15. Appendix: File map & quick references

---

## 1. Overview

AgentDNS is a proof-of-concept platform for the emerging agent economy. Autonomous agents (servers) advertise capabilities, discover other agents through a UDP-based mesh, and perform peer-to-peer economic negotiations using a deterministic negotiation protocol. When a buyer and seller agree, payment is requested via HTTP 402 micropayment challenges (x402). The Elsa X402 client signs and sends payment headers; when payment is authorized, the agreed action (for example: a token swap) is executed and optionally settled on-chain.

This repository splits responsibilities between three layers:

* **Agent Reasoning Layer** (OpenClaw agent logic) — decides whether to accept/counter offers, when to call tools, and how to respond to natural language commands. In the current codebase this layer is intentionally mocked for deterministic demos.
* **Networking & Negotiation Layer** (Agent DNS + UDP mesh) — peer discovery and negotiation state machine.
* **Execution & Settlement Layer** (Elsa X402 client + contract/DEX execution) — payment signing, x402 API calls, and optional on-chain transactions.

---

## 2. Goals & Use Cases

**Primary goals**

* Demonstrate a working agent-to-agent economic flow (discover → negotiate → pay → settle).
* Provide robust payment plumbing (ECDSA-signed x402 headers, session accounting, budget enforcement).
* Keep demos stable by mocking LLM components while integrating live x402 endpoints.

**Target use cases**

* Autonomous DeFi trading bots negotiating execution fees with service providers.
* Machine-to-machine API marketplaces where AI agents purchase data/compute per call.
* A hackathon demo that clearly shows agent discovery, negotiation, micropayment, and on-chain settlement.

---

## 3. High-Level Architecture

```
User / CLI / Frontend
       ↓
OpenClaw Agent (decision loop, mocked LLM)
       ↓
Agent DNS Mesh (UDP P2P) — discovery & negotiation
       ↓
Negotiation Protocol (NEGO_OFFER → COUNTER → NEGO_ACCEPT → PAYMENT_402_REQUIRED → PAYMENT_402_COMPLETED)
       ↓
Elsa X402 Payment Client (ECDSA payments, session & budget management)
       ↓
x402 API (https://x402-api.heyelsa.ai) → DEX / DeFi Protocols → Blockchain (optional on-chain settlement)
```

Key notes:

* The system supports running multiple agents on the same LAN (two-laptop demo) or across machines.
* Negotiation is deterministic and logged; OpenClaw integration provides the hook for upgrading decision logic to an LLM later.

---

## 4. Components (Detailed)

### 4.1 Elsa X402 Payment Client (`lib/ElsaX402PaymentClient.js`)

**Status**: READY

**Responsibilities**:

* Sign ECDSA messages with the configured private key (ethers.js) to produce X-PAYMENT headers.
* Send POST requests to the live x402 endpoint using axios.
* Track session budgets, daily limits and API call counts to avoid overspending.
* Provide helper methods used by higher-level route handlers and tools: `getSwapQuote()`, `analyzeWallet()`, `executeSwap()` (dry-run and confirmed)

**Key behaviours**:

* Generates a JWT-like payload that includes request metadata and signs using the configured wallet.
* Implements local rate-limiting / budget check before attempting network calls.
* Returns native responses from the x402 API or throws consistent errors for the caller to handle.

**Implementation tips**:

* Keep the signing function small and testable. Unit-test the generated X-PAYMENT header against a fixed keypair.
* Environment toggles allow execution tools to be disabled for safe demos (`ELSA_ENABLE_EXECUTION_TOOLS=false`).

### 4.2 Peer-to-Peer Communication (`agent-dns-server.js`)

**Status**: READY

**Responsibilities**:

* Maintain a lightweight UDP mesh network (P2PMeshNetwork class) that broadcasts/receives JSON messages to/from local peers.
* Announce agent capabilities, listen for peer announcements, and forward negotiation messages.
* Provide a state machine for negotiation messages: NEGO_OFFER, COUNTER, NEGO_ACCEPT, PAYMENT_402_REQUIRED, PAYMENT_402_COMPLETED.

**Notes**:

* Messages are AES-256 encrypted on the wire (shared symmetric keys configured per node) to protect privacy for local demos.
* The mesh is intentionally minimal (no DHT, no NAT traversal) — designed for LAN/hackathon use.

### 4.3 Negotiation Protocol

**State machine**:

* `NEGO_OFFER` — seller offers price & terms
* `COUNTER` — buyer or seller counters the offer
* `NEGO_ACCEPT` — one party accepts
* `PAYMENT_402_REQUIRED` — seller issues an HTTP 402 challenge for the required fee
* `PAYMENT_402_COMPLETED` — buyer sends signed payment, seller validates, then executes

**Message format (JSON)**:

```json
{
  "type": "NEGO_OFFER",
  "id": "uuid",
  "from": "agent-id",
  "to": "agent-id",
  "payload": {
    "service": "swap",
    "pair": "USDC/WETH",
    "amount": 100,
    "price": 2500
  },
  "ts": 1620000000000
}
```

**Failure/fallback rules**:

* If payment is not completed within `PAYMENT_TIMEOUT_MS`, the negotiation aborts and both parties log the failure.
* If x402 payment fails, the seller can optionally move to a fallback `demo-mode` response that returns simulated results.

### 4.4 OpenClaw Agent Integration (`lib/OpenClawAgent.js` and `agent-dns-openclaw-integration.js`)

**Status**: MOCKED — designed as a deterministic demo layer.

**Responsibilities**:

* Expose `decideOnTrade()`, `handleCommand()` and higher-level decision functions used by REST endpoints.
* In the current branch, decision logic is implemented as rule-based heuristics for reliability during demos.

**Mock behaviours**:

* `decideOnTrade()` uses simple rules (e.g., sellers reject < 90% of budget; buyers accept if gap < 5%).
* Natural language commands in `/openclaw/command` are parsed using heuristics (string includes checks). This avoids LLM rate-limits and variability during demos.

**Upgrade path**:

* Replace the rule-based core with a prompt-to-LLM flow (Anthropic/Claude, OpenAI or a local LLM). Provide a minimal prompt schema and parse JSON intent outputs.

### 4.5 REST API & CLI

**Status**: MIXED

**Express Endpoints (representative)**:

* `GET /health` — healthcheck
* `GET /portfolio` — fetch portfolio (calls Elsa client; demo fallback on error)
* `GET /price/:token` — current token price (calls Elsa `price` tool; fallback price when x402 fails)
* `POST /trade/propose` — propose a trade to a discovered peer
* `POST /trade/execute` — finalize a trade (verifies x402 payment and optionally executes swap)
* `POST /openclaw/command` — accept a natural-language instruction (mock parser currently)

**CLI**:

* `npm run seller` — start a seller node (port 8080, mesh port 9999 by default)
* `npm run buyer` — start a buyer node (port 8081, connect to a seller)
* `npm run cli` — interactive shell for manual proposals and monitoring

**Demo-mode behaviours**:

* Many endpoints have `try/catch` wrappers that return `{ status: 'demo mode' }` or a hardcoded price when the Elsa API times out or errors. This ensures the demo never dead-ends.

---

## 5. Installation & Local Setup

**Prerequisites**:

* Node 18+ (recommended)
* npm or yarn
* Local LAN access for multi-machine demos (seller & buyer on same subnet)
* Optionally: an EVM wallet and small amount of testnet funds for on-chain demos

**Clone & install**:

```bash
git clone <your-repo-url>
cd axylossss
npm install
```

**Environment**
Create a `.env` file in the repository root (see Section 6).

---

## 6. Environment Variables

Create `.env` with the following keys (example values shown):

```
NODE_ENV=development
PORT=8080
MESH_PORT=9999
MESH_BROADCAST_ADDR=255.255.255.255
AGENT_ID=seller-01
AES_MESH_KEY=0123456789abcdef0123456789abcdef
ELSA_API_BASE=https://x402-api.heyelsa.ai
ELSA_PRIVATE_KEY=0xYOUR_PRIVATE_KEY_FOR_SIGNING
ELSA_ENABLE_EXECUTION_TOOLS=false
ELSA_SESSION_DAILY_LIMIT=5.00
ELSA_SESSION_CALL_LIMIT=100
PAYMENT_TIMEOUT_MS=120000
```

**Notes**:

* `AES_MESH_KEY` is a demo symmetric key; rotate it for each agent during real tests.
* `ELSA_PRIVATE_KEY` is used to sign X-PAYMENT headers; **never** commit it to source control.
* `ELSA_ENABLE_EXECUTION_TOOLS` toggles whether the execution tools (on-chain swap) will run. Keep false for demo-only runs.

---

## 7. Running the System (Examples)

**Seller node (local)**

```bash
PORT=8080 MESH_PORT=9999 npm run seller
```

**Buyer node (other laptop on same network)**

```bash
PORT=8081 MESH_PORT=9998 npm run buyer
```

**Interactive CLI (single machine)**

```bash
npm run cli
# from the shell: propose trade --pair WETH/USDC --amount 1
```

**Run a swap (with execution enabled)**

```bash
ELSA_ENABLE_EXECUTION_TOOLS=true npm run buyer
# then propose a trade via the CLI or REST endpoint that will progress to PAYMENT_402_REQUIRED
```

---

## 8. API Reference (quick)

> All endpoints are available on the agent HTTP port (e.g. [http://localhost:8080](http://localhost:8080))

### `GET /health`

**Description**: Returns { status: 'ok', ts: <now> }.

### `GET /portfolio`

**Returns**: Portfolio snapshot. Calls Elsa `analyzeWallet()`; if Elsa times out, returns `status: 'demo mode'` and a mock portfolio.

### `GET /price/:token`

**Returns**: Price object `{ token, price, ts }`. Fallback to `price: 2500` on error.

### `POST /trade/propose`

Payload example:

```json
{ "to": "seller-01", "service": "swap", "pair":"USDC/WETH", "amount": 100 }
```

**Flow**: sends NEGO_OFFER message to peer via UDP mesh. Returns `status: 'sent'` with negotiation id.

### `POST /trade/execute`

Payload example:

```json
{ "negotiationId": "uuid", "paymentJwt": "..." }
```

**Flow**: Verifies the x402 payment signature, validates the negotiation state, and (if enabled) triggers on-chain swap execution.

### `POST /openclaw/command`

**Description**: accepts natural language command. In current implementation, uses simple string checks (mock). Example commands: "price WETH", "show portfolio", "buy WETH if price < 2400".

---

## 9. Negotiation & Payment Flow

### Sequence (concise)

1. Buyer calls `POST /trade/propose` or uses CLI.
2. Seller receives NEGO_OFFER via mesh and replies with COUNTER or NEGO_ACCEPT.
3. If agreement reached, Seller sends `PAYMENT_402_REQUIRED` with required fee & metadata.
4. Buyer uses ElsaX402PaymentClient to sign payment and call the x402 API with `X-PAYMENT` header.
5. Seller verifies payment via x402 response and responds `PAYMENT_402_COMPLETED` over mesh.
6. If `ELSA_ENABLE_EXECUTION_TOOLS=true`, seller triggers actual swap through Elsa `executeSwap` tool; otherwise a dry-run result is returned.

### Important failure cases

* Payment timeout → negotiation aborted and both nodes log a `PAYMENT_TIMEOUT` event.
* x402 API error → seller can optionally return demo-mode result to complete the demo flow.

---

## 10. Security Considerations

**Local demo vs production**

* The mesh uses AES-256 symmetric encryption for simplicity (demo). A production system should use authenticated key exchange (e.g., ephemeral ECDH + signatures) and ZK-proof of payment if required.

**Key management**

* Never commit `ELSA_PRIVATE_KEY` to the repo. Use OS-level secret stores or ephemeral wallets.

**Payment verification**

* The seller must verify the x402 response and ensure the signed payload matches the negotiation id to avoid replay attacks.

**Rate-limiting & budgets**

* Client-side budget enforcement is implemented, but server-side budgeting and daily caps on the x402 account would be required in production.

**Trade safety**

* When enabling `ELSA_ENABLE_EXECUTION_TOOLS`, use transaction simulation and slippage limits.

---

## 11. Demo Plan & Suggested Scripts (ETHMumbai-friendly)

**Goal**: Show agent discovery, negotiation, x402 payment, and on-chain settlement in under 2 minutes.

**Prep (before the pitch)**

* Two laptops on the same Wi‑Fi network
* Seller: `npm run seller` (port 8080)
* Buyer: `npm run buyer` (port 8081)
* Ensure `.env` has `ELSA_ENABLE_EXECUTION_TOOLS=false` for safe demo unless you have testnet funds ready

**Script**

1. Show the architecture slide (15s)
2. Start seller and buyer terminals side-by-side — show `Peer discovered` lines (15s)
3. From CLI on buyer laptop: `propose trade WETH 1` (the CLI prints the NEGO sequence) (30s)
4. Show `HTTP 402 Payment Required` printed on buyer terminal (10s)
5. Show `Signing X-PAYMENT header` and `Payment authorized` from Elsa client logs (15s)
6. Show seller `PAYMENT_402_COMPLETED` followed by `Executing swap` or `demo result` and print TX hash if execution on (20s)
7. Close with the product statement: "This enables agent-to-agent commerce with cryptographic payments and on-chain settlement." (15s)

**Pro tip**: Keep UI simple — a tiny web page showing `Connected Peers`, `Current Negotiations`, and `Last TX` will make the demo more accessible.

---

## 12. Testing & Troubleshooting

**Common issues**

* `Peer not discovered` — check both machines are on the same subnet and `MESH_PORT` is not blocked by OS firewall.
* `x402 API errors` — verify `ELSA_API_BASE` and network connectivity. Check signed JWT payload for timestamp drift.
* `Private key errors` — ensure `ELSA_PRIVATE_KEY` is in correct hex format and matches the wallet expected by the x402 account.

**Local unit tests**

* Add unit tests for `ElsaX402PaymentClient.signPayment()` to ensure predictable signatures for a given keypair.
* Add an E2E integration test that runs `seller` and `buyer` in a child process and asserts the negotiation completes (mock x402 or use a sandbox endpoint).

---

## 13. Roadmap & Future Enhancements

**Short-term (hackathon & post-hackathon)**

* Replace the mocked OpenClaw decision engine with a real LLM backend (Anthropic/Claude or OpenAI) and a robust prompt-to-JSON intent schema.
* Add a minimal web dashboard for live visualization of the mesh, negotiations, and transactions.
* Add a credentials store for safe private key handling.

**Medium-term (research / product)**

* Add secure peer authentication (ECDH key exchange + signed announcements).
* Implement micropayment streaming for long-running services (beyond single 402 payments).
* Add an on-chain registry for discoverable agent meta-data (optional: ENS-style names for agents).

**Long-term**

* Integrate reputation & escrow systems (on-chain) to support larger economic activity among agents.
* Design and publish an Agent Economy RFC (protocol spec) that standardizes NEGO message formats and x402 usage for agent-to-agent commerce.

---

## 14. Contributing

**How to contribute**

* Fork the repo, create a topic branch, and open a PR with clear tests.
* For major changes (LLM integration, protocol changes), open an issue first describing the change and intended design.

**Code style**

* ESLint & Prettier: keep code standard and format before PRs.

**Testing**

* Add unit tests for critical functions (payment signing, mesh message encryption/decryption, negotiation state transitions)

---

## 15. Appendix: File map & quick references

```
/ (repo root)
├─ .env.example
├─ package.json
├─ README.md (short)
├─ lib/
│  ├─ ElsaX402PaymentClient.js
│  ├─ OpenClawAgent.js
│  └─ P2PMeshNetwork.js
├─ agent-dns-server.js
├─ agent-dns-openclaw-integration.js
├─ agent-dns-openclaw-integration-fixed.js
├─ cli/
│  └─ index.js
├─ routes/
│  └─ trade.js
└─ utils/
   └─ encryption.js
```

**Quick commands**

* `npm run seller` — start seller node
* `npm run buyer` — start buyer node
* `npm run cli` — interactive CLI
* `ELSA_ENABLE_EXECUTION_TOOLS=true npm run buyer` — enable live on-chain execution (use with caution)

---

If you'd like, I can now:

* split this document into `README.md` + `API.md` + `CONTRIBUTING.md` files inside the repo, or
* create a minimal web dashboard preview (20–30 lines of React) to visualize peers & negotiations, or
* replace the mock OpenClaw decision function with a small LLM prompt-to-JSON adapter using your preferred provider.

Which of these would you like next?
