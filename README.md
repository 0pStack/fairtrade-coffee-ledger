# Fair Trade Coffee Ledger

A decentralized logistics ledger for Fair Trade coffee. Every time a batch of beans
moves from a farm to a roastery, or from a roastery to a café, the shipment is logged
as a transaction and mined into a block. The chain is protected by a SHA-256
Proof-of-Work mechanism, so rewriting a past shipment is detectable — and expensive.

Built as a Node.js + Express REST API with TypeScript, developed test-first with
Vitest and Supertest.

Course assignment — Medieinstitutet, Backend Node.js.

---

## Requirements

| | |
| --- | --- |
| Node.js | **24 or later** (22.18+ also works) |
| npm | 10 or later |

> The source is TypeScript and is executed directly by Node's native type stripping —
> there is no build step. That feature is only available from Node 22.18 / 24, so an
> older Node will fail to start. Check with `node --version`.

## Getting started

```bash
npm install
npm run dev          # http://localhost:3000, auto-restarts on save
```

| Script | What it does |
| --- | --- |
| `npm run dev` | Runs the server through nodemon, restarting on every save |
| `npm start` | Runs the server once |
| `npm test` | Runs the full test suite |
| `npm run test:watch` | Re-runs affected tests as you edit |
| `npm run coverage` | Test suite plus a coverage report (`coverage/index.html`) |
| `npm run typecheck` | Type checks without emitting — Node strips types but never checks them |

---

## API

### `GET /`

Lists the routes the API serves, so opening the root URL in a browser is informative
rather than a 404.

```bash
curl http://localhost:3000/
```

```json
{
  "name": "Fair Trade Coffee Ledger",
  "description": "Proof-of-Work blockchain ledger for tracking coffee shipments",
  "endpoints": {
    "GET /blockchain": "Return the full chain, the pending pool and its validity",
    "POST /transactions": "Queue a coffee shipment for the next block",
    "POST /mine": "Mine every pending transaction into a new block"
  }
}
```

### `GET /blockchain`

Returns the whole chain so anyone can audit a batch's journey.

```bash
curl http://localhost:3000/blockchain
```

```json
{
  "length": 1,
  "chain": [
    {
      "index": 0,
      "timestamp": 1787159591156,
      "transactions": [],
      "previousHash": "0",
      "nonce": 0,
      "hash": "6a4beb9611b54063167db563afbebb6b53a9d0e87a8598a8b2665cf4348b1ba3"
    }
  ],
  "pendingTransactions": [],
  "isValid": true
}
```

`isValid` is the live result of `isChainValid()` — the audit runs on every read.

### `POST /transactions`

Validates a shipment and queues it. It is **not** part of the ledger until it has been
mined.

```bash
curl -X POST http://localhost:3000/transactions \
  -H "Content-Type: application/json" \
  -d '{"sender":"Finca La Esperanza","recipient":"Nordic Roastery","batchId":"BATCH-001","weightKg":60}'
```

**201 Created**

```json
{
  "message": "Shipment queued for block 1",
  "blockIndex": 1,
  "transaction": {
    "sender": "Finca La Esperanza",
    "recipient": "Nordic Roastery",
    "batchId": "BATCH-001",
    "weightKg": 60
  }
}
```

**400 Bad Request** — every problem is reported at once, not just the first:

```json
{
  "error": "Validation failed",
  "details": ["batchId is required and must be a non-empty string"]
}
```

| Field | Rule |
| --- | --- |
| `sender` | non-empty string |
| `recipient` | non-empty string |
| `batchId` | non-empty string |
| `weightKg` | finite number greater than 0 |

### `POST /mine`

Takes every pending transaction, runs Proof-of-Work until a valid hash is found,
appends the block, empties the pool and returns the new block.

```bash
curl -X POST http://localhost:3000/mine
```

**201 Created**

```json
{
  "message": "Block 1 mined",
  "block": {
    "index": 1,
    "timestamp": 1787159591660,
    "transactions": [
      {
        "sender": "Finca La Esperanza",
        "recipient": "Nordic Roastery",
        "batchId": "BATCH-001",
        "weightKg": 60
      }
    ],
    "previousHash": "6a4beb9611b54063167db563afbebb6b53a9d0e87a8598a8b2665cf4348b1ba3",
    "nonce": 488,
    "hash": "000c1de74857f78671fa67729df0a506ccac8cfa009411753338409fad2323e4"
  }
}
```

**400 Bad Request** when the pool is empty — no empty blocks are allowed into the ledger:

```json
{ "error": "Cannot mine: no pending transactions" }
```

Unknown routes return `404` as JSON rather than Express's default HTML.

---

## How the Proof-of-Work works

A block's hash is `SHA-256` over `index + previousHash + timestamp + transactions + nonce`.
Everything in that payload is fixed by what the block *says* — except the nonce, which is a
meaningless counter. So the only way to find a hash starting with the required number of
zeros is to keep incrementing the nonce and re-hashing:

```ts
while (!hash.startsWith(target)) {
  nonce += 1;
  hash = calculateHash({ ...candidate, nonce });
}
```

Each additional leading zero makes a valid hash roughly 16× rarer: difficulty 1 needs about
16 attempts, difficulty 3 about 4,096. The work is unavoidable to produce and trivial to
verify — one hash — which is what makes the ledger tamper-evident rather than tamper-proof.

### Difficulty is configured by environment

A real mining loop would time out the test suite, so difficulty is resolved from the
environment (`src/config.ts`):

| Environment | Difficulty |
| --- | --- |
| `NODE_ENV=test` | 1 (immediate) |
| anything else | 3 |
| `POW_DIFFICULTY=n` | `n` — overrides both |

`vitest.config.ts` sets `NODE_ENV=test` explicitly, so the whole suite runs in under a second.

### `isChainValid()`

Walks the chain from block 1 (the genesis block has no predecessor) and checks two things
per block:

1. does the stored `hash` still match a fresh recomputation of the block's contents?
2. does `previousHash` still match the real hash of the block before it?

Check 1 catches an edited block. Check 2 catches the smarter fraudster who edits a block
**and** re-hashes it, because that changes the hash every following block points at. This is
why rewriting history means re-mining the entire remainder of the chain — the cost the
Proof-of-Work exists to impose.

---

## Test-driven development

Every feature was written test-first. The tests below were committed **failing**, and the
implementation followed in the next commit.

| # | Feature | RED — test first | GREEN — implementation |
| --- | --- | --- | --- |
| 1 | `calculateHash` | [`976da77`](https://github.com/0pStack/fairtrade-coffee-ledger/commit/976da77) | [`fd164d0`](https://github.com/0pStack/fairtrade-coffee-ledger/commit/fd164d0) |
| 2 | Genesis block + `addTransaction` | [`75fa4b2`](https://github.com/0pStack/fairtrade-coffee-ledger/commit/75fa4b2) | [`667cdcf`](https://github.com/0pStack/fairtrade-coffee-ledger/commit/667cdcf) |
| 3 | Proof-of-Work + env difficulty | [`b150e22`](https://github.com/0pStack/fairtrade-coffee-ledger/commit/b150e22) | [`dd52f29`](https://github.com/0pStack/fairtrade-coffee-ledger/commit/dd52f29) |
| 4 | `isChainValid` tamper detection | [`a930a7e`](https://github.com/0pStack/fairtrade-coffee-ledger/commit/a930a7e) | [`5c99221`](https://github.com/0pStack/fairtrade-coffee-ledger/commit/5c99221) |
| 5 | REST API + validation middleware | [`c667b4f`](https://github.com/0pStack/fairtrade-coffee-ledger/commit/c667b4f) | [`1672354`](https://github.com/0pStack/fairtrade-coffee-ledger/commit/1672354) |
| 6 | API index route | [`0721d63`](https://github.com/0pStack/fairtrade-coffee-ledger/commit/0721d63) | [`3e2600d`](https://github.com/0pStack/fairtrade-coffee-ledger/commit/3e2600d) |

Each RED commit message records the exact failure, e.g. *"Fails with: TypeError:
blockchain.isChainValid is not a function"*.

### Coverage

```
File               | % Stmts | % Branch | % Funcs | % Lines
-------------------|---------|----------|---------|--------
All files          |     100 |      100 |     100 |     100
 src               |     100 |      100 |     100 |     100
  app.ts           |     100 |      100 |     100 |     100
  blockchain.ts    |     100 |      100 |     100 |     100
  config.ts        |     100 |      100 |     100 |     100
  hash.ts          |     100 |      100 |     100 |     100
 src/middleware    |     100 |      100 |     100 |     100
  ...ransaction.ts |     100 |      100 |     100 |     100
```

65 tests across 4 files. Thresholds are enforced at 80% in `vitest.config.ts`, so the suite
fails if coverage regresses. `server.ts` is excluded — it only binds a port — and `types.ts`
is excluded because interfaces are erased at runtime and have no executable statements.

Integration tests use **Supertest**, which starts the exported app on an ephemeral port
inside the test process. No server needs to be running to test the API.

---

## Project structure

```
src/
  types.ts                        Transaction and Block shapes
  hash.ts                         calculateHash — SHA-256 over a block
  config.ts                       getDifficulty — reads NODE_ENV / POW_DIFFICULTY
  blockchain.ts                   Blockchain class: chain, pool, mining, validation
  middleware/
    validateTransaction.ts        Rejects malformed shipments at the boundary
  app.ts                          createApp() — routes and middleware, no listen()
  server.ts                       The only file that binds a port
tests/
  hash.test.ts                    Unit
  blockchain.test.ts              Unit
  config.test.ts                  Unit
  api.test.ts                     Integration (Supertest)
```

### Design notes

**`app.ts` exports the app; `server.ts` calls `listen()`.** If the app module bound a port at
import time, importing it from a test would start a real server and the suite would hang or
collide on the port. This split is what makes the API testable.

**`createApp(blockchain?)` takes the ledger as a parameter.** Each test gets an isolated
chain instead of sharing module-level state, so tests cannot leak into one another.

**Blocks are never hardcoded.** Even the genesis block is produced by `createGenesisBlock()`,
so every block in the chain came from the same code path.

**Updates are immutable.** Mining hands the pending array to the new block and then points
`pendingTransactions` at a fresh array. Clearing it in place with `.length = 0` would have
emptied the array the block itself holds.

**Validation is hand-written middleware** rather than a schema library. The rules are four
fields deep, and writing them out keeps the dependency count at zero. In a production service
this is where a schema validator such as Zod would earn its place, deriving the `Transaction`
type from the schema instead of duplicating it.

---

## Known limitations

- **The ledger is in memory.** Restarting the server starts a fresh chain. Persistence was
  out of scope for the assignment.
- **One node, not a network.** There is no peer discovery or consensus, so "decentralized"
  describes the data structure rather than the deployment.
- **`JSON.stringify` is used to serialize transactions for hashing**, which is sensitive to
  key order. Safe here because every transaction is constructed by one code path, but a real
  multi-node ledger would need canonical serialization so independent nodes agree.
- **`isChainValid()` does not re-check the Proof-of-Work difficulty** of stored blocks, per
  the specified two checks. Adding `hash.startsWith('0'.repeat(difficulty))` would also reject
  a forger who re-hashed a block without doing the work.
