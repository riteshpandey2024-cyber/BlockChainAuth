# Implementation Plan: Blockchain Product Authenticity Demo

A web app that demonstrates the design in [PROJECT_DESIGN_EXPLAINED.md](PROJECT_DESIGN_EXPLAINED.md): registered nodes, producers, consensus, each node's own blocks, and a central monitor that shows where the chains differ.

**Status:** built. See [blockchain-demo/README.md](blockchain-demo/README.md) to run it. Milestones 1 to 9 are implemented, and 21 backend tests pass.

**What differs from this plan:**

- The backend is **Python FastAPI** (not TypeScript/Express), with `eth-account` for signing. The frontend is React + Vite, as planned. No WorkIndia packages are used.
- Only **Proof-of-Authority** is kept. BFT voting (milestone 10) was built and then removed to keep the demo simple.
- The Tag is **never rewritten** and the signer never changes at Handover (Option A). The signed label is the Product ID plus the Tag identifier; the counters are not signed. `WRITE_TAG` was dropped.
- Added: a `LOG_FAILURE` transaction, an `unconfirmed` scan verdict, a "do not trust the server database" scan option, a node-behaviour control (honest, offline, equivocate, tamper), and a `SET_TOLERANCE` transaction.
- Not built: membership voting, certifying agency, record updates, key revocation, a customer history page.

The sections below are the original plan and are kept for reference.

**Main assumption:** the blockchain is **simulated in code** (several in-process nodes in one TypeScript backend) instead of running real Ethereum nodes. This gives full control over blocks, forks and faulty nodes, which a demo needs. A real Solidity contract on a local chain could be added later. See section 13.

---

## 1. Goals

The app must let you:

1. Register nodes and see them join the network.
2. Register producers.
3. Create products, sign tags, transfer ownership, sell, and scan.
4. See the **blocks each node is making**, side by side.
5. See when nodes hold **different blocks** (a fork) and how consensus resolves it.
6. See a **central monitor** that compares all nodes and flags the ones that disagree.
7. Add nodes, producers and products live while the app runs.

---

## 2. Stack and layout

| Part | Choice | Why |
| --- | --- | --- |
| Backend | Node.js + TypeScript, Express, `ws` (WebSocket) | One language end to end, easy live updates |
| Crypto | `ethers` (`Wallet`, `verifyMessage`) | Real secp256k1 signing and signer recovery, as in the design doc |
| Frontend | React + Vite + Tailwind | Fast to build dashboards |
| State | In memory, plus a JSON snapshot file | No database needed for a demo |

```
blockchain-demo/
  server/
    src/
      chain/        block.ts  tx.ts  hash.ts  state.ts
      contract/     productContract.ts      // the "smart contract" rules
      node/         node.ts  mempool.ts  sync.ts
      consensus/    poa.ts  bft.ts  index.ts
      network/      bus.ts                  // simulated network between nodes
      services/     keyVault.ts  tagService.ts  ipfsMock.ts  db.ts
      monitor/      monitor.ts              // central server view
      api/          routes.ts  ws.ts
      index.ts
  web/
    src/pages/       Network.tsx  Nodes.tsx  Producers.tsx  Products.tsx
                     Scan.tsx  Consensus.tsx  Monitor.tsx
    src/components/  BlockCard.tsx  ChainLane.tsx  DiffBadge.tsx
                     VoteTracker.tsx  EventLog.tsx
```

---

## 3. Core data types (`chain/`)

```ts
type Hex = string;

interface Tx {
  type: 'REGISTER_NODE' | 'REGISTER_PRODUCER' | 'CREATE_PRODUCT'
      | 'TRANSFER' | 'SELL' | 'BUMP_READ' | 'WRITE_TAG';
  payload: any;
  from: Hex;          // sender address
  nonce: number;
  sig: Hex;           // transaction signature
}

interface Block {
  index: number;
  prevHash: Hex;
  timestamp: number;
  proposer: Hex;        // node address
  txs: Tx[];
  stateRoot: Hex;       // hash of contract state after the txs
  hash: Hex;            // hash of everything above
  votes: { node: Hex; sig: Hex }[];   // used in BFT mode
}
```

Block hash = `keccak256` of the fields above. Any change to any field changes it, which is what the monitor compares across nodes.

---

## 4. The contract (`contract/productContract.ts`)

A deterministic state machine. Every node runs the **same** function on the same block, so they reach the same `stateRoot`.

```ts
interface Product {
  id: string; dataHash: Hex; owner: Hex; signer: Hex;
  tagId: string; writeCounter: number; readCounter: number;
  status: 'registered' | 'sold'; history: HistoryEntry[];
}
interface ContractState {
  validators: Set<Hex>;              // approved nodes
  producers: Map<Hex, ProducerInfo>;
  products: Map<string, Product>;
}

function applyTx(state: ContractState, tx: Tx): Result {
  switch (tx.type) {
    case 'REGISTER_PRODUCER': /* only admin */ ...
    case 'CREATE_PRODUCT':    /* sender must be an approved producer; id unused;
                                 store dataHash, signer = sender, owner = sender */ ...
    case 'TRANSFER':          /* require tx.from === product.owner */ ...
    case 'SELL':              /* not already sold */ ...
    case 'BUMP_READ':         /* validateTag(...) then readCounter += 1 */ ...
  }
}

// Read-only: runs on ONE node, changes nothing
function validateTag(state, tag): { ok: boolean; reason?: string } {
  // 1. product exists
  // 2. msgHash = keccak256(id | tagId | writeCounter | contractAddr)
  // 3. recovered = ethers.verifyMessage(msgHash, sig); require recovered === product.signer
  // 4. tag.tagId === product.tagId && tag.writeCounter === product.writeCounter
  // 5. read counter: tag.readCounter vs product.readCounter
  //      lower than stored  => stale (clone or replay), flag
  //      higher than stored => allowed up to TOLERANCE (configurable in the UI)
}
```

The read counter is **not** part of the signed message, matching the corrected design (the chip raises it on every scan, so signing it would break the signature).

---

## 5. A node (`node/node.ts`)

```ts
class Node {
  address: Hex; wallet: Wallet;
  chain: Block[]; state: ContractState; mempool: Tx[];
  peers: Node[]; mode: 'honest' | 'offline' | 'byzantine';

  receiveTx(tx)        // validate signature + nonce, add to mempool, gossip
  proposeBlock()       // only if it is this node's turn
  receiveBlock(b)      // re-check prevHash, proposer allowed, each tx valid, stateRoot matches
  readCall(fn, args)   // uses ONLY this node's state; nothing is agreed
  sync(from: Node)     // adopt the majority/heaviest chain after a fork
}
```

`receiveBlock` re-runs every transaction on a copy of this node's own state and compares `stateRoot`. This is how "each node verifies independently" really happens in the demo.

`mode` lets you demo faults:

- `offline` drops messages, so the node's chain falls behind.
- `byzantine` proposes a tampered block, or sends different blocks to different peers.

---

## 6. Network simulator (`network/bus.ts`)

An in-process message bus between nodes, with controls the UI can change:

```ts
bus.send(from, to, msg)         // delivers after latency
bus.setLatency(ms)
bus.partition([A, B], [C, D])   // two groups cannot hear each other
bus.heal()
```

The partition control is what produces **different blocks on different nodes**.

---

## 7. Consensus (`consensus/`)

One interface, two implementations, switchable in the UI.

```ts
interface Consensus {
  onTick(node: Node): void;
  onBlock(node: Node, b: Block): void;
}
```

### Mode A: Proof-of-Authority (`poa.ts`), build first

- Validators take turns: proposer = `validators[blockIndex % n]`.
- A node accepts a block only from the scheduled proposer.
- If two branches exist, pick the **heaviest** (most distinct signers, then longest), as in section 3A of the design doc.
- Partitions create real forks, then heal into one chain.

### Mode B: BFT-style voting (`bft.ts`)

- The proposer broadcasts a block; validators check it and broadcast signed votes.
- Finalize on `floor(2n/3) + 1` votes or more.
- A byzantine proposer sending two different blocks gets no quorum, so nothing commits until the next proposer.
- The vote tally streams to the UI.

---

## 8. Services (`services/`)

| File | Role |
| --- | --- |
| `keyVault.ts` | Holds each producer's private key (as in the design). A toggle in the UI shows the "who holds the key" risk |
| `tagService.ts` | Builds and signs tag data `{id, tagId, writeCounter}` and returns the tag JSON |
| `ipfsMock.ts` | Content-addressed store: `put(record) -> hash`, `get(hash)` |
| `db.ts` | Off-chain records and tag copies for the "database check" step |

---

## 9. Central server and monitor (`monitor/monitor.ts`)

Polls every node and builds one comparison view:

```ts
function snapshot() {
  const heads = nodes.map(n => ({
    node: n.address, height: n.chain.length, headHash: n.head().hash,
  }));
  const groups = groupBy(heads, h => h.headHash);
  const majority = largestGroup(groups);
  return {
    heads, groups,
    forked: groups.length > 1,
    divergent: heads.filter(h => h.headHash !== majority.headHash),
    firstDivergentIndex: findFirstDifferentBlock(nodes),
  };
}
```

It also records commit history so the UI can draw the fork as a tree and show the exact block where nodes first disagree.

---

## 10. API (`api/routes.ts`)

| Method | Route | Does |
| --- | --- | --- |
| POST | `/nodes` | Register a node (generates a keypair, sends `REGISTER_NODE`) |
| POST | `/nodes/:id/mode` | Set `honest`, `offline` or `byzantine` |
| POST | `/producers` | Register a producer (admin) |
| POST | `/products` | Create a product: record to IPFS and DB, hash on-chain |
| POST | `/products/:id/tag` | Sign and return tag data |
| POST | `/products/:id/transfer` | Owner-only transfer |
| POST | `/scan` | Full scan: DB check, contract read, IPFS check, then `BUMP_READ` transaction |
| POST | `/network/partition`, `/network/heal` | Split or rejoin the nodes |
| POST | `/consensus/mode` | Switch between `poa` and `bft` |
| GET | `/monitor` | The snapshot above |
| WS | `/events` | Live stream: tx, proposal, vote, block, fork, sync |

---

## 11. Web UI pages

| Page | Shows | Interactions |
| --- | --- | --- |
| **Network** | Each node as a card: address, mode, height, head hash. Lanes of recent blocks per node side by side, with differing blocks highlighted red | Add node, toggle mode, partition and heal |
| **Consensus** | Current proposer, vote tracker (X of N votes), threshold, PoA/BFT switch, a live step list | Switch mode, force a byzantine proposal |
| **Producers** | Registered producers, their addresses, and which are approved | Register a producer |
| **Products** | Table of products with owner, counters and history | Create, transfer, sell |
| **Scan** | A virtual NFC tag as editable JSON, plus a step-by-step result: DB check, signer recovered vs registered, counters, IPFS hash | Buttons: *Scan*, *Edit a field*, *Clone this tag*, *Replay old scan* |
| **Monitor** (central server) | Majority head vs each node's head, divergent nodes flagged, fork tree, "first different block" | Auto-refresh, block diff viewer |
| **Event log** | Every transaction, block and vote in time order | Filter by node |

Components: `BlockCard` shows index, short hash, proposer, transaction count and state root. `DiffBadge` compares two nodes' blocks field by field.

---

## 12. Build order, with a visible check at each step

| # | Milestone | Check you can see |
| --- | --- | --- |
| 1 | `hash.ts`, `Block`, `Tx`, signing with ethers | A script prints a signed transaction and recovers the right signer |
| 2 | `productContract.ts` + unit tests | Edited tag fails, cloned tag fails, non-owner transfer fails |
| 3 | `Node` with its own chain; run 4 nodes in one process | Each node holds an identical chain after a few transactions |
| 4 | `bus` + PoA consensus | Nodes rotate proposers; the same head hash everywhere |
| 5 | Express API + WebSocket events | `curl` creates a product and the events stream |
| 6 | Network and Producers pages | Add a node and a producer in the browser and see them appear |
| 7 | Products and Scan pages | A genuine scan passes; editing a field fails with the reason shown |
| 8 | Partition and heal | Two groups build different blocks, then merge to one chain |
| 9 | Monitor page | Divergent nodes shown red; first different block pinpointed |
| 10 | BFT mode + vote tracker | A byzantine proposer gets no quorum, and you can see why |

---

## 13. Demo script

1. Start with 4 validators and register 2 producers.
2. Create product `W-1001` and sign its tag. Show the block on every node.
3. Scan it: genuine, with the read counter going up on all nodes.
4. Edit a tag field: rejected, with the recovered address shown not matching the registered one.
5. Clone the tag and scan: counter mismatch.
6. Switch one node to `byzantine`, or partition 2 nodes from 2: the monitor shows different blocks.
7. Heal the network: the minority nodes sync to the majority chain.

---

## 14. Decisions needed before building

1. **Simulated chain (recommended) or real Ethereum nodes** (Hardhat or Besu with a Solidity contract)? Real nodes are more authentic but give much less control over forks and faulty nodes.
2. **Start with PoA only** (faster to build), or build both PoA and BFT?
3. **Is a presentation the deliverable, or also a code submission?** This affects how much testing and documentation work is needed.

---

## 15. Known limits of the demo

- A simulated chain demonstrates the mechanisms but is not a production blockchain: no real networking, no persistence guarantees, one process.
- The server holds producers' private keys (as proposed in the design), so a compromised server can sign valid-looking tags. The key-vault toggle is meant to make this visible.
- Tag hardware is simulated. A real NFC chip's counter and serial-number behavior is not tested.
- The contract is TypeScript rules standing in for a smart contract, not Solidity.
