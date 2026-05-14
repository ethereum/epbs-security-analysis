# ePBS: Formal Pseudocode and Algorithms

This document contains the formal definitions, algorithms, and helper functions for ePBS (EIP-7732). It is the formal layer of a two-part treatment: the [overview](https://github.com/ethereum/epbs-security-analysis/blob/master/README.md) (on the `master` branch of this repository) provides a self-contained middle-layer document with abstracted code and figures; this document provides the full formal model with proofs.

**Spec version.** This document targets [`ethereum/consensus-specs`](https://github.com/ethereum/consensus-specs) at commit [`fa8bb08`](https://github.com/ethereum/consensus-specs/commit/fa8bb08), Gloas fork: [`specs/gloas/`](https://github.com/ethereum/consensus-specs/tree/master/specs/gloas). The Gloas specs are work-in-progress and may differ from the EIP-7732 draft summary.

## 12. Fork Choice: Model, Definitions, and Algorithms

This section formalizes the ePBS fork-choice rule. Ethereum's existing fork-choice function (LMD-GHOST, as part of the Gasper consensus protocol) selects the canonical chain by iteratively picking the child with the highest attestation weight. ePBS does not replace this mechanism — it extends it by making the *fork-choice node* — a `(root, payload_status)` pair — the primary abstraction. Multiple nodes can reference the same block, each carrying a different payload status (PENDING, FULL, EMPTY), which represents the two-phase block model. A new tiebreaker based on PTC votes, and a zero-return rule that delegates the immediately previous slot's payload-status decision to the tiebreaker, complete the extension. We first define the model (blocks, votes, nodes, the fork-choice tree), then present compact algorithms with numbered lines suitable for proving safety properties. Each definition and algorithm line is traceable to the spec ([`fork-choice.md`](https://github.com/ethereum/consensus-specs/blob/master/specs/gloas/fork-choice.md), [`beacon-chain.md`](https://github.com/ethereum/consensus-specs/blob/master/specs/gloas/beacon-chain.md)). Throughout this document, all unqualified spec-file mentions (e.g., *Spec reference: X in `fork-choice.md`*) refer to files in the [`specs/gloas/`](https://github.com/ethereum/consensus-specs/tree/master/specs/gloas) folder.

### 12.1 Model and Definitions

**Definition 1** (Slot structure). Time is divided into *slots* of fixed duration `SLOT_DURATION` (12 seconds). Each slot N is partitioned into four phases:

| Phase                    | Time            | Deadline constant                                      | Actor       | Action                                                                                           |
| ------------------------ | --------------- | ------------------------------------------------------ | ----------- | ------------------------------------------------------------------------------------------------ |
| **Proposal**       | t = 0           | —                                                     | Proposer    | Broadcasts beacon block containing the builder's bid                                             |
| **Attestation**    | t = T_att       | `ATTESTATION_DUE_BPS_GLOAS = 2500` (25%, deadline)   | Attesters   | Broadcast attestations for the beacon block                                                      |
| **Builder reveal** | t ∈ (0, T_ptc) | —                                                     | Builder     | Broadcasts `SignedExecutionPayloadEnvelope`                                                    |
| **PTC vote**       | t ∈ (0, T_ptc] | `PAYLOAD_ATTESTATION_DUE_BPS = 7500` (75%, deadline) | PTC members | Broadcast payload timeliness votes within the first 75% of the slot (deadline, not exact firing) |

With `SLOT_DURATION = 12s`: T_att = 3s, T_ptc = 9s. The ordering T_att < T_ptc is a fundamental design constraint: attesters vote before the builder is expected to reveal, and PTC members vote after the builder has had time to reveal. This ordering implies that same-slot attesters cannot observe the payload status at the time they vote.

A note on the PTC entry: the spec specifies `PAYLOAD_ATTESTATION_DUE_BPS` as a *deadline*, not as an exact firing time — an honest PTC member must broadcast its vote *within the first* 75% of the slot. The Section 12.2 PTC handler models the worst-case (deadline) firing for proof-obligation simplicity, but a rational/cautious member typically waits close to T_ptc to maximise information. See the timing-semantics commentary in Section 12.2 for details.

*Spec reference: timing constants in `validator.md`; `ATTESTATION_DUE_BPS_GLOAS`, `PAYLOAD_ATTESTATION_DUE_BPS` in `validator.md`.*

**Definition 2** (Payload status). A *payload status* is one of three values: EMPTY = 0, FULL = 1, PENDING = 2. A node with status EMPTY represents the world where the referenced block's execution payload was not revealed; FULL represents the world where it was revealed and validated; PENDING represents the entry point before the full/empty determination is made.

*Spec reference: `PAYLOAD_STATUS_EMPTY`, `PAYLOAD_STATUS_FULL`, `PAYLOAD_STATUS_PENDING` in `fork-choice.md`.*

**Definition 3** (Block). A *block* B has a slot `slot(B)`, a proposer, a parent `parent(B)` (another block), a root `root(B)` (a unique cryptographic hash identifying B), and a *bid* `bid(B)` containing two hashes:

- `bid(B).block_hash`: the hash of the execution payload the builder commits to deliver.
- `bid(B).parent_block_hash`: the execution chain parent the builder declares it builds upon.

The genesis block B_genesis has no parent. We write B ≼ B' when B is an ancestor of B' (or B = B'), and B ≺ B' when B ≼ B' and B ≠ B'. Two blocks *conflict* if neither is an ancestor of the other. Given a root r, we write B_r for the block with `root(B_r) = r`.

*Spec reference: `BeaconBlock` with `signed_execution_payload_bid.message.block_hash` and `parent_block_hash` in `beacon-chain.md`.*

**Definition 4** (Parent payload status). For any non-genesis block B, the *parent payload status* of B is:

$$
\text{parentStatus}(B) = \begin{cases} \text{FULL} & \text{if } \text{bid}(B).\text{parent\_block\_hash} = \text{bid}(\text{parent}(B)).\text{block\_hash} \\ \text{EMPTY} & \text{otherwise} \end{cases}
$$

This comparison determines whether B was built on the full or empty version of its parent. If the parent's builder revealed a valid execution payload, `process_parent_execution_payload` (called inside the next block's `process_block`) updated `state.latest_block_hash` to `bid(parent(B)).block_hash`, and B's builder, reading that state, set `bid(B).parent_block_hash` to the same value — hence FULL. If the parent's builder withheld, `state.latest_block_hash` was never updated, B's builder read a stale value, and the comparison fails — hence EMPTY.

*Spec reference: `get_parent_payload_status` in `fork-choice.md`.*

**Definition 4b** (Block status — chain-relative). Write $B \prec \mathit{chain}$ to mean "$B$ is a non-head canonical block on $\mathit{chain}$" (i.e., $B$ is on $\mathit{chain}$ and $B \neq \mathit{head}(\mathit{chain})$), and let $\mathit{child}(\mathit{chain}, B)$ denote $B$'s unique canonical child on $\mathit{chain}$ (the block whose $\mathit{parent\_root}$ equals $B$'s hash-tree root). Then:

$$
\mathit{block\_status}(\mathit{chain}, B) = \begin{cases} \text{FULL} & \text{if } B \prec \mathit{chain} \text{ and } \mathit{bid}(\mathit{child}(\mathit{chain}, B)).\mathit{parent\_block\_hash} = \mathit{bid}(B).\mathit{block\_hash} \\ \text{EMPTY} & \text{if } B \prec \mathit{chain} \text{ and } \mathit{bid}(\mathit{child}(\mathit{chain}, B)).\mathit{parent\_block\_hash} \neq \mathit{bid}(B).\mathit{block\_hash} \\ \text{undefined} & \text{otherwise} \end{cases}
$$

**Why undefined at the head.** When $B = \mathit{head}(\mathit{chain})$, $B$'s `bid.block_hash` has been committed but no successor has yet referenced it. Status becomes defined once $B$'s child lands.

This is the chain-relative companion to Definition 4: for $B \prec \mathit{chain}$, $\mathit{block\_status}(\mathit{chain}, B) = \mathrm{parentStatus}(\mathit{child}(\mathit{chain}, B))$. Definition 4 reads the FULL/EMPTY declaration from the child's perspective (intrinsic to $B$'s child); Definition 4b reads the same fact relative to a named chain.

*Spec reference:* No direct spec function. This formalises the on-chain meaning of FULL/EMPTY in the overview's §3, parameterised by the canonical chain.

**Definition 4c** (Payload hash chain). The *payload hash chain* is the collection of $\mathit{bid}(B).\mathit{block\_hash}$ values committed by every FULL (in the sense of Definition 4b) block on the canonical chain, inheriting the canonical chain's order. Equivalently, a payload hash $h$ belongs to the payload hash chain if and only if there exists a non-head canonical block $B^*$ such that $\mathit{bid}(B^*).\mathit{block\_hash} = h$ AND $\mathit{block\_status}(\mathit{canonical}, B^*) = \text{FULL}$.

(FULL is undefined at the head, so the head's $\mathit{bid}.\mathit{block\_hash}$ is *not* on chain — it only enters once a successor lands.)

**Inductive construction (head-to-genesis).** Given a head beacon block $B$ on the canonical chain, the payload hash chain can be enumerated by walking backwards from $B$. Let $h^{(0)}, h^{(1)}, h^{(2)}, \ldots$ be the sequence defined by:

- **Base case.** $h^{(0)} := \mathit{bid}(B).\mathit{parent\_block\_hash}$ — the hash of the latest confirmed execution payload (the one referenced by $B$'s bid as its parent).
- **Recurrence (for each $k \geq 0$).** Let $B^{(k)}$ be the unique canonical ancestor of $B$ such that $\mathit{bid}(B^{(k)}).\mathit{block\_hash} = h^{(k)}$ — i.e., $B^{(k)}$ itself is the block whose bid committed to the payload with hash $h^{(k)}$. Then set $h^{(k+1)} := \mathit{bid}(B^{(k)}).\mathit{parent\_block\_hash}$.
- **Termination.** The recurrence stops when no such $B^{(k)}$ exists — i.e., when $h^{(k)}$ corresponds to a payload at or before genesis (or the slot before ePBS activation).

This enumeration produces the chain in *reverse* execution order (head-to-genesis): $h^{(0)}$ is the most recent, $h^{(k)}$ gets older as $k$ grows.

*Spec reference:* No direct spec function. This formalises the overview's §3 "payload hash on chain" notion.

**Definition 4d** (Payload hash on chain). A payload hash $h$ is **on chain** if it belongs to the payload hash chain (Definition 4c) of the canonical beacon chain.

*Spec reference:* Pedagogical. The strongest hash-level statement derivable from on-chain data alone.

**Definition 5** (Fork-choice node). A *fork-choice node* is a pair `(r, s)` where `r` is a block root and `s` is a payload status. The node — not the block — is the primary fork-choice abstraction: `get_head` traverses nodes, `get_weight` scores nodes, and `is_supporting_vote` targets nodes. Up to three nodes can reference the same block B: `(r, PENDING)`, `(r, FULL)`, and `(r, EMPTY)`.

*Spec reference: `ForkChoiceNode` in `fork-choice.md`.*

**Definition 6** (Fork-choice tree). The *fork-choice tree* T is defined by the children relation. For a node `(r, s)`:

- If `s = PENDING`: the children are `(r, EMPTY)` (always) and `(r, FULL)` (only if `r ∈ store.payloads`, i.e., the execution payload for r was locally received and validated — see Definition 8).
- If `s ∈ {FULL, EMPTY}`: the children are `(r', PENDING)` for each block B' with root r' such that `parent(B') = B_r` and `parentStatus(B') = s`.

The tree therefore alternates: PENDING → {FULL, EMPTY} → PENDING → {FULL, EMPTY} → ...

*Spec reference: `get_node_children` in `fork-choice.md`.*

**Definition 7** (Vote). A *vote* is a tuple `(slot, root, payload_present)` where `slot` is the attestation slot, `root` is the block root the attester considers the chain head, and `payload_present` is derived from the attestation's `data.index` field (`True` if `index = 1`, `False` if `index = 0`). Each validator has at most one vote stored (the most recent).

A vote is a *same-slot vote* for block B if `slot = slot(B)`. Same-slot voters always set `payload_present = False` because, by Definition 1, the attestation phase (T_att) precedes the builder reveal window: attesters vote at 25% of the slot while the builder has until 75%. At the time of voting, the attester has no way to know whether the payload will arrive. This is formalized in the honest attester behavior (Section 12.2, Attester lines 9–10). (Votes with `slot < slot(B)` are invalid: the spec asserts `vote.slot >= block.slot` in `is_supporting_vote`.)

A vote is a *non-same-slot vote* for block B if `slot > slot(B)`. This occurs when no block was proposed in the voter's assigned slot and their head is an earlier block. Non-same-slot voters observe the payload status and set `payload_present` accordingly (Section 12.2, Attester lines 11–12).

*Spec reference: `LatestMessage` and `update_latest_messages` in `fork-choice.md`; attestation `data.index` rules in `validator.md`.*

**Definition 8** (Store). The *store* is the local fork-choice state maintained by each node. We use the spec field names verbatim:

- `blocks[r]`: the block B_r, for each known block root r.
- `block_states[r]`: the post-state of block B_r after `process_block` (which now includes parent execution effects via `process_parent_execution_payload`). Present for every known block.
- `payloads[r]`: the `ExecutionPayloadEnvelope` for block B_r, stored after `verify_execution_payload_envelope` succeeds. Present only if the payload was received and verified.
- `latest_messages[i]`: the latest vote (Definition 7) of validator i, stored as a `LatestMessage(slot, root, payload_present)` record.
- `equivocating_indices`: the set of validator indices that have been observed equivocating (double-voting). Their votes are excluded from weight computation.
- `payload_timeliness_vote[r]`: array of PTC_SIZE entries (each `None` initially, then `True`/`False`) recording each PTC member's `payload_present` vote for block with root r.
- `payload_data_availability_vote[r]`: same structure for `blob_data_available` votes.
- `justified_checkpoint`: the current justified checkpoint `(epoch, root)`. The fork-choice traversal starts from this root.
- `checkpoint_states[C]`: the beacon state at checkpoint C. Used as the balance source for weight computation.
- `proposer_boost_root`: the root of the block receiving proposer boost in the current slot (or the zero root if none).

The current slot is derived from the store's wall-clock time via the spec helper `get_current_slot(store)`; we write it inline (rather than as a store field) wherever it appears in the algorithms below.

*Spec reference: `Store` in `fork-choice.md` — `block_states`, `payloads`, `latest_messages`, `equivocating_indices`, `payload_timeliness_vote`, `payload_data_availability_vote`, `proposer_boost_root`, `block_timeliness`; `get_current_slot` in `phase0/fork-choice.md`.*

**Definition 9** (Ancestor). For a root r and a slot t with `t ≤ slot(B_r)`, the *ancestor of r at slot t*, written `anc(r, t)`, is the fork-choice node identifying which block at slot t is in B_r's chain and what its payload status is:

- If `slot(B_r) ≤ t`: return `(r, PENDING)`. The block B_r is at or before the queried slot; it is its own ancestor, and no FULL/EMPTY determination applies at this level.
- Otherwise: walk back from B_r via `parent` until finding a block D such that `slot(parent(D)) ≤ t < slot(D)`. Return `(root(parent(D)), parentStatus(D))`.

In the second case, D is the earliest descendant in B_r's chain whose parent is at or before slot t. D's `bid.parent_block_hash` declares which version (FULL or EMPTY) of parent(D) it built on, read via parentStatus (Definition 4).

In practice, `anc` is called by `is_supporting_vote` (Definition 10, Case 2) to answer: "a validator voted for block C — which version (FULL or EMPTY) of an earlier block B is in C's ancestry?" The function walks from C back to B's slot and reads the declaration at the link that crosses B's slot. It reads only block-level data (bid hashes and parent pointers); no BeaconState is consulted.

*Spec reference: `get_ancestor` in `fork-choice.md`.*

**Definition 10** (Supporting vote). A vote `m = (slot, root, payload_present)` *supports* a fork-choice node `(r, s)` if:

**Case 1** (`root = r`): the vote targets this exact block.

- If `s = PENDING`: the vote supports the node (any vote for B supports B before the full/empty split).
- If `s ∈ {FULL, EMPTY}` and `slot = slot(B_r)`: the vote does **not** support the node. This is a same-slot vote with no payload opinion. (Votes with `slot < slot(B_r)` are invalid — a vote cannot target a block from a future slot.)
- If `s ∈ {FULL, EMPTY}` and `slot > slot(B_r)`: the vote supports the node iff `(payload_present = True ∧ s = FULL)` or `(payload_present = False ∧ s = EMPTY)`.

**Case 2** (`root ≠ r`): the vote targets a descendant of this block.

- Let `(r', s') = anc(root, slot(B_r))`.
- The vote supports `(r, s)` iff `r' = r` and `(s = PENDING` or `s = s')`.

*Spec reference: `is_supporting_vote` in `fork-choice.md`.*

**Definition 11** (Payload timeliness). The execution payload for block with root r is *timely* if both:

1. `r ∈ store.payloads` (the node locally has the validated payload), and
2. more than `PTC_SIZE / 2` entries in `payload_timeliness_vote[r]` are True.

Blob data is *available* under the analogous condition on `payload_data_availability_vote[r]`.

*Spec reference: `is_payload_timely`, `is_payload_data_available` in `fork-choice.md`.*

**Definition 12** (Weight). The *weight* of a fork-choice node `(r, s)` is:

- If `s ≠ PENDING` and `slot(B_r) + 1 = current_slot`: return 0. This is the zero-return rule for the immediately previous slot. The FULL/EMPTY decision is delegated to the tiebreaker (Definition 13).
- Otherwise: return `attest_weight(r, s) + boost_weight(r, s)`, where:
  - `attest_weight(r, s)` is the sum of effective balances of all active, unslashed, non-equivocating validators whose vote supports `(r, s)` per Definition 10.
  - `boost_weight(r, s)` is the proposer boost score if the boost vote (a vote with `root = proposer_boost_root`, `slot = current_slot`, `payload_present = False`) supports `(r, s)`, and 0 otherwise.

*Spec reference: `get_weight`, `get_attestation_score` in `fork-choice.md`.*

**Definition 13** (Tiebreaker). For a fork-choice node `(r, s)`, distinguish two regimes by the relation between $\mathit{slot}(B_r)$ and the current slot:

*Regime A — block is not the immediately previous slot's block (`s = PENDING` or `slot(B_r) + 1 ≠ current_slot`).* The tiebreaker is the natural payload-status integer (Definition 2): EMPTY=0, FULL=1, PENDING=2. There is no ambiguity to resolve here — `get_weight` already produces non-zero scores via Definition 12, and the tiebreaker only matters under exact weight ties.

*Regime B — block is the immediately previous slot's block (`s ∈ {FULL, EMPTY}` and `slot(B_r) + 1 = current_slot`).* `get_weight` returns 0 for both `(r, FULL)` and `(r, EMPTY)` under the zero-return rule (Definition 12), so the tiebreaker is the *sole* arbiter:

- If `s = EMPTY`: the tiebreaker value is 1.
- If `s = FULL`: the tiebreaker value is 2 if `should_extend_payload(r)`, else 0.

`should_extend_payload(r)` first checks the hard guard: if `is_payload_verified(store, r)` is False, it returns False immediately. Otherwise, it returns True if any of: (a) the execution payload is timely and blob data is available (Definition 11), (b) `proposer_boost_root` is the zero root, (c) `parent(B_{proposer\_boost\_root}) ≠ B_r`, (d) `is_parent_node_full(store, B_{proposer\_boost\_root})`.

The effect in Regime B: when `should_extend_payload(r)` holds, FULL(2) > EMPTY(1) → FULL wins. Otherwise, EMPTY(1) > FULL(0) → EMPTY wins. Regime A is therefore a "natural-order" pass-through; Regime B is the load-bearing case where the PTC primary path and the proposer-based fallbacks resolve FULL vs EMPTY for the immediately previous slot.

*Spec reference: `get_payload_status_tiebreaker`, `should_extend_payload` in `fork-choice.md`.*

#### The two-chain structure

A fundamental structural property of ePBS is that the protocol maintains two logically distinct chains that can diverge in their slot structure:

- The **consensus chain** is a sequence of beacon blocks, linked by `parent_root`, one per slot (or with gaps for missed slots). This chain always advances slot by slot: Block(98) → Block(99) → Block(100). Every beacon block exists regardless of whether its builder reveals an execution payload.
- The **execution chain** is a sequence of execution payloads, linked by `parent_hash` (`payload.parent_hash = state.latest_block_hash`). This chain only advances when a builder actually reveals a valid execution payload. If a builder withholds, the execution chain has a gap at that slot.

The two chains can therefore have different shapes. For example, if slot 98 has an execution payload, slot 99's builder withholds, and slot 100's builder reveals:

```
Consensus chain:   Block(98) ──→ Block(99) ──→ Block(100)
                      ↓              ↓              ↓
Execution chain:  Payload(98)      [gap]       Payload(100)
                      ↑                             |
                      └─────────────────────────────┘
                        Payload(100).parent_hash = Payload(98).block_hash
```

Block(100)'s execution payload builds on slot 98's execution state, skipping slot 99 entirely — because slot 99's builder never revealed, so the execution state never advanced past slot 98. Meanwhile, Block(100) on the consensus side has `parent_root = Block(99).root` — it builds on slot 99's beacon block, which did exist (it just had no payload).

The mechanism that tracks where the execution chain is at any given time is `state.latest_block_hash`. This field is only updated by `apply_parent_execution_payload` (Helper 12b, line 29), which is called inside `process_parent_execution_payload` (Helper 12c) at the beginning of the next block's `process_block`. If a builder withholds, no subsequent block declares FULL for that slot, `latest_block_hash` is not updated, and still points to the last slot that had a revealed execution payload. The next builder reads this value and builds on that older execution state.

The two chains reconnect through `bid.parent_block_hash`: each block declares which execution chain tip its builder was building on. This declaration is what `get_parent_payload_status` (Helper 3 / Definition 4) reads to determine FULL vs. EMPTY, and what `get_ancestor` (Algorithm 3) walks through when attributing descendant votes to the correct FULL/EMPTY branch. The consensus chain is linear, but the execution chain branches at every FULL/EMPTY split — and the fork-choice tree (Definition 6) represents both possibilities.

### 12.2 Honest Behavior

Before presenting the fork-choice algorithms, we define how each actor in the protocol is expected to behave honestly. These behaviors are the assumptions that the fork-choice rule is designed around: the algorithms in Section 12.3 produce correct outcomes *because* honest actors follow these rules. Understanding who does what — and when — is a prerequisite for understanding why the algorithms work.

**A note on what "honest behavior" means in practice.** The code below specifies **event-driven behavior**: each `@Upon` decorator declares the condition under which a piece of code executes. In practice, a validator runs **client software** (such as Lighthouse, Prysm, Teku, Lodestar, or Nimbus) that has a scheduler watching the clock and network events. When a condition is met — the clock reaches a specific point in the slot, or a message arrives on the gossip network — the client internally performs the corresponding steps. A validator is honest if its client follows these steps; a validator is dishonest (Byzantine) if its client deviates — for example, by signaling `data.index = 1` without having seen the execution payload, withholding an attestation, or signing two conflicting blocks.

**A note on `# (Ax)` markers.** Inline comments of the form `# (A1)`, `# (A2)`, … `# (A6)` tag specific lines of the handlers below where a *behavioural assumption* about honest actors fires. The assumptions are stated by the companion overview document (§8) and discharged by Lemmas H1–H6 (plus Assumption H7) in §12.4. In brief: A1 = honest builder bid-reveal consistency (Lemma H3); A2 = same-slot attesters signal `data.index = 0` (Lemma H1); A3 = non-same-slot attesters signal FULL/EMPTY consistently (Lemma H2); A4 = honest PTC members report observation, not validity (Lemma H4); A5 = honest builder withholds when block is not local head (Lemma H5); A6 = honest builder reveals cautiously (Assumption H7 — a non-normative strengthening). The markers are forward references to §12.4.

**Pedagogical abstractions.** The handlers below use a small set of helper names that do not appear in the Gloas specs as named functions. Each one abstracts spec prose or a generic operation, for readability. The mapping is:

| Pedagogical name                                                                                | Spec source                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                              |
| ----------------------------------------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `collect_valid_bids(state, slot)`                                                             | "Listen to the `execution_payload_bid` gossip global topic and save an accepted `signed_execution_payload_bid` from a builder. The block proposer MAY obtain these signed messages by other off-protocol means." (`validator.md`, §"Signed execution payload bid")                                                                                                                                                                                                                                                                                                                                                                                                                                |
| `select_one_bid(bids)`                                                                        | "Select one bid …" (`validator.md`, §"Signed execution payload bid")                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                 |
| `aggregate_ptc_votes(slot - 1)`                                                               | "The proposer MUST aggregate all payload attestations with the same data into a given `PayloadAttestation` object." (`validator.md`, §"Payload attestations")                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                       |
| `construct_beacon_block(state, body)`, `construct_beacon_block_body(state, slot, **kwargs)` | `construct_beacon_block` is standard `BeaconBlock` container construction (`slot`, `proposer_index`, `parent_root`, `state_root`, `body`), where `state_root` is filled in after a pre-signature `state_transition` with `validate_result=False`. `construct_beacon_block_body` populates the unchanged-from-pre-ePBS body fields (`randao_reveal`, `eth1_data`, `graffiti`, slashings, `attestations`, `deposits`, `voluntary_exits`, `sync_aggregate`, `bls_to_execution_changes`) per `phase0/validator.md` and accepts the ePBS-new fields (`signed_execution_payload_bid`, `payload_attestations`, `parent_execution_requests`) as keyword overrides. |
| `broadcast_data_column_sidecars(envelope, builder)`                                           | `get_data_column_sidecars(beacon_block_root, slot, cells_and_kzg_proofs)` from `builder.md` plus per-sidecar publication to the `data_column_sidecar_{subnet_id}` gossip subnets (`p2p-interface.md`).                                                                                                                                                                                                                                                                                                                                                                                                                                                                                           |
| `is_head_of_chain(block, store)`                                                              | Shorthand for `get_head(store).root == hash_tree_root(block)` — the "head of the builder's chain" check from `builder.md` §"Honest payload withheld messages".                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                     |
| `is_attester_for_slot(state, validator_index, slot)`                                          | Shorthand for "the validator is assigned to one of the slot's attestation committees", determined via `get_committee_assignment(state, compute_epoch_at_slot(slot), validator_index)` (`phase0/validator.md`).                                                                                                                                                                                                                                                                                                                                                                                                                                                                                       |
| `has_beacon_block_for_slot(store, slot)`, `get_beacon_block_root_for_slot(store, slot)`     | Inline prose checks in `validator.md` §"Constructing the `PayloadAttestationMessage`" ("If the validator has not seen any beacon block for the assigned slot, do not submit a payload attestation").                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                |
| `has_execution_payload_envelope(store, root)`                                                 | Inline prose check in `validator.md` §"Constructing the `PayloadAttestationMessage`" ("If a previously seen `SignedExecutionPayloadEnvelope` references the block …"); equivalent to `is_payload_verified(store, root)` in `fork-choice.md`.                                                                                                                                                                                                                                                                                                                                                                                                                                                 |
| `check_blob_data(store, root)`                                                                | Spec function `is_data_available(beacon_block_root)` from `fork-choice.md`; we keep `store` in the signature so reads are visible at the call site.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                |
| `EXECUTION_ENGINE.build_payload(...)`, `EXECUTION_ENGINE.get_payload(payload_id)`           | The execution-engine JSON-RPC method `engine_getPayloadV6` invoked after `notify_forkchoice_updated` (`builder.md` step 3; `fork-choice.md` §`notify_forkchoice_updated`). We collapse the two-step flow to a single conceptual call returning a `(payload, execution_requests, blob_kzg_commitments)` bundle.                                                                                                                                                                                                                                                                                                                                                                              |
| `sign(x)`, `broadcast(x)`                                                                   | Standard BLS signing (via `compute_signing_root` + `bls.Sign` with the appropriate `DOMAIN_*`) and gossip publication. The spec uses domain-specific named signers (`get_attestation_signature`, `get_proposer_preferences_signature`, `get_payload_attestation_message_signature`, `get_block_signature`); we use `sign(x)` as a uniform shorthand for these and `broadcast(x)` for the gossip layer.                                                                                                                                                                                                                                                                                 |
| `get_safe_execution_block_hash(store)`                                                        | Implementation-defined helper invoked by `prepare_execution_payload` (`fork-choice.md` §`notify_forkchoice_updated`).                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                             |
| `(r, status)` tuple form                                                                      | Pedagogical shorthand for `ForkChoiceNode(root=r, payload_status=status)` (`fork-choice.md` line 104). The algorithms below alternate between the tuple form and the spec container form for compactness.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                            |
| `FULL`, `EMPTY`, `PENDING`                                                                | Shorthand for the spec constants `PAYLOAD_STATUS_FULL`, `PAYLOAD_STATUS_EMPTY`, `PAYLOAD_STATUS_PENDING` (`fork-choice.md` lines 74–76).                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                        |
| `slot(B)`, `parent(B)`, `state(B)`, `root(B)`, `head(chain)`, `child(chain, B)`                                           | Math notation: "the slot of B", "the parent block of B", "the post-state of B", "the hash tree root of B" — correspond to `B.slot`, `store.blocks[B.parent_root]`, `store.block_states[hash_tree_root(B)]`, and `hash_tree_root(B)` respectively. `head(chain)` is the head block of a canonical chain (the latest block in the sequence); `child(chain, B)` is `B`'s unique canonical child on `chain` (the block whose `parent_root` equals `B`'s hash-tree root), introduced in Definition 4b.                                                                                                                                                                                                                                                                                                                                                                                                                                               |
| `time_in_slot`, `T_ATT`, `T_PTC`                                                          | Client-scheduler variables abstracted from the spec deadlines `ATTESTATION_DUE_BPS_GLOAS` and `PAYLOAD_ATTESTATION_DUE_BPS` (`validator.md`); `time_in_slot` is the wall-clock offset within the current slot.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                   |
| `@Upon(condition)` decorator                                                                  | Framing convention for event-driven handlers, with the informal English event syntax inside the parentheses (e.g., "received SignedBeaconBlock for current slot"). The spec describes these triggers in prose; we use `@Upon` for compactness.                                                                                                                                                                                                                                                                                                                                                                                                                                                         |
| `validator` runtime object                                                                    | A client-side process-state record (distinct from the spec `Validator` container). Fields used: `validator.index` (the validator's assigned index in `state.validators`), `validator.fee_recipient` (execution-layer payment address), `validator.gas_limit` (preferred gas), `validator.wants_external_builder` (local preference flag toggled per slot).                                                                                                                                                                                                                                                                                                                                   |
| `builder` runtime object                                                                      | A client-side process-state record (distinct from the spec `Builder` container). Fields used: `builder.index` (the assigned builder index), `builder.bid_amount` (Gwei amount the builder will offer this slot), `builder.privkey` (the builder's BLS private key), `builder.stored_payload`, `builder.stored_requests` (the payload + execution-requests bundle stashed at bid time and re-used at reveal time per Lemma H3).                                                                                                                                                                                                                                                               |

The @Upon handler names themselves (`attest`, `propose`, `submit_bid`, `reveal_payload`, `ptc_vote`, `broadcast_preferences`) are this document's framing for the honest behaviour described in `validator.md` and `builder.md`; they are not spec function names.

There are three roles a validator can play under ePBS: **attester**, **proposer**, and **PTC member**. A validator may hold multiple roles in the same slot (e.g., an attester who is also a PTC member). In addition, **builders** are a separate class of staked participants who are not validators.

#### Honest Attester Behavior

```python
@Upon(time_in_slot == T_ATT and is_attester_for_slot(state, validator.index, slot))
def attest(validator, slot, state, store):
    head = get_head(store)
    head_block = store.blocks[head.root]
    current_epoch = get_current_epoch(state)
    data = AttestationData()
    data.slot = slot
    data.beacon_block_root = head.root
    data.source = state.current_justified_checkpoint
    data.target = Checkpoint(epoch=current_epoch, root=get_block_root(state, current_epoch))
    if head_block.slot == slot:                                    # Same-slot attestation
        data.index = 0                                             # (A2) signal zero — no payload opinion possible
    else:                                                          # Non-same-slot attestation
        data.index = 1 if head.payload_status == FULL else 0       # (A3) signal FULL/EMPTY consistently
    broadcast(sign(data))
```

The `@Upon` guard fires at T_att (25% of the slot, i.e., 3 seconds) for every validator assigned to the slot's committee. The attester runs the fork-choice function to determine the chain head, then constructs an `AttestationData` with the standard fields (slot, head root, source and target checkpoints). The ePBS-specific behavior is entirely in how `data.index` is set (lines 9–12).

**Lines 9–10: Same-slot attestation.** If the head block is from the attester's own slot (`head_block.slot == slot`, where `head_block = store.blocks[head.root]`), the attester always sets `data.index = 0`. This is because the attestation deadline is at 25% of the slot (T_att = 3s, Definition 1), while the builder has until 75% (T_ptc = 9s) to reveal the execution payload. At the time the attester votes, they cannot know whether the payload will arrive, so they express no opinion. This is the behavior that Lemma 2 relies on: same-slot attesters are payload-neutral because they are *required* to set `data.index = 0`.

**Lines 11–12: Non-same-slot attestation.** If the head block is from a previous slot (the attester's slot had no block proposed, so their head fell back to an older block), the attester can observe whether that block's execution payload was delivered. They read the payload status from the fork-choice result: if `get_head` returned a FULL node, set `data.index = 1`; if EMPTY, set `data.index = 0`. This is the behavior that Lemma 8 relies on: non-same-slot attesters provide the attestation weight that resolves FULL vs. EMPTY when the immediately next slot is missed.

*Spec reference: attestation `data.index` rules in `validator.md`.*

#### Honest Proposer Behavior

```python
@Upon(time_in_slot == 0 and is_proposer(state, validator.index))
def propose(validator, slot, state, store):
    parent_root = hash_tree_root(state.latest_block_header)
    # Select a bid (external builder or self-build)
    bids = collect_valid_bids(state, slot)                         # From gossip + off-protocol
    if bids and validator.wants_external_builder:
        selected_bid = select_one_bid(bids)                        # Spec does not prescribe selection
    else:                                                          # Self-build path
        # Derive execution-engine head pointers per fork-choice.md.
        # The finalized beacon block itself does not commit to any execution-hash
        # value beyond its bid's two fields; we use parent_block_hash because it
        # identifies an execution payload already integrated into the execution
        # chain (the block_hash field commits only to a payload the finalized
        # block's builder would have revealed, which under self-build does not
        # apply). The spec leaves the exact choice to the proposer.
        finalized_block = store.blocks[store.finalized_checkpoint.root]
        finalized_block_bid = finalized_block.body.signed_execution_payload_bid.message
        finalized_block_hash = finalized_block_bid.parent_block_hash
        safe_block_hash = get_safe_execution_block_hash(store)
        payload_id = prepare_execution_payload(
            store, state, safe_block_hash, finalized_block_hash,
            validator.fee_recipient, EXECUTION_ENGINE,
        )
        payload, execution_requests, blob_kzg_commitments = EXECUTION_ENGINE.get_payload(payload_id)
        bid = ExecutionPayloadBid(
            builder_index=BUILDER_INDEX_SELF_BUILD,
            value=0, execution_payment=0, slot=slot,
            parent_block_hash=state.latest_block_hash,
            parent_block_root=parent_root,
            block_hash=payload.block_hash,
            prev_randao=get_randao_mix(state, get_current_epoch(state)),
            fee_recipient=validator.fee_recipient,
            gas_limit=payload.gas_limit,
            blob_kzg_commitments=blob_kzg_commitments,
            execution_requests_root=hash_tree_root(execution_requests),
        )
        selected_bid = SignedExecutionPayloadBid(bid, bls.G2_POINT_AT_INFINITY)
    # Construct and broadcast the beacon block. construct_beacon_block populates
    # the unchanged-from-pre-ePBS body fields (randao_reveal, eth1_data, graffiti,
    # slashings, attestations, deposits, voluntary_exits, sync_aggregate,
    # bls_to_execution_changes) per phase0/validator.md.
    body = construct_beacon_block_body(state, slot,
        signed_execution_payload_bid=selected_bid,
        payload_attestations=aggregate_ptc_votes(slot - 1),         # PTC votes from previous slot
        parent_execution_requests=(
            store.payloads[parent_root].execution_requests
            if should_extend_payload(store, parent_root)
            else ExecutionRequests()
        ),
    )
    block = construct_beacon_block(state, body)
    # Pre-signature state-root computation uses a temp SignedBeaconBlock with empty signature
    state_transition(state, SignedBeaconBlock(message=block, signature=BLSSignature()),
                     validate_result=False)
    broadcast(SignedBeaconBlock(block, sign(block)))
```

```python
@Upon(slot in get_upcoming_proposal_slots(state, validator.index))  # Before the slot (optional)
def broadcast_preferences(validator, slot, state):
    preferences = ProposerPreferences(
        dependent_root=get_proposer_dependent_root(state, compute_epoch_at_slot(slot)),
        proposal_slot=slot,
        validator_index=validator.index,
        fee_recipient=validator.fee_recipient,                     # Destination address, not a price
        gas_limit=validator.gas_limit,                             # Preferred max gas for the payload
    )
    broadcast(SignedProposerPreferences(preferences, sign(preferences)))
```

The proposer has two event handlers. The `broadcast_preferences` handler fires before the slot for each upcoming proposal slot and is optional — without it, the gossip network will not forward any builder bids for that slot (the gossip validation rules require that preferences have been seen before forwarding a bid), so the proposer must either self-build or obtain a bid through off-protocol means. `fee_recipient` is the execution-layer address where the proposer wants to receive payment (a destination, not a price), and `gas_limit` is the proposer's preferred maximum gas. Neither field specifies a minimum bid amount; the competitive dimension is `bid.value` on the builder side.

The `propose` handler fires at the beginning of the slot (t = 0). There are two paths for bid selection:

- **External builder** (lines 5–6): The proposer selects one bid from those collected. Bid selection is not prescribed by the spec — it simply says "select one bid." A rational proposer would choose the highest `bid.value`.
- **Self-build** (lines 7–22): If the proposer does not want to use an external builder (or no bids are available), it sets `builder_index = BUILDER_INDEX_SELF_BUILD` with `value = 0` (the proposer does not pay itself) and signature = `G2_POINT_AT_INFINITY` (the BLS identity element, a sentinel meaning "no real signer"). The proposer constructs the execution payload itself via `prepare_execution_payload` (line 8), which internally calls `apply_parent_execution_payload` on a copy of the state when building on FULL — the same local simulation that external builders perform (see the note in the builder's `submit_bid` description).

After selecting a bid, the proposer constructs the beacon block body via `construct_beacon_block_body` (which populates the unchanged-from-pre-ePBS fields like `randao_reveal`, `eth1_data`, etc., and accepts the three ePBS-new fields as overrides). PTC votes from slot N-1 are included in the slot N block via `payload_attestations`, making them part of the permanent on-chain record. The proposer runs `state_transition` with `validate_result=False` over a temporary `SignedBeaconBlock` (with an empty signature placeholder) to compute the state root, then signs and broadcasts the real `SignedBeaconBlock`.

*Spec reference: proposer duties in `validator.md`.*

#### Honest PTC Member Behavior

The four lookup helpers below — `has_beacon_block_for_slot`, `get_beacon_block_root_for_slot`, `has_execution_payload_envelope`, `check_blob_data` — are pedagogical names for the corresponding inline checks in the spec prose at [`validator.md`](https://github.com/ethereum/consensus-specs/blob/master/specs/gloas/validator.md) ("Constructing a payload attestation message"). The spec uses `is_data_available(beacon_block_root)` for the blob-data check; the other three are described in prose rather than as named helpers.

```python
@Upon(time_in_slot == T_PTC and validator.index in get_ptc(state, slot))
def ptc_vote(validator, slot, state, store):
    if not has_beacon_block_for_slot(store, slot):
        return                                                     # Cannot vote without a block
    block_root = get_beacon_block_root_for_slot(store, slot)
    data = PayloadAttestationData()
    data.beacon_block_root = block_root
    data.slot = slot
    if has_execution_payload_envelope(store, block_root):
        data.payload_present = True                                # (A4) report observation — builder revealed
    else:
        data.payload_present = False                               # (A4) report observation — builder did not reveal
    data.blob_data_available = check_blob_data(store, block_root)  # (A4) spec: is_data_available
    msg = PayloadAttestationMessage(
        validator_index=validator.index,
        data=data,
        signature=sign(data),
    )
    broadcast(msg)
```

**Spec timing semantics.** The `@Upon(time_in_slot == T_PTC ...)` clause above models the worst-case (most conservative) firing time. The spec ([`validator.md`](https://github.com/ethereum/consensus-specs/blob/master/specs/gloas/validator.md)) actually specifies a *deadline*, not an exact firing time: a PTC member must broadcast the `PayloadAttestationMessage` *within the first* `PAYLOAD_ATTESTATION_DUE_BPS` (= 75%, i.e., 9 seconds) of the slot. An honest member is therefore allowed to vote as soon as it has decided its vote (e.g., as soon as the payload is observed), but a *cautious* member waits close to the deadline to maximise information — voting earlier risks emitting `payload_present = False` and then having the payload arrive later. The proofs in Section 12.4 depend only on (a) the value the member commits to and (b) the deadline being met; they do not depend on when within `(0, T_PTC]` the broadcast actually happens. We therefore keep the simpler firing-at-deadline form in the handler above and rely on the fact that all proof obligations follow from the worst case.

The `@Upon` guard fires for validators who are PTC members for this slot. A PTC member casts a **witness statement** — a report of what they observed, not a judgment on validity. The PTC vote handler checks payload arrival (`has_execution_payload_envelope`), not execution validity (`verify_execution_payload_envelope`). Since PTC members are validators running full nodes, they do validate the execution payload as part of normal node operation (via `on_execution_payload_envelope`) — but this validation is independent of their PTC duty and is not a precondition for the vote.

**Lines 3–4: No block, no vote.** If the validator has not seen any beacon block for the assigned slot, it returns without voting. The vote would be ignored anyway (Algorithm 9, line 5 checks that the vote's slot matches the block's slot).

**Lines 9–12: Payload observation.** This is the core PTC duty. The member checks whether a `SignedExecutionPayloadEnvelope` referencing this block has been seen. If yes: `payload_present = True`. If not: `payload_present = False`. The member does NOT verify the payload's execution validity — they only check whether the payload arrived on the gossip network and passed the gossip-level validation rules (correct builder index, correct block hash, valid signature). The execution engine validation happens separately in `on_execution_payload_envelope` (Algorithm 8).

**Line 13: Blob data observation.** Similarly, the member reports whether the blob data associated with this block is available. Blobs are large binary data chunks (EIP-4844) used by rollups for temporary data availability. Each blob is split into 128 data columns distributed across the p2p network via subnets — no single node downloads all columns. The PTC member checks whether the columns it is responsible for arrived and pass KZG proof verification against the commitments in the builder's bid (`bid.blob_kzg_commitments`). This signal feeds into `is_payload_data_available` (Helper 5), which is checked alongside `is_payload_timely` in the PTC primary path of `should_extend_payload` (Algorithm 6, line 6). Both must pass for the tiebreaker to favor FULL.

**Why witness statements, not validity judgments?** Requiring PTC members to validate the execution payload would put the execution engine on the critical path of the PTC vote — a 512-member committee that must vote within a tight window. By making PTC votes observational, the protocol decouples the timeliness question ("did the payload arrive?") from the validity question ("is the payload correct?"). Validity is checked asynchronously by each node's execution engine when it runs `verify_execution_payload_envelope` (Helper 12). The fork-choice tree structure enforces validity separately: even if all 512 PTC members vote `payload_present = True`, the FULL branch only exists if `store.payloads` is populated (Lemma 1), which requires the execution engine to accept the payload.

*Spec reference: PTC duty in `validator.md`; `PayloadAttestationMessage` construction in `validator.md`.*

#### Honest Builder Behavior

Builders are a separate class of staked participants — they are **not validators**. They do not attest, do not propose beacon blocks, and do not earn staking yield. Their sole activity is constructing execution payloads and bidding for the right to have them included.

```python
@Upon(received SignedProposerPreferences for upcoming_slot)
def submit_bid(builder, upcoming_slot, state, execution_engine, proposer_preferences):
    # Read the current chain state
    parent_block_hash = state.latest_block_hash                    # Execution chain tip
    parent_block_root = hash_tree_root(state.latest_block_header)  # Consensus chain tip
    # Construct the execution payload via the execution engine. The bundle
    # returned by engine_getPayloadV6 contains the payload plus the execution
    # requests and blob KZG commitments as siblings (see builder.md steps 3, 12, 13).
    payload, execution_requests, blob_kzg_commitments = execution_engine.build_payload(
        parent_hash=parent_block_hash,
        timestamp=compute_time_at_slot(state, upcoming_slot),
        prev_randao=get_randao_mix(state, get_current_epoch(state)),
        fee_recipient=proposer_preferences.fee_recipient,
        gas_limit=proposer_preferences.gas_limit,
    )
    # Fill in the bid fields
    bid = ExecutionPayloadBid(
        parent_block_hash=parent_block_hash,
        parent_block_root=parent_block_root,
        block_hash=payload.block_hash,                             # The commitment
        prev_randao=payload.prev_randao,
        fee_recipient=proposer_preferences.fee_recipient,          # Must exactly match preferences
        gas_limit=proposer_preferences.gas_limit,                  # Must exactly match preferences
        builder_index=builder.index,
        slot=upcoming_slot,
        value=builder.bid_amount,                                  # Amount offered to proposer (Gwei)
        execution_payment=0,                                       # Must be 0 for gossip bids
        blob_kzg_commitments=blob_kzg_commitments,
        execution_requests_root=hash_tree_root(execution_requests),
    )
    signature = get_execution_payload_bid_signature(state, bid, builder.privkey)
    broadcast(SignedExecutionPayloadBid(bid, signature))
    builder.stored_payload = payload                               # (A1) saved for reveal — same object used in reveal_payload
    builder.stored_requests = execution_requests
```

```python
@Upon(received SignedBeaconBlock for current slot)
def reveal_payload(builder, block, store, execution_engine):
    bid = block.body.signed_execution_payload_bid.message
    if bid.builder_index != builder.index:
        return                                                     # Not our bid
    if not is_head_of_chain(block, store):
        return                                                     # (A5) honest withholding when block is not local head
    # (A6) Non-normative cautious-reveal precondition (formalised as Assumption H7):
    #      (i) t_rev >= T_att,
    #      (ii) >= PROPOSER_SCORE_BOOST = 40% real attestation weight observed
    #           for block.root,
    #      (iii) no proposer equivocation by block.proposer_index for block.slot.
    #      The spec only mandates A5; A6 is the strengthening the proofs in this
    #      document rely on.
    block_root = hash_tree_root(block)
    state = store.block_states[block_root]
    # Construct the payload
    envelope = ExecutionPayloadEnvelope(
        payload=builder.stored_payload,                            # (A1) same payload object as in submit_bid
        execution_requests=builder.stored_requests,
        builder_index=builder.index,
        beacon_block_root=block_root,
        parent_beacon_block_root=block.parent_root,
    )
    # Sign and broadcast
    signature = get_execution_payload_envelope_signature(state, envelope, builder.privkey)
    signed_envelope = SignedExecutionPayloadEnvelope(envelope, signature)
    broadcast(signed_envelope)
    broadcast_data_column_sidecars(envelope, builder)              # Builder's responsibility under ePBS
```

The builder has two event handlers, corresponding to the two phases of builder activity.

**`submit_bid`** fires when the builder observes a `SignedProposerPreferences` for an upcoming slot. The builder constructs the actual execution payload **before** submitting the bid — `bid.block_hash` (line 18) is the hash of an already-built payload, not a promise to build one later. The builder must have the payload ready at bid time because `block_hash` is a binding commitment: when the builder later reveals the execution payload, `verify_execution_payload_envelope` (Helper 12, line 16) asserts that the revealed execution payload's hash matches the committed `block_hash`. The `fee_recipient` and `gas_limit` must exactly match the proposer's preferences — the gossip network enforces this. `bid.value` (line 24) is the competitive dimension: the amount the builder offers to pay the proposer. The builder stores the payload locally (lines 30–31) for later reveal.

A subtlety in lines 3–4: the builder reads `state.latest_block_hash` to determine which execution chain tip to build on. But at this point, `apply_parent_execution_payload` has not yet run for the current slot's block (it runs inside `process_block` for the *next* block). How does the builder know the correct value? The answer is that the builder calls `prepare_execution_payload` (spec: [`validator.md`](https://github.com/ethereum/consensus-specs/blob/master/specs/gloas/validator.md)), which locally runs `apply_parent_execution_payload` on a **copy** of the state when building on FULL. This local simulation advances `latest_block_hash` on the copy, computes the correct withdrawals, and determines the correct execution chain head — all before the block is proposed. The network's eventual `process_block` produces the same deterministic result. If building on EMPTY, the simulation is skipped and `latest_block_hash` retains its stale value.

**`reveal_payload`** fires when the builder sees any beacon block for the current slot. The builder first checks whether the block contains its own bid (line 4) — if not, it returns. It then retrieves the payload it already built (line 13) and wraps it in a `SignedExecutionPayloadEnvelope`.

- **Lines 5–6: Honest withholding (cautious-reveal strategy).** The spec ([`builder.md`](https://github.com/ethereum/consensus-specs/blob/master/specs/gloas/builder.md)) states the minimal rule: an honest builder MAY choose not to reveal if the block "was not timely and thus it is not the head of the builder's chain." This is the loose rule captured by Lemma H5. **The proofs in this document assume a strictly stronger strategy** — the **cautious-reveal** rule formalised as Assumption H7: an honest builder broadcasts the payload only after (i) $t_{\mathrm{rev}} \geq T_{\mathrm{att}}$, (ii) ≥ `PROPOSER_SCORE_BOOST = 40%` of real attestation weight for the block has been observed, and (iii) no proposer equivocation by `block.proposer_index` for `block.slot` is visible. The 40% threshold is calibrated to the proposer boost so that, under β < 20%, the slot-N+1 proposer cannot reorg slot N via boost alone (Lemma 7 Part (b)). The equivocation check prevents revealing before a late equivocation surfaces (the builder might reveal at t ≈ 0.5s purely on the boost, but a competing block B' could arrive at t ≈ 1s and trigger slashing). Waiting until ~t ≈ 4–5s (when attestations have propagated and the equivocation window is largely closed) gives the builder time to observe both attestation support and equivocations before making the irreversible decision to broadcast. If the builder withholds because it detected an equivocation, `process_proposer_slashing` clears the `BuilderPendingPayment` when the equivocation evidence is included on-chain, protecting the builder from epoch-boundary payment charges. Without H7 (under H5 alone), the proofs only deliver the weaker conclusion that the FULL outcome *can* prevail; with H7, they deliver the stronger conclusion that it *does* prevail at slot N+1 under an honest next-slot proposer (Lemma 9, second claim).
- **Lines 7–8: State source.** The builder reads `store.block_states[block_root]` — the post-state produced by `on_block` (Algorithm 7) when nodes processed the beacon block. This is the same state that `on_execution_payload_envelope` (Algorithm 8, line 5) uses for verification.
- **Lines 9–14: Envelope construction.** The builder wraps its stored execution payload and execution requests into an `ExecutionPayloadEnvelope`, referencing the beacon block root and the parent beacon block root. The `builder_index` must match the bid's `builder_index`.
- **Lines 14–15: Signing.** The payload is signed using `get_execution_payload_envelope_signature`. Every node that later processes this payload will verify this signature via `verify_execution_payload_envelope_signature` (Helper 12, line 5).
- **Line 17: Data column sidecars.** Under ePBS, the builder — not the proposer — is responsible for broadcasting blob data column sidecars.

*Spec reference: bid construction in `builder.md`; payload construction in `builder.md`; honest payload withholding in `builder.md`.*

#### Slot Timeline Summary (All Actors)

The following table summarizes the complete timeline of actions within a single slot, combining all three validator roles and the builder:

| Time                              | Actor         | Trigger                                  | Action                                                                                                                          |
| --------------------------------- | ------------- | ---------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------- |
| Before slot                       | Proposer      | `@Upon(upcoming proposal slot)`        | Broadcasts `SignedProposerPreferences` (optional)                                                                             |
| Before slot                       | Builder       | `@Upon(received preferences)`          | Constructs execution payload, broadcasts `SignedExecutionPayloadBid`                                                          |
| t = 0 (0%)                        | Proposer      | `@Upon(time_in_slot == 0)`             | Broadcasts `SignedBeaconBlock` with builder's bid and prev-slot PTC votes                                                     |
| t = 0+                            | All nodes     | `@Upon(received block)`                | Run `on_block` (Algorithm 7): process beacon block, initialize PTC scorecards                                                 |
| t = T_att (25%)                   | Attesters     | `@Upon(time_in_slot == T_ATT)`         | Broadcast attestations with `data.index` payload signal                                                                       |
| t ∈ (0, T_ptc)                   | Builder       | `@Upon(received block with bid)`       | Broadcasts `SignedExecutionPayloadEnvelope` once cautious-reveal conditions are met (Assumption H7)                           |
| t ∈ (0, T_ptc)                   | All nodes     | `@Upon(received payload)`              | Run `on_execution_payload_envelope` (Algorithm 8): validate execution payload, populate `store.payloads`                    |
| t ∈ (0, T_ptc] (deadline at 75%) | PTC members   | `@Upon(within T_PTC, before deadline)` | Broadcast `PayloadAttestationMessage` (witness statement). Spec deadline is t ≤ T_PTC; cautious members wait close to T_PTC. |
| t = T_ptc+                        | All nodes     | `@Upon(received PTC vote)`             | Run `on_payload_attestation_message` (Algorithm 9): update PTC scorecard                                                      |
| Next slot, t = 0                  | Next proposer | `@Upon(time_in_slot == 0)`             | Includes aggregated PTC votes in next block; runs `get_head`                                                                  |

*Spec reference: timing constants `ATTESTATION_DUE_BPS_GLOAS = 2500`, `PAYLOAD_ATTESTATION_DUE_BPS = 7500` in `validator.md`.*

The following diagram shows the same timeline as the table above, with the complete message flow between actors:

![Slot lifecycle](https://raw.githubusercontent.com/ethereum/epbs-security-analysis/master/figures/fig2-spec-update-v30.png)

*The slot lifecycle under ePBS. Each column represents an actor, color-coded: Proposer (orange), Builder (violet), Attesters with Consensus Client and Execution Client (green/blue/teal), and PTC Members (red). PTC Members are a separate column linked to Attesters by a "subsampled from" arrow, since they are selected from the slot's attestation committees. The Builder is a separate staked participant, not a validator. Lifelines are solid when the actor is active and dotted when idle. The key structural feature is the separation between block publication (Phase 1, Proposer to all participants) and payload delivery (Phase 3, Builder to all participants): the beacon block contains only the builder's bid, and the actual execution payload arrives separately.*

### 12.3 Algorithms

The following algorithms operate on a store (Definition 8) and a set of known blocks.

#### Algorithm 1: get_head

```python
def get_head(store):
    blocks = get_filtered_block_tree(store)                        # Dict[Root, BeaconBlock]
    head = (store.justified_checkpoint.root, PENDING)
    while True:
        children = get_node_children(store, blocks, head)          # Definition 6
        if not children:
            return head
        head = max(children, key=lambda c: (                       # Definitions 12, 13
            get_weight(store, c), c.root, get_payload_status_tiebreaker(store, c)
        ))
```

This is the main fork-choice function: given the current store, it returns the node that the validator considers the chain head. The algorithm starts at the justified checkpoint (line 3) and walks forward through the tree, one level at a time. At each level, it asks "what are the children of the current node?" (line 5) and picks the child with the highest weight (lines 8–10). If there are no children, the current node is the head (lines 6–7). The tree alternates between two kinds of branching: from a PENDING node, the children are FULL and EMPTY (the payload-status split); from a FULL or EMPTY node, the children are the next blocks in the chain (as PENDING nodes). This means the algorithm alternates between deciding "was the payload revealed?" and "which block comes next?" at every step. Lines 8–10 break ties using three keys in order: (1) attestation weight (Definition 12), (2) block root (lexicographic, for determinism), (3) the tiebreaker (Definition 13), which matters only for the immediately previous slot where weight is 0 for both FULL and EMPTY.

*Spec reference: `get_head` in `fork-choice.md`.*

#### Algorithm 2: get_node_children

```python
def get_node_children(store, blocks, node):
    r, s = node
    if s == PENDING:
        children = [(r, EMPTY)]                                    # EMPTY always exists
        if is_payload_verified(store, r):
            children.append((r, FULL))                             # FULL only if payload validated
        return children
    else:                                                          # s in {FULL, EMPTY}
        return [                                                   # Definition 4
            (child_root, PENDING)
            for child_root in blocks.keys()
            if blocks[child_root].parent_root == r
               and get_parent_payload_status(store, blocks[child_root]) == s
        ]
```

Given a node in the fork-choice tree, this function returns its children. There are exactly two cases depending on the node's payload status:

- **PENDING node** (lines 3–7): The block exists but we have not yet decided whether its payload was delivered. The node always has an EMPTY child (line 4) — the "payload was not delivered" world always exists as a possibility. It has a FULL child (lines 5–6) only if `is_payload_verified(store, r)` returns true — i.e., the node has locally received and validated the execution payload (`r ∈ store.payloads`). This is the enforcement mechanism: if the builder withheld the payload or the payload was invalid, `store.payloads` is never populated (see Algorithm 8), so the FULL branch simply does not exist in the tree and cannot be selected.
- **FULL or EMPTY node** (lines 8–13): The payload status is decided. The children are PENDING nodes for the next blocks in the chain — specifically, for all blocks B' whose parent is this block and whose `parentStatus` (Definition 4) matches this node's status. A block declaring "I built on my parent's FULL version" becomes a child of the FULL node only; a block declaring "I built on the EMPTY version" becomes a child of the EMPTY node only.

*Spec reference: `get_node_children` in `fork-choice.md`.*

#### Algorithm 3: get_ancestor

```python
def get_ancestor(store, root, target_slot):
    block = store.blocks[root]
    if block.slot <= target_slot:
        return (root, PENDING)                                     # Block at or before target
    parent = store.blocks[block.parent_root]
    while parent.slot > target_slot:                               # Walk back toward target slot
        block = parent
        parent = store.blocks[block.parent_root]
    return (block.parent_root, get_parent_payload_status(store, block))  # Read block's declaration about parent
```

This function answers the question: "in the chain of a given block, what happened at a given target slot — which block was there, and was it FULL or EMPTY?" In one sentence: **what did the first block after the target slot declare about the target?** More precisely: the function finds the first block in the chain whose slot is strictly after the target, and reads what that block declared about the block at the target slot. The answer comes from the chain structure itself — from the permanent `parentStatus` declarations that each block makes about its parent — not from any BeaconState, PTC vote, or attestation.

The reason the function asks the *descendant* rather than the target block itself is that a block does not know its own payload status at creation time. Block B is proposed at `t = 0`, but the builder reveals the execution payload later (up to `t = 9s`). B cannot declare "I am FULL" — that has not happened yet when B is created. But the next block in the chain (say C, one or more slots later) does know: C's builder read `state.latest_block_hash` and set `bid.parent_block_hash` accordingly. If B's execution payload was revealed, C's hash matches B's committed hash → FULL. If not → EMPTY. The function simply reads this declaration.

If the block itself is at or before slot t (line 3), it is its own ancestor and is returned as a PENDING node (line 4) — no FULL/EMPTY determination is needed because we are asking about the block's own slot or a future slot, and no descendant has declared anything yet.

Otherwise (lines 5–9), the function walks backward through the chain by following parent pointers. It maintains two variables: B (the current block) and P (B's parent). It keeps walking (lines 6–8) until P is at or before the target slot t. At that point, B is the first block in the chain whose slot is strictly after t, and P is at or before t. Line 9 returns `(parent.root, get_parent_payload_status(store, block))` — the block at the target slot, and what the first descendant past the target declared about it via `parentStatus` (Definition 4). No BeaconState is consulted; only block-level bid hashes are read.

The function is used by `is_supporting_vote` (Algorithm 4, Case 2) to determine which FULL/EMPTY branch a descendant block passed through at an earlier slot.

*Spec reference: `get_ancestor` in `fork-choice.md`.*

#### Algorithm 4: is_supporting_vote

```python
def is_supporting_vote(store, node, vote):
    r, s = node
    block = store.blocks[r]
    if vote.root == r:                                             # Case 1: vote for this block
        if s == PENDING:
            return True                                            # Any vote supports PENDING
        assert vote.slot >= block.slot                             # Votes cannot target future blocks
        if vote.slot == block.slot:
            return False                                           # Same-slot: no payload opinion
        return (s == FULL) if vote.payload_present else (s == EMPTY)
    else:                                                          # Case 2: vote for a descendant
        anc_root, anc_status = get_ancestor(store, vote.root, block.slot)
        return anc_root == r and (s == PENDING or s == anc_status)
```

This function answers the question: "does vote m count toward the weight of fork-choice node (r, s)?" A vote is a tuple `(slot, root, payload_present)` — it says "at this slot, I believe this block root is the head, and here is my payload opinion." The function has two cases depending on whether the voter voted directly for block r, or for a descendant of r.

**Case 1** (lines 4–10): The voter voted for this exact block (`m.root = r`).

- Lines 5–6: If we are asking about the PENDING node, the vote always counts. Every vote for a block supports the block's existence, regardless of payload opinion.
- Line 7: Assert that the vote's slot is not before the block's slot (a vote cannot target a block from the future relative to the vote's own slot — this would be an invalid state).
- Lines 8–9: If we are asking about FULL or EMPTY, but the voter is a same-slot voter (`m.slot == slot(B)`), the vote does **not** count. This is the key ePBS rule: same-slot attesters cast their votes before the builder reveals (Definition 1), so they have no payload opinion and cannot influence the FULL/EMPTY decision.
- Line 10: If the voter is a non-same-slot voter (`m.slot > slot(B)`), they observed the payload status and signaled it via `payload_present`. The vote counts for FULL if `payload_present = True`, for EMPTY if `payload_present = False`.

**Case 2** (lines 11–13): The voter voted for a descendant of block r (`m.root ≠ r`). The voter did not vote for r directly, but their vote for a later block implies a position on r. Line 12 calls `get_ancestor` (Algorithm 3) to walk back from the voted-for block to r's slot and read which FULL/EMPTY branch the descendant chain passed through. Line 13 then checks: is r the correct ancestor, and does the branch match? If so, the vote supports this node. Note that Case 2 always supports PENDING nodes (the `s = PENDING` disjunct), because any descendant passing through r implies support for r's existence.

*Spec reference: `is_supporting_vote` in `fork-choice.md`.*

#### Algorithm 5: get_weight

```python
def get_weight(store, node):
    r, s = node
    if s != PENDING and store.blocks[r].slot + 1 == get_current_slot(store):
        return 0                                                   # Tiebreaker decides for prev slot
    state = store.checkpoint_states[store.justified_checkpoint]
    score = 0
    for i in get_active_validator_indices(state, get_current_epoch(state)):
        if (not state.validators[i].slashed                        # Slashed validators are excluded
                and i in store.latest_messages
                and i not in store.equivocating_indices
                and is_supporting_vote(store, node, store.latest_messages[i])):
            score += state.validators[i].effective_balance
    if should_apply_proposer_boost(store):                         # Proposer boost as synthetic vote
        boost_message = LatestMessage(
            slot=get_current_slot(store),
            root=store.proposer_boost_root,
            payload_present=False,
        )
        if is_supporting_vote(store, node, boost_message):
            score += get_proposer_score(store)
    return score
```

This function computes the total attestation weight behind a fork-choice node (r, s). The weight determines which child `get_head` (Algorithm 1) selects at each step of the traversal.

**Lines 3–4: The zero-return rule.** If the node is a FULL or EMPTY node (`s ≠ PENDING`) and the block is from the immediately previous slot (`slot + 1 = current_slot`), the function returns 0 immediately. This is because no validator can have a meaningful payload opinion yet: same-slot attesters voted before the builder revealed, and non-same-slot attesters (from the current slot) have not attested yet. The FULL vs. EMPTY decision for this slot is delegated entirely to the tiebreaker (Definition 13), which uses PTC votes instead of attestation weight.

**Lines 5–12: Attestation score.** For all other nodes, the function iterates over every active, unslashed, non-equivocating validator and checks whether their latest vote supports this node (via `is_supporting_vote`, Algorithm 4). If it does, their effective balance is added to the score. Two important points: (1) the scan covers ALL validators' latest votes, not just the committee of this block's slot — a vote for block Y at slot 99 contributes weight to every ancestor node Y passes through, via Case 2 of `is_supporting_vote`; (2) the effective balances come from the checkpoint state (line 4), not the block's own state.

**Lines 13–20: Proposer boost.** If a block was received on time in the current slot, the proposer boost acts as a synthetic vote: the function constructs a fake vote `(current_slot, proposer_boost_root, False)` and checks whether it supports this node via `is_supporting_vote`, just like a real vote. If it does, a fixed boost score is added. The `payload_present = False` in the synthetic vote means the boost supports PENDING and EMPTY but never FULL — this is intentional because the boost is applied before the builder reveals.

*Spec reference: `get_weight`, `get_attestation_score` in `fork-choice.md`.*

#### Algorithm 6: should_extend_payload

```python
def should_extend_payload(store, root):
    if not is_payload_verified(store, root):
        return False                                               # Hard guard: payload must exist
    proposer_root = store.proposer_boost_root
    return (
        (is_payload_timely(store, root) and is_payload_data_available(store, root))
        or proposer_root == Root()                                 # No block yet: default FULL
        or store.blocks[proposer_root].parent_root != root         # Boost is for another branch
        or is_parent_node_full(store, store.blocks[proposer_root]) # Boost block declares parent FULL
    )
```

This function decides whether the fork-choice tiebreaker (Definition 13) should favor FULL over EMPTY for the immediately previous slot's block. It is called only when the zero-return rule (Algorithm 5, lines 3–4) applies — meaning attestation weight is 0 for both FULL and EMPTY — so this function is the sole decision-maker for the most recent block's execution payload status.

**Line 2–3: Hard guard.** Before any PTC or fallback logic, the function checks `is_payload_verified(store, root)` — i.e., whether `root ∈ store.payloads`. If the payload was never received or failed verification, the function returns False immediately and the tiebreaker favors EMPTY. This hard guard is the outermost gate: no amount of PTC votes or fallback conditions can override it. It ensures that `should_extend_payload` never returns True for a payload that the local node has not verified.

**Line 6: The PTC primary path.** If the PTC voted that the execution payload is timely (more than half of PTC members voted `payload_present = True`) and the blob data is available (`is_payload_data_available`), the function returns True — the payload was delivered on time, so FULL should win.

**Lines 7–9: Fallback conditions.** These handle cases where the PTC did not reach a positive majority, but the protocol can still infer that FULL is the correct choice. Each condition independently returns True:

- Line 7: No block has been received yet for the current slot (`proposer_boost_root` is the zero root). Defaulting to FULL gives the builder more time to deliver before the next proposer commits to the EMPTY path.
- Line 8: A block was received, but it is not a child of the block in question — it is on a different branch entirely. The PTC vote is irrelevant to this branch.
- Line 9: A block was received, it is a child of this block, and it declares that it built on the FULL version (`is_parent_node_full`). The next proposer already committed to FULL, so the tiebreaker should agree.

These fallbacks limit the damage a malicious PTC can cause (see the adversarial table in Section 12.4): even if every PTC member lies and votes against a valid execution payload, the FULL path often still wins through conditions 7–9.

*Spec reference: `should_extend_payload` in `fork-choice.md`.*

#### Algorithm 7: on_block

```python
def on_block(store, signed_block):
    block = signed_block.message
    assert block.parent_root in store.block_states
    if is_parent_node_full(store, block):                           # FULL parent: payload must exist
        assert is_payload_verified(store, block.parent_root)
    assert get_current_slot(store) >= block.slot                    # Block cannot be from the future
    assert block.slot > compute_start_slot_at_epoch(store.finalized_checkpoint.epoch)
    finalized_checkpoint_block = get_checkpoint_block(
        store, block.parent_root, store.finalized_checkpoint.epoch,
    )
    assert store.finalized_checkpoint.root == finalized_checkpoint_block
    state = copy(store.block_states[block.parent_root])             # Always use block_states
    root = hash_tree_root(block)
    state_transition(state, signed_block, validate_result=True)     # Includes process_parent_execution_payload
    store.blocks[root] = block
    store.block_states[root] = state
    store.payload_timeliness_vote[root] = [None] * PTC_SIZE          # Initialize PTC scorecard
    store.payload_data_availability_vote[root] = [None] * PTC_SIZE
    notify_ptc_messages(store, state, block.body.payload_attestations)
    record_block_timeliness(store, root)
    update_proposer_boost_root(store, root)
    update_checkpoints(store, state.current_justified_checkpoint, state.finalized_checkpoint)
    compute_pulled_up_tip(store, root)                              # Eager unrealized justification/finality
```

This function is called when a new beacon block arrives over the network. It processes the block at the consensus layer and updates the store. Execution payload verification happens separately in Algorithm 8 (`on_execution_payload_envelope`), but the *parent's* execution effects are now applied inside `process_block` (via `process_parent_execution_payload` — see Helper 11) during this call.

**Lines 3–5: Parent payload verification.** The function asserts that the parent block's state exists in the store (line 3). If the block declares its parent was FULL (`is_parent_node_full` returns True), line 5 asserts that the parent's execution payload has been verified and stored in `store.payloads` — i.e., `is_payload_verified(store, block.parent_root)` must hold. If the parent's execution payload was never received or was invalid, this assertion fails and the block is rejected. This is the mechanism that enforces the two-world model: a block cannot claim FULL ancestry without the payload actually existing.

**Lines 6–8: Validity checks.** The block cannot be from the future (line 6), must be after the finalized slot (line 7), and must descend from the finalized checkpoint (line 8).

**Line 9: Pre-state selection.** The starting state is always `store.block_states[block.parent_root]` — the post-state of the parent block after `process_block`. There is no separate FULL/EMPTY state selection: the parent's execution effects (if any) are applied by `process_parent_execution_payload` *inside* `state_transition` at line 11, not by selecting a different pre-state.

**Line 11:** Runs the full state transition (`process_block`) on the pre-state. This now includes `process_parent_execution_payload` as its first step (see Helper 11), which applies the parent's execution-layer effects (EL requests, builder payment settlement, `latest_block_hash` update) if the block declares FULL. The remaining steps validate consensus fields (proposer signature, attestations, slashings, etc.).

**Lines 12–15:** Stores the block and its post-state in the store. Lines 14–15 initialize empty PTC vote arrays (`payload_timeliness_vote`, `payload_data_availability_vote`) for this block — these arrays will be filled later as PTC members broadcast their votes (Algorithm 9) or as the next block includes aggregated PTC attestations (line 16).

**Lines 16–20:** Processes included PTC attestations from the previous slot (`notify_ptc_messages`), records block timeliness, applies proposer boost, updates checkpoints, and eagerly computes the pulled-up unrealized justification/finality for the new block.

*Spec reference: `on_block` in `fork-choice.md`.*

#### Algorithm 8: on_execution_payload_envelope

```python
def on_execution_payload_envelope(store, signed_envelope):
    envelope = signed_envelope.message
    assert envelope.beacon_block_root in store.block_states
    assert is_data_available(envelope.beacon_block_root)           # Blob columns verified
    state = store.block_states[envelope.beacon_block_root]
    verify_execution_payload_envelope(state, signed_envelope, EXECUTION_ENGINE)  # EL verification
    store.payloads[envelope.beacon_block_root] = envelope          # Enables FULL node
```

This function is called when the builder's execution payload arrives over the network (constructed and broadcast as described in Section 12.2, Builder Phase 2), separately from the beacon block. It verifies the payload and stores it — but does not apply execution-layer state effects. Those effects are applied later, when the next block's `process_parent_execution_payload` runs inside `process_block`.

**Line 3:** Asserts that the beacon block this execution payload belongs to has already been processed by `on_block`. The execution payload cannot be verified in isolation — it needs the consensus-only state for reference.

**Line 4:** Asserts that the blob data (EIP-4844 data columns) associated with this execution payload is available. Without blob data, the execution payload cannot be fully validated.

**Line 5:** Reads the state `block_states[root]` (no copy needed — verification does not mutate state).

**Line 6:** Verifies the execution payload against the EL: checks bid consistency (block hash, RANDAO, gas limit, builder index), verifies withdrawals, validates via the execution engine. No state mutation or payment processing happens here.

**Line 7:** Stores the verified payload in `store.payloads`. This single write is the gate that controls whether the FULL node exists: Algorithm 2 (line 5) checks `is_payload_verified(store, r)` (i.e., `r ∈ store.payloads`) to decide whether the FULL child node is created. If `verify_execution_payload_envelope` at line 6 fails, line 7 is never reached, `store.payloads` is never populated, and the FULL node never appears in the fork-choice tree.

*Spec reference: `on_execution_payload_envelope` in `fork-choice.md`.*

#### Algorithm 9: on_payload_attestation_message

```python
def on_payload_attestation_message(store, msg, is_from_block):
    root = msg.data.beacon_block_root
    assert root in store.block_states
    state = store.block_states[root]
    if msg.data.slot != state.slot:
        return                                                     # Vote must match block's slot
    ptc = get_ptc(state, msg.data.slot)
    # The same validator can occupy multiple PTC positions because
    # compute_balance_weighted_selection samples with replacement (Helper 30).
    ptc_indices = [i for i, v_idx in enumerate(ptc) if v_idx == msg.validator_index]
    assert len(ptc_indices) > 0
    if not is_from_block:
        assert msg.data.slot == get_current_slot(store)            # Gossip: current slot only
        assert is_valid_indexed_payload_attestation(
            state,
            IndexedPayloadAttestation(
                attesting_indices=[msg.validator_index],
                data=msg.data,
                signature=msg.signature,
            ),
        )
    for idx in ptc_indices:                                        # Write vote into every position the validator holds
        store.payload_timeliness_vote[root][idx] = msg.data.payload_present
        store.payload_data_availability_vote[root][idx] = msg.data.blob_data_available
```

This function is called when a PTC member's vote arrives (constructed as described in Section 12.2, PTC Member Behavior), either from the gossip network (during the current slot) or from a block that included aggregated PTC attestations from the previous slot. PTC votes do not carry attestation weight — they do not appear in `get_weight` (Algorithm 5). Instead, they populate the PTC scorecard (`payload_timeliness_vote`, `payload_data_availability_vote`) that feeds into `is_payload_timely` (Definition 11), which feeds into `should_extend_payload` (Algorithm 6), which feeds into the tiebreaker (Definition 13). The chain of influence is: PTC vote → scorecard → timeliness check → tiebreaker → FULL vs. EMPTY decision for the immediately previous slot.

**Lines 2–11: Validation.** The vote must reference a known block (line 3), the vote's slot must match the block's slot (line 5), and the voter must occupy at least one PTC position for that slot (lines 7–11). Because `compute_balance_weighted_selection` (Helper 30) samples with replacement, a single validator can hold multiple PTC slots; we collect every position the voter holds into `ptc_indices`.

**Lines 12–18: Source distinction.** Votes arrive from two sources: gossip (broadcast by PTC members during the slot) and block inclusion (aggregated by the next proposer). Gossip-received votes (`is_from_block = False`) are subject to additional checks: the vote must be for the current slot — stale PTC votes are rejected — and the signature must be valid via `is_valid_indexed_payload_attestation`. Block-included votes skip these checks because block processing already validated them.

**Lines 19–21: Scorecard update.** The voter's `payload_present` and `blob_data_available` signals are written into *every* PTC position the voter holds. Once enough entries are True (more than `PTC_SIZE / 2`), `is_payload_timely` returns True and the tiebreaker favors FULL.

*Spec reference: `on_payload_attestation_message` in `fork-choice.md`.*

### 12.4 Properties

This section establishes the formal properties of the ePBS fork-choice rule, organized in three layers. We begin with properties of the honest behavior specifications (Section 12.2) — things that follow directly from how honest actors construct their messages. We then prove structural properties of the fork-choice algorithms (Section 12.3) — invariants that hold regardless of who is honest. Finally, we combine both layers to prove what the fork-choice rule produces under honest behavior.

#### Property table

The [overview document](https://github.com/ethereum/epbs-security-analysis/blob/master/README.md) (Section 4) restricts its property list to four externally observable claims — those an observer with full visibility of network messages and on-chain state can verify, without inspecting any node's internal state. The following table maps each of the four overview properties to the formal lemmas proved in this section:

| Property                                  | Statement                                                                                                                                                                                                                                                                                                    | Proved by                                                                                                                                                                                                                                                                                                                             |
| ----------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| P1: Unconditional payment to proposer     | Timely-proposed bid settles to the proposer within at most two epochs of slot N; payment is single (no double-charge); builder cannot avoid paying                                                                                                                                                           | Lemma 11 (combining Lemmas 9 and 10 — settlement via Path A or Path B) + Lemma 12 (single payment — Path A clears Path B) + Lemma 12b (solvency precondition at bid admission) + Lemma 21 (index correctness) + 2-epoch bound via the Helper 14 ring buffer (upper-half write at bid time, lower-half quorum check one epoch later) |
| P2: Builder revealing protection          | Honest reveal under H7 ⇒$\mathit{bid}(B).\mathit{block\_hash}$ is in the payload hash chain (Definition 4c) of the canonical beacon chain — equivalently, $\mathit{block\_status}(\mathit{canonical}, B) = \text{FULL}$ — under honest PTC majority and honest slot-N+1 proposer                      | Lemma 7 Part (a) (FULL/EMPTY tiebreaker selects FULL via PTC primary path) + Lemma 7 Part (b) (anti-reorg argument under β < 20% and PROPOSER_SCORE_BOOST = 40%) + Lemma 18 (honest builder's bid declares FULL when local store has the parent's execution payload) — combined, deliver the "hash in payload hash chain" claim     |
| P3: Builder withholding protection        | Honest withhold ⇒ neither builder charged nor proposer paid; Byzantine alone cannot meet quorum                                                                                                                                                                                                             | Lemma 16 + Lemma 20 case (c) (Case 1 of P3: no-weight withhold — the 60% = 40% + 20% calibration ensures honest participation is required) + Lemma 22 part (iv) (Case 2 of P3: equivocation-driven withhold —`process_proposer_slashing` clears the IOU before the epoch-boundary check)                                          |
| P4: Data availability for chain inclusion | If a payload hash$h$ is in the payload hash chain (Definition 4c) of the canonical beacon chain — equivalently, $h = \mathit{bid}(B).\mathit{block\_hash}$ for some FULL block $B$ — then the corresponding execution payload is available and valid, and its associated blob data is also available | Lemma 18b (DA confirmed ⟺ hash on chain), under honest super-majority + honest slot-N+1 proposer. P4 claims the (⇒) direction (hash on chain ⇒ DA confirmed); Lemma 18b's biconditional strengthens this                                                                                                                           |

Several internal mechanisms and non-externally-observable facts underlie the four properties without being properties themselves. They are still proved formally here because the property proofs reference them:

| Internal mechanism                                      | Statement                                               | Proved by                                                                                                                                                                                           |
| ------------------------------------------------------- | ------------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| PTC bounds proposer power                               | PTC overrides a single malicious proposer's EMPTY       | Lemma 14 + adversarial table below — feeds the proof of P2 (revealing protection) since the PTC primary path or fallback (c) is what surfaces the FULL declaration on chain after an honest reveal |
| Builder solvency at bid time                            | Builder cannot overbid across concurrent slots          | Lemma 12b — precondition for P1 (unconditional payment); without solvency, the on-chain withdrawal could not be honoured                                                                           |
| Bid commitments are binding                             | Revealed execution payload must match bid hash          | Lemma H3 — a behavioural fact about honest builders treated as Assumption A1 (and lemma-discharged here), feeding the proof of P2                                                                  |
| Missed-slot FULL/EMPTY resolution                       | Attestation weight resolves when next slot is missed    | Lemma 8 + Lemma 28 (empty-slot state preservation, stated in §12.5) — analysed as a corner case of the fork-choice rule in §5 Phase 5 of the overview                                            |
| Two-phase block processing                              | Consensus advances without execution payload            | Helper 11 (process_block) structure; Algorithm 7 vs Algorithm 8 separation                                                                                                                          |
| Execution validity gates FULL-node existence (per node) | FULL node exists iff payload validated                  | Lemma 1 — a per-node invariant that supports P4 (data availability for chain inclusion) externally                                                                                                 |
| Same-slot payload neutrality (weight computation)       | Same-slot attesters cannot contribute FULL/EMPTY weight | Lemma 2 (structural) + Lemma H1 (behavioural) — internal to the fork-choice weight, not externally observable                                                                                      |
| PTC votes are witness statements (honest behavior)      | Honest PTC reports observation, not validity            | Lemma H4 — discharged as a behavioural assumption (A4); not externally checkable from a single PTC vote                                                                                            |

#### Discharging the overview's assumptions

The [overview document](https://github.com/ethereum/epbs-security-analysis/blob/master/README.md) (Section 8) states explicit assumptions — structural (S-prefix), behavioral (A-prefix), and algorithmic (G-prefix) — that its proof sketches depend on. This document provides the code and proofs that discharge every assumption. The tables below show the correspondence.

**Structural assumptions** become standing hypotheses in this document:

| Overview assumption                  | Standing hypothesis in this document                                                                                                              |
| ------------------------------------ | ------------------------------------------------------------------------------------------------------------------------------------------------- |
| S1 (synchrony Δ < T_att)            | Stated explicitly in each lemma that needs it (e.g., Lemma 7 Part (b), Lemma 18b)                                                                 |
| S2 (per-committee β < 20%)          | Derived from balance-weighted committee sampling: Lemmas 25–26 (concentration), used by Lemma 7, Lemma 16, Lemma 18b, Lemma 24                   |
| S3 (PTC β < 50%)                    | Used by Lemma 7 Part (a), Lemma 14 — corresponds to the PTC `payload_timeliness_vote` / `payload_data_availability_vote` majority thresholds |
| S4 (per-property honesty hypotheses) | Stated as explicit lemma hypotheses where invoked (e.g., "honest slot-N+1 proposer" in Lemma 7, Lemma 9, Lemma 18, Lemma 18b)                     |

**Behavioral assumptions** are discharged by proving the claimed behavior from the `@Upon` handler code in Section 12.2:

| Overview assumption                              | Discharged by                                                                                                                                                                                                                                                                         |
| ------------------------------------------------ | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| A1 (builder bid-reveal consistency)              | Lemma H3 — the `submit_bid` handler stores the payload (line 14) and `reveal_payload` uses the same object (line 5)                                                                                                                                                              |
| A2 (same-slot attesters signal zero)             | Lemma H1 — the `attest` handler's `if head_block.slot == slot` branch (with `head_block = store.blocks[head.root]`)                                                                                                                                                            |
| A3 (non-same-slot attesters signal consistently) | Lemma H2 — the `attest` handler's `else` branch (lines 11–12)                                                                                                                                                                                                                   |
| A4 (PTC reports observation, not validity)       | Lemma H4 — the `ptc_vote` handler calls `has_execution_payload_envelope` (line 9), not the execution engine                                                                                                                                                                      |
| A5 (honest builder withholding is conditional)   | Lemma H5 — the `reveal_payload` handler's only early-return path (lines 5–6, counting from `def`)                                                                                                                                                                               |
| A6 (honest builder reveals cautiously)           | Assumption H7 — cautious-reveal strategy:$t_{\mathrm{rev}} \geq T_{\mathrm{att}}$ AND ≥ 40% real attestation weight observed AND no proposer equivocation, before broadcast. Used by Lemmas 7, 9, 11, 17 to deliver concrete anti-reorg / Path-A-fires guarantees under β < 20%. |

**Algorithmic assumptions** are discharged by providing the complete algorithm/helper code and proving the claimed properties:

| Overview assumption | Claim                                                                                                                 | Discharged by                                                                                                                                                                                                                                         |
| ------------------- | --------------------------------------------------------------------------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| G-Struct            | `get_node_children` returns FULL/EMPTY for PENDING, PENDING for FULL/EMPTY                                          | Algorithm 2 (lines 3–13) + Lemma 1 (FULL iff `store.payloads`) + Lemma 3 (tree alternation)                                                                                                                                                        |
| G-Weight            | `get_weight` returns 0 for FULL/EMPTY of immediately previous slot                                                  | Algorithm 5 (lines 3–4) + Lemma 4 (zero-return rule)                                                                                                                                                                                                 |
| G-Vote.1            | Same-slot votes excluded from FULL/EMPTY weight; non-same-slot votes attributed by `payload_present`                | Algorithm 4 Case 1 (lines 4–10) + Lemma 2 (same-slot exclusion)                                                                                                                                                                                      |
| G-Vote.2            | Descendant votes attributed by chain-structural `parentStatus`, not voter's `payload_present`                     | Algorithm 4 Case 2 (lines 11–13) + Lemma 5 (descendant attribution)                                                                                                                                                                                  |
| G-Tiebreaker        | `get_head` consults `should_extend_payload` via tiebreaker only between children from `get_node_children`       | Algorithm 1 (lines 8–10):`max` over `children` only; Definition 13 maps to `should_extend_payload`                                                                                                                                             |
| G-PayAttest         | `process_attestation` increments `payment.weight` exactly when same-slot + new flag + amount > 0                  | Helper 15 (lines 28–32): the three conditions are checked in order                                                                                                                                                                                   |
| G-PayEpoch          | `process_builder_pending_payments` queues iff `weight ≥ quorum`                                                  | Helper 14 (lines 2–5): the `if payment.weight >= quorum` condition                                                                                                                                                                                 |
| G-Solvency          | `can_builder_cover_bid` accounts for all obligations                                                                | Helper 6 calls Helper 20 (`get_pending_balance_to_withdraw_for_builder`), which sums both `builder_pending_withdrawals` and `builder_pending_payments`                                                                                          |
| G-BidAdmit          | `process_execution_payload_bid` creates exactly one IOU at the correct slot index when accepted                     | Helper 8 (`process_execution_payload_bid`): admission preconditions on lines 6–14, IOU creation on lines 17–26 (gated by `bid.value > 0`)                                                                                                       |
| G-Settle            | `apply_parent_execution_payload` invokes `settle_builder_payment` with the correct payment index when parent FULL | Helper 12c (`process_parent_execution_payload`) gates the call on `bid.parent_block_hash == parent_bid.block_hash`; Helper 12b (`apply_parent_execution_payload`) computes the index by epoch and calls Helper 12a (`settle_builder_payment`) |
| G-Slashing          | `process_proposer_slashing` clears the matching `BuilderPendingPayment` within the 2-epoch window                 | Helper 22 (`process_proposer_slashing`) part (iv) of Lemma 22 — the slashing-driven clearing path                                                                                                                                                  |
| G-BlockAdmit        | `on_block` rejects a block declaring FULL ancestry without `store.payloads[parent_root]`                          | Algorithm 7 (`on_block`) line 5 — the `is_payload_verified(store, block.parent_root)` hard assertion; cf. Lemma 13                                                                                                                               |

#### Properties of honest behavior

The following properties follow directly from the `@Upon` handlers in Section 12.2. They do not depend on the fork-choice algorithms — they are consequences of how honest actors construct their messages. These properties serve as the formally verified assumptions that the fork-choice proofs rely on.

---

**Lemma H1** (Same-slot attesters always signal zero). *If an honest attester's fork-choice head is a block from the attester's own slot (i.e., `head_block.slot == slot` where `head_block = store.blocks[head.root]`), then the attester sets data.index = 0.*

This property ensures that same-slot attesters never express a payload opinion, which is the behavioral basis for Lemma 2 (same-slot attesters are payload-neutral in the fork-choice weight).

*Proof.* By the `attest` handler (Section 12.2, Attester), the branch guard checks `if head_block.slot == slot` (with `head_block = store.blocks[head.root]`). When this condition is true, the body unconditionally executes `data.index = 0`. No other branch is reachable in that arm. ∎

---

**Lemma H2** (Non-same-slot attesters signal consistently with fork-choice). *If an honest attester's fork-choice head is a block from a previous slot (i.e., `head_block.slot < slot` where `head_block = store.blocks[head.root]`), then the attester sets data.index = 1 if head.payload_status == FULL, and data.index = 0 otherwise.*

This property ensures that non-same-slot attesters faithfully report what their fork-choice tells them. It is the behavioral basis for Lemma 8 (attestation fallback for missed slots): honest non-same-slot attesters provide weight to the correct FULL or EMPTY branch.

*Proof.* By the `attest` handler (Section 12.2, Attester), when `head_block.slot != slot` (the `else` branch, with `head_block = store.blocks[head.root]`), the body executes `data.index = 1 if head.payload_status == FULL else 0`. The signal is fully determined by `head.payload_status`. ∎

---

**Lemma H3** (Builder bid-reveal consistency). *If an honest builder submits a bid with block_hash = H and later reveals an execution payload for that bid, the revealed execution payload has block_hash = H.*

This property ensures that honest builders honor their commitments. It is the behavioral basis for asserting that `verify_execution_payload_envelope` (Helper 12, line 16) will not fail for honest builders: the assertion `payload.block_hash == bid.block_hash` holds because the builder stored the payload at bid time and reveals the same object.

*Proof.* By the `submit_bid` handler (Section 12.2, Builder), the builder constructs a payload via the execution engine (lines 7–13), sets `bid.block_hash = payload.block_hash` (line 18), and stores the payload locally: `builder.stored_payload = payload` (line 30). By the `reveal_payload` handler, the builder sets `envelope.payload = builder.stored_payload` (line 12). Since `stored_payload` is the same object whose `block_hash` was used in the bid, and the builder does not modify it between submission and reveal, `envelope.payload.block_hash == bid.block_hash`. ∎

---

**Lemma H4** (PTC votes reflect observation, not validity). *An honest PTC member sets payload_present = True if and only if it has locally observed a SignedExecutionPayloadEnvelope referencing the block. The PTC vote is conditioned on payload arrival, not on execution validity. (The member's node does validate the execution payload via `on_execution_payload_envelope` as part of normal operation, but this is independent of the PTC vote handler.)*

This property formalizes the "witness statement" design: PTC votes are observational reports, not validity judgments. It is the basis for the separation between PTC influence (tiebreaker) and execution validity (store.payloads gate, Lemma 1): even if all PTC members vote True, the FULL node only exists if the execution engine accepts the execution payload.

*Proof.* By the `ptc_vote` handler (Section 12.2, PTC), counting from `def ptc_vote(...)` as line 1, lines 8–11: `if has_execution_payload_envelope(store, block_root): data.payload_present = True; else: data.payload_present = False`. The function `has_execution_payload_envelope` checks whether the payload was received on the gossip network — it does not invoke the execution engine or call `verify_execution_payload_envelope`. The handler contains no call to any execution validation function. ∎

---

**Lemma H5** (Honest builder withholding is conditional). *An honest builder withholds the execution payload (does not broadcast the payload) only if the beacon block containing its bid is not the head of the builder's chain.*

This property bounds the conditions under which an honest builder may fail to reveal. It distinguishes honest withholding (a rational response to a non-canonical block) from dishonest withholding (strategic execution payload suppression).

*Proof.* By the `reveal_payload` handler (Section 12.2, Builder), counting from `def reveal_payload(...)` as line 1: line 5 checks `if not is_head_of_chain(block, store)` and line 6 returns without broadcasting. This is the only early-return path after the builder-index check (lines 3–4). If `is_head_of_chain(block, store)` is true, the function proceeds to construct, sign, and broadcast the payload (lines 7 onward). ∎

---

**Lemma H6** (Proposer preferences constrain builder bids). *A gossip-propagated builder bid can only be forwarded by the network if a matching SignedProposerPreferences has been broadcast, and the bid's fee_recipient and gas_limit exactly match the preferences.*

This property establishes that builders cannot unilaterally choose the payment destination or gas limit — they must match what the proposer specified. It follows from the gossip validation rules rather than the honest behavior handlers directly, but it constrains the inputs that the honest behavior code operates on.

*Proof.* By the gossip validation rules in `p2p-interface.md`: an `execution_payload_bid` message is `[IGNORE]`d if no `SignedProposerPreferences` for the bid's slot has been seen, and `[REJECT]`ed if `bid.fee_recipient != preferences.fee_recipient` or `bid.gas_limit != preferences.gas_limit`. These rules are enforced by every honest node in the gossip network before forwarding. Consequently, any bid that reaches the proposer via gossip satisfies both constraints. (Off-protocol bids are not subject to these rules.) ∎

---

**Assumption H7** (Honest builder reveals via the cautious strategy). *Throughout this assumption and below, a* **real attestation** *for a block $B$ means an `Attestation` (or `SingleAttestation`) signed by a validator in the slot committee for `slot(B)` with `data.beacon_block_root = B.root` — i.e., an actual validator-cast vote, as distinct from the synthetic proposer-boost vote that `get_weight` injects via Algorithm 5 lines 13–20.*

*An honest builder broadcasts the `SignedExecutionPayloadEnvelope` for a beacon block $B$ with bid `bid` only after the builder has observed (in its local store): (i) a single non-equivocating beacon block at slot $\mathit{slot}(B)$ containing `bid` (no proposer equivocation by $B.\mathit{proposer\_index}$ for that slot), and (ii) at least `PROPOSER_SCORE_BOOST = 40%` of a slot committee's effective balance in real attestations supporting $B$. Equivalently, an honest builder withholds whenever either condition fails.*

This is the **cautious-reveal** strategy: the builder waits until the beacon block has accumulated at least as much real attestation weight as a single proposer boost is worth, and until no proposer-side equivocation evidence has surfaced. The 40% threshold is calibrated to `PROPOSER_SCORE_BOOST = 40` so that, under the per-committee Byzantine bound β < 20% (Section 12.4 adversarial model), the slot-N+1 proposer cannot reorg slot N via boost alone: the boost grants at most 40% synthetic weight to a competing branch, but the committed branch has ≥ 40% real weight plus whatever further honest attestations accrue.

**On the role of the 40% threshold vs. the consensus-layer bound.** The 40% in H7 is the *builder-side observable* — the threshold the builder can locally measure before deciding to reveal. It is not directly the bound that proves canonicity; that role is played by the synchrony-derived $(1 - \beta)\, W$ lower bound on $\mathit{weight}((B.\mathit{root}, \mathrm{PENDING}))$ at slot N+1 (see Lemma 7 Part (b)). The two thresholds play complementary roles: H7's 40% is necessary for the builder to confidently commit to revealing under partial synchrony, while $(1 - \beta)\, W$ is the consensus-layer bound used to argue that the boost-only reorg fails. Under full synchrony, the 40% threshold is *automatically met* well before $T_{\mathrm{ptc}}$ (since $(1 - \beta) > 0.8 > 0.4$ under β < 0.2), and the consensus-layer bound takes over.

The spec (`builder.md`) does not strictly mandate this strategy — it only says an honest builder *may* withhold when the block "is not the head of the builder's chain". H5 captures that minimal rule. H7 is a strictly stronger behavioural assumption that the proofs in this section rely on whenever they need a concrete anti-reorg or Path-A-fires guarantee. Without H7, the proofs only deliver the weaker Lemma H5 conclusion (payload is broadcast iff block is locally head), which is insufficient to argue that the FULL outcome survives into the canonical chain.

*Exhibit of the strategy.* H7 is a **strengthened behavioural assumption**, not a derivation from the spec; we exhibit the strategy rather than derive it. Its content is precisely the commitment that an honest builder runs, before the broadcast at `reveal_payload` lines 7 onward, two extra checks beyond the spec-mandated `is_head_of_chain(block, store)` (Lemma H5):

1. **Real-attestation observation:** the builder polls its local store for attestations supporting `block.root` and waits until the cumulative real attestation weight (Definition: real attestation = validator-cast vote, excluding the synthetic proposer-boost vote) reaches `PROPOSER_SCORE_BOOST = 40%` of a slot committee's effective balance.
2. **Equivocation-evidence absence:** the builder polls its local store for any `SignedBeaconBlockHeader` pair that would constitute a `ProposerSlashing` against `block.proposer_index` at `block.slot`, and proceeds only if none is present.
3. **Delivery-window separator:** the builder waits until at least $T_{\mathrm{att}}$ before treating its no-equivocation check as binding — i.e., $t_{\mathrm{rev}} \geq T_{\mathrm{att}}$. This closes the synchrony gap that Lemma 7 Part (b) relies on. The argument: an equivocation $B''$ that could split slot-$N$ honest attesters at $T_{\mathrm{att}}$ must have been broadcast strictly before $T_{\mathrm{att}} - \Delta$ (otherwise synchrony delivers it too late to influence the vote at $T_{\mathrm{att}}$). At reveal time $t_{\mathrm{rev}} \geq T_{\mathrm{att}}$, anything broadcast before $t_{\mathrm{rev}} - \Delta \geq T_{\mathrm{att}} - \Delta$ is necessarily visible in the builder's view. So a splitting equivocation is detectable by the builder if it exists. Note: in practice $t_{\mathrm{rev}} \geq T_{\mathrm{att}} + \Delta$ is forced anyway by condition (i) — observing $\geq 40\%$ real attestation weight requires waiting for honest votes cast at $T_{\mathrm{att}}$ to propagate, which under synchrony $\Delta$ takes until at least $T_{\mathrm{att}} + \Delta$.

If any check fails before $T_{\mathrm{ptc}}$, the builder withholds (early-return path). The `reveal_payload` pseudocode in Section 12.2 captures the spec's minimal rule (`is_head_of_chain`) at lines 5–6; the three H7 checks are layered on top of that and constitute the strategy's defining content. Because H7 is an assumption, not a theorem, there is no formal proof — only the explicit statement of the strategy that honest builders are assumed to follow. Compare A6 in the [overview](https://github.com/ethereum/epbs-security-analysis/blob/master/README.md) (§8), where this is stated explicitly as a non-normative strengthening. ∎

*Remark on dischargeability.* H7 cannot be discharged from the spec alone; it can only be discharged by (i) augmenting `builder.md` with the cautious-reveal rule, or (ii) showing empirically that production builder implementations adopt it. Either route would convert H7 from a behavioural assumption into a derivable lemma. Under the current spec, every Lemma in §12.4–§12.5 that cites H7 is conditional on builders adopting the cautious strategy; without it, those lemmas degrade to the weaker H5-only conclusions noted in the surrounding remarks.

*Remark.* Under H7, the builder's strategic exposure is bounded: if the cautious checks pass, the reveal cannot be reorged at slot N+1 by proposer boost alone (under β < 20%). Conversely, if the builder cannot get the checks to pass before the PTC deadline at $T_{\mathrm{ptc}}$, withholding is the rational choice: the bid IOU recorded by `process_execution_payload_bid` (Helper 8) settles via Path B at the epoch boundary if the beacon block was widely attested (Lemma 10), and via no path at all otherwise (Lemma 16) — so the builder pays at most `bid.value` and only when the block was canonical.

---

#### Structural properties of the fork-choice algorithms

The central object of this section is `get_head` (Algorithm 1): the function that every node runs to determine the canonical chain. `get_head` traverses the fork-choice tree starting from the justified checkpoint, alternating at each block between two decisions — "was the execution payload delivered?" (the FULL/EMPTY split) and "which block comes next?" (the block-selection step). To make these decisions, it relies on `get_weight` (Algorithm 5) for attestation-based scoring, `is_supporting_vote` (Algorithm 4) for classifying individual votes, `should_extend_payload` (Algorithm 6) for the PTC-based tiebreaker, and `get_node_children` (Algorithm 2) for determining what branches exist in the tree.

The goal of this section is to prove that this collection of algorithms satisfies the properties that the ePBS protocol relies on for safety. We begin with **structural properties**: invariants that hold purely from the logic of the algorithms, regardless of network conditions, honest/dishonest actors, or timing. These are the foundation — later sections will build on them to reason about honest-case correctness and adversarial resilience.

The structural properties we establish are:

1. **Execution validity controls node existence** (Lemma 1): the FULL node for a block exists in the fork-choice tree if and only if the execution payload was locally received and validated. This is the mechanism that prevents invalid or withheld payloads from being treated as delivered.
2. **Same-slot attesters are payload-neutral** (Lemma 2): validators who attest in the same slot as a block cannot influence the FULL vs. EMPTY weight for that block. Their votes count for the block's existence (PENDING) and for ancestor nodes, but not for the payload-status decision.
3. **The tree alternates deterministically** (Lemma 3): the fork-choice tree has a fixed structure where PENDING nodes always branch into {FULL, EMPTY} and FULL/EMPTY nodes always branch into PENDING children. There are no other shapes.
4. **The zero-return rule delegates the previous slot to the tiebreaker** (Lemma 4): for the immediately previous slot's block, attestation weight is always zero for both FULL and EMPTY, so the FULL/EMPTY decision is made entirely by the tiebreaker (which reads PTC votes).
5. **Descendant votes are correctly attributed** (Lemma 5): a vote for a descendant block is attributed to the correct FULL or EMPTY branch at every ancestor, based on the chain of `parentStatus` declarations — not on the voter's `payload_present` signal.

---

**Lemma 1** (Execution validity controls branch existence). *For any block with root r, the node (r, FULL) is a child of (r, PENDING) in the fork-choice tree if and only if r ∈ store.payloads.*

This lemma establishes that the FULL branch is gated entirely by local execution validation. If the builder withholds the execution payload, or reveals an invalid execution payload that fails `verify_execution_payload_envelope`, then `store.payloads[r]` is never populated (Algorithm 8 line 7 is never reached), and the FULL child does not exist. This is the fundamental enforcement mechanism of ePBS: it is not PTC votes or attestation weight that prevents an invalid execution payload from being treated as delivered — it is the absence of the FULL branch in the tree.

*Proof.* By Algorithm 2 (get_node_children), when the input node has status PENDING (line 3), the children are constructed as follows: EMPTY is always added (line 4), and FULL is added if and only if `is_payload_verified(store, r)` (i.e., `r ∈ store.payloads`) (lines 5–6). The set `store.payloads` is only modified in one place in the protocol: Algorithm 8 (on_execution_payload_envelope), line 7, which writes `store.payloads[root] = envelope` only after `verify_execution_payload_envelope` completes successfully at line 6. If `verify_execution_payload_envelope` fails (any assertion fails — invalid signature, block hash mismatch, EL rejection, etc.), execution does not reach line 7, `store.payloads[r]` remains absent, and Algorithm 2 line 5 evaluates to false. Therefore, (r, FULL) is a child of (r, PENDING) if and only if `verify_execution_payload_envelope` completed successfully for root r, which is equivalent to `r ∈ store.payloads`. ∎

---

**Lemma 2** (Same-slot attesters are payload-neutral). *Let B be a block with root r. If a vote m has m.root = r and m.slot = slot(B) (a same-slot vote), then m does not support (r, FULL) and does not support (r, EMPTY).*

This lemma establishes that same-slot attesters — who are required to set `data.index = 0` (Section 12.2, Attester lines 9–10) because they vote at 25% of the slot, before the builder reveals at up to 75% (Definition 1) — cannot influence the FULL/EMPTY weight for the block they attested to. Their votes are not "counted as EMPTY" — they are excluded entirely from both sides.

*Proof.* By Algorithm 4 (is_supporting_vote), when `vote.root == r` (line 4, Case 1): if `s == PENDING`, lines 5–6 return True (the vote supports PENDING). If `s ∈ {FULL, EMPTY}`: line 7 asserts `vote.slot >= block.slot` (votes targeting future blocks are invalid). If `vote.slot == block.slot` (the same-slot case covered by this lemma), lines 8–9 return False — the function exits before reaching line 10 (the payload-status check). Therefore, for any same-slot vote with `m.slot = slot(B)` and any `s ∈ {FULL, EMPTY}`, `is_supporting_vote(store, (r, s), m) = False`. Since `get_weight` (Algorithm 5, lines 7–12) only adds a validator's balance when `is_supporting_vote` returns True, the vote contributes zero weight to both (r, FULL) and (r, EMPTY). ∎

*Remark.* Same-slot votes do support (r, PENDING) (Algorithm 4, lines 5–6), which matters when competing blocks exist at the same slot (proposer equivocation). They also support ancestor nodes via Case 2 (Algorithm 4, lines 11–13), where `get_ancestor` determines the FULL/EMPTY attribution from the chain structure, not from the voter's `payload_present` field.

---

**Lemma 3** (Tree structure alternation). *Every node in the fork-choice tree satisfies exactly one of the following:*

- *(a) It is a PENDING node, and its children are a subset of {(r, FULL), (r, EMPTY)} with the same root r.*
- *(b) It is a FULL or EMPTY node, and its children are all PENDING nodes, each with a distinct root.*

This lemma establishes that the fork-choice tree has a rigid alternating structure: PENDING → {FULL, EMPTY} → PENDING → {FULL, EMPTY} → ... There are no PENDING-to-PENDING edges, no FULL-to-FULL edges, and no other shapes. This structure is what makes it possible to reason about the tree as alternating between "which payload status?" and "which block?" at every level.

*Proof.* By Algorithm 2 (get_node_children), lines 3–7 handle the case `s == PENDING` and return only nodes of the form `(r, EMPTY)` or `(r, FULL)` — both with the same root r as the input and with status in {FULL, EMPTY}. This establishes (a). Lines 8–13 handle the case `s ∈ {FULL, EMPTY}` and return only nodes of the form `(block.root, PENDING)` where the roots range over child blocks. The PENDING status is hardcoded at line 10. The roots are distinct because each `r'` corresponds to a different block B' in the filtered tree. This establishes (b). Since Algorithm 2 exhaustively covers all cases (`s = PENDING` or `s ∈ {FULL, EMPTY}`), no other node shapes are possible, and `get_head` (Algorithm 1, line 5) only discovers children through Algorithm 2. ∎

---

**Lemma 4** (Zero-return rule). *For any block B with root r such that slot(B) + 1 = current_slot, and for any s ∈ {FULL, EMPTY}: get_weight(store, (r, s)) = 0.*

This lemma establishes that for the immediately previous slot's block, attestation weight plays no role in the FULL/EMPTY decision. The decision is delegated entirely to the tiebreaker (Definition 13), which reads PTC votes via `should_extend_payload` (Algorithm 6). This is why the PTC exists: for the most recent block, no validator has had the opportunity to cast a meaningful non-same-slot attestation about its payload status, so a separate voting mechanism (the PTC) is needed.

*Proof.* By Algorithm 5 (get_weight), line 3 checks: `s != PENDING and store.blocks[r].slot + 1 == get_current_slot(store)`. When `s ∈ {FULL, EMPTY}`, the first conjunct is true. When `slot(B) + 1 = get_current_slot(store)`, the second conjunct is true. Both conditions hold, so line 4 executes: `return 0`. The function exits before reaching the attestation score computation (lines 5–12) or the proposer boost (lines 13–20). ∎

*Remark.* The tiebreaker (Definition 13) assigns FULL a priority of 2 if `should_extend_payload` returns True, and 0 otherwise. EMPTY always gets priority 1. Since `get_head` (Algorithm 1, lines 8–10) uses the tiebreaker as the third sort key (after weight and root, both of which are equal for FULL and EMPTY of the same block), the PTC-based tiebreaker is the sole decision-maker for the immediately previous slot.

---

**Lemma 5** (Descendant vote attribution). *Let m = (slot, root, payload_present) be a vote where root ≠ r, and let B_r be the block with root r. If is_supporting_vote(store, (r, s), m) = True for s ∈ {FULL, EMPTY}, then the chain from the voted-for block back to B_r passes through the s branch at r. The attribution is determined by the chain of parentStatus declarations (Definition 4), not by m.payload_present.*

This lemma establishes that when a validator votes for a descendant block, their vote's FULL/EMPTY attribution at an ancestor is determined by the chain structure — specifically, by the `bid.parent_block_hash` declarations that each block in the chain makes about its parent (Definition 4). The voter's own `payload_present` field is irrelevant for ancestor attribution; it only matters for the block the voter directly voted for (Case 1 of Algorithm 4).

*Proof.* By Algorithm 4 (is_supporting_vote), when `vote.root ≠ r` (line 11, Case 2): line 12 computes `(anc_root, anc_status) = get_ancestor(store, vote.root, block.slot)`. Line 13 returns `anc_root == r and (s == PENDING or s == anc_status)`. For `s ∈ {FULL, EMPTY}`, the disjunct `s = PENDING` is false, so the condition reduces to `anc_root == r and s == anc_status`. Therefore, the vote supports `(r, s)` if and only if `s == anc_status`, where `anc_status` is the payload status returned by `get_ancestor`. By Algorithm 3 (get_ancestor), `s'` is computed as `parentStatus(D)` where D is the first block in the chain from the voted-for block back to B_r's slot. By Definition 4, `parentStatus(D)` compares `bid(D).parent_block_hash` against `bid(parent(D)).block_hash` — block-level data that is fixed at block creation time. The field `vote.payload_present` does not appear anywhere in lines 11–13 of Algorithm 4 or in Algorithm 3. ∎

*Remark.* This property is crucial for consistency: if two validators both vote for the same descendant block C, their votes are attributed to the same FULL/EMPTY branch at every ancestor, because the attribution depends on C's chain (which is the same for both voters), not on each voter's individual `payload_present` field. This prevents the situation where two votes for the same block end up on opposite sides of a FULL/EMPTY split at an ancestor.

---

#### Properties under honest behavior

The structural properties (Lemmas 1–5) describe invariants of the algorithms themselves, independent of who is honest or dishonest. We now prove what the fork-choice rule produces when the relevant actors follow the protocol as specified in Section 12.2 (Honest Behavior). These properties require **assumptions** about honest behavior — stated explicitly in each lemma and grounded in the pseudocode of Section 12.2 — and they compose the structural lemmas to reason about end-to-end outcomes: given a specific scenario (builder reveals, builder withholds, slot is missed), what does `get_head` return?

---

**Lemma 6** (Builder withholding implies EMPTY). *If the builder for block B with root r does not reveal a valid execution payload, then for all times after B is received, get_head cannot return any node (r, FULL).*

This is the most basic safety property of ePBS: if the builder does not deliver (or honestly withholds — see Section 12.2, Builder Phase 2 lines 5–6), the protocol does not pretend the execution payload exists. The proof is a direct consequence of Lemma 1.

*Proof.* If the builder does not reveal a valid execution payload, `on_execution_payload_envelope` (Algorithm 8) is never called for root r (either the payload never arrives, or `verify_execution_payload_envelope` at line 6 fails). In either case, line 7 is never reached — the payload is never stored in `store.payloads` — so `r ∉ store.payloads`. By Lemma 1, (r, FULL) is not a child of (r, PENDING) in the fork-choice tree. By Lemma 3, the only way to reach a node (r, FULL) in the tree is as a child of (r, PENDING). Since this child does not exist, `get_head` (Algorithm 1) can never visit (r, FULL): the traversal at line 5 calls `get_node_children`, which does not include (r, FULL) in its output, so the `max` at lines 8–10 cannot select it, and if the traversal reaches (r, PENDING) as a leaf candidate, its only child is (r, EMPTY). Therefore, `get_head` cannot return (r, FULL). ∎

*Remark.* This holds regardless of what the PTC votes. Even if every PTC member votes `payload_present = True`, the FULL node does not exist in the tree (Lemma 1), so `should_extend_payload` is never consulted for a node that does not exist — the tiebreaker only runs between children that `get_node_children` actually returns. PTC votes can influence which existing node wins, but they cannot create a node.

---

**Lemma 7** (Honest builder and honest PTC imply FULL, and the block is not reorged). *Let B be a block with root r at slot N. Assume: (1) the builder reveals a valid execution payload for B via the cautious strategy of Assumption H7, so that r ∈ store.payloads and B has accumulated at least 40% of a slot committee's effective balance in real attestations by reveal time; (2) a majority of PTC members are honest and observe the payload, so that more than PTC_SIZE / 2 entries in payload_timeliness_vote[r] are True and more than PTC_SIZE / 2 entries in payload_data_availability_vote[r] are True. Then, when get_head is executed at slot N + 1 (i.e., current_slot = N + 1), the traversal selects (r, FULL) over (r, EMPTY), and B-FULL is canonical for every honest validator running get_head at slot N + 1 — no slot-N+1 proposer can reorg slot N via proposer boost alone (under β < 20%).*

This is the honest-case correctness property: when the builder delivers via cautious reveal (Assumption H7) and the PTC honestly reports it (Section 12.2, PTC lines 9–10), the fork-choice rule correctly picks the FULL path for the most recent block, AND the FULL outcome is robust against a malicious slot-N+1 proposer trying to reorg slot N with their own boost. **Parts (a) and (b) of this lemma together are the formal proof of property P2 (Builder revealing protection) from §4 of the overview** — Part (a) delivers the FULL declaration on chain at slot N+1 (the claim that the execution payload is *included* in the canonical chain), and Part (b) is the anti-reorg guarantee that the beacon block itself is not reorged out.

*Proof.* The proof has two parts: (a) the FULL-vs-EMPTY tiebreaker at slot N+1 selects FULL; (b) the slot-N+1 proposer cannot reorg slot N via boost.

*Part (a) — tiebreaker selects FULL.* Since `r ∈ store.payloads` (assumption (1)), by Lemma 1 both (r, FULL) and (r, EMPTY) are children of (r, PENDING). When `get_head` reaches (r, PENDING), it calls `get_node_children` (Algorithm 2), which returns {(r, EMPTY), (r, FULL)}. It then evaluates `get_weight` and the tiebreaker for each child.

Since `slot(B) + 1 = N + 1 = current_slot`, by Lemma 4, `get_weight(store, (r, FULL)) = 0` and `get_weight(store, (r, EMPTY)) = 0`. Both weights are equal. Both nodes have the same root r. The selection at Algorithm 1 lines 8–10 therefore falls to the tiebreaker (Definition 13).

For (r, FULL): since `s = FULL` and `slot(B) + 1 = current_slot`, Definition 13 evaluates `should_extend_payload(store, r)`. The hard guard at Algorithm 6 line 2 passes (`is_payload_verified(store, r)` is true by (1)). At line 6: `is_payload_timely(store, r)` and `is_payload_data_available(store, r)` both hold by (2). `should_extend_payload` returns True via the PTC primary path. The tiebreaker assigns FULL priority 2.

For (r, EMPTY): Definition 13 assigns priority 1.

At Algorithm 1 lines 8–10, `max` compares `(0, r, 2)` for FULL against `(0, r, 1)` for EMPTY. FULL wins.

*Part (b) — anti-reorg of slot N at slot N+1.*

Let $W$ denote a slot committee's total effective balance (we treat $W_N \approx W_{N+1} = W$ in the arithmetic below; under the validator-set stability assumption these are equal up to rewards/penalties at epoch boundaries, and the slot-to-slot deviation is negligible for the inequalities here).

**Claim.** A malicious slot-N+1 proposer cannot make a sibling block $B'$ (with $B'.\text{parent\_root} = B.\text{parent\_root}$) outweigh $B$ at the $\mathit{parent}(B)$-children level of the slot-N+1 fork-choice traversal via proposer boost alone, under $\beta < 0.2$. Concretely, $\mathit{weight}(\mathit{store}, (B.\mathit{root}, \mathrm{PENDING})) > \mathit{weight}(\mathit{store}, (B'.\mathit{root}, \mathrm{PENDING}))$ at slot N+1, where weights are evaluated by Algorithm 5 (`get_weight`).

(The comparison is at the PENDING level because $\mathit{slot}(B) = N$ and $\mathit{slot}(B') = N+1$; the FULL/EMPTY decision for $B$ is handled separately by Part (a) above, and the zero-return rule of Lemma 4 makes the FULL/EMPTY weights equal anyway.)

**Step 1: Lower bound on $\mathit{weight}((B.\mathit{root}, \mathrm{PENDING}))$ at slot N+1.**

By H7 (cautious reveal), the honest builder broadcasts the payload only after observing no proposer equivocation by `block.proposer_index` for slot N. Combined with the network-model synchrony assumption (every honest node receives any honestly broadcast message within delay $\Delta$, where $\Delta$ is sufficiently small that messages broadcast at one phase deadline arrive before the next; in particular $\Delta < T_{\mathrm{att}} - 0 = 3\,\text{s}$ for slot N's beacon block):

- *(no observable equivocation through $t_{\mathrm{rev}}$)* The builder's H7 check at reveal time $t_{\mathrm{rev}} \geq T_{\mathrm{att}}$ confirms no equivocation has surfaced in the builder's view. Under synchrony, anything broadcast before $t_{\mathrm{rev}} - \Delta$ is in the builder's view at $t_{\mathrm{rev}}$. Since $t_{\mathrm{rev}} \geq T_{\mathrm{att}}$ and $\Delta < T_{\mathrm{att}}$, this means no equivocation was broadcast before $t_{\mathrm{rev}} - \Delta \geq T_{\mathrm{att}} - \Delta > 0$, hence no equivocation was broadcast before $T_{\mathrm{att}} - \Delta$ either. By synchrony, anything broadcast before $T_{\mathrm{att}} - \Delta$ is in every honest validator's view at $T_{\mathrm{att}}$; so an equivocation $B''$ would also be visible to every honest validator at $T_{\mathrm{att}}$ (and would force the builder to withhold), but no such $B''$ exists.
- *(synchrony of B itself)* B was broadcast at $t = 0$ by the slot-N proposer; under $\Delta < T_{\mathrm{att}}$, every honest slot-N validator has B in view by $T_{\mathrm{att}}$.
- *(honest behaviour, Lemma H1)* Each honest slot-N validator runs `get_head` at $T_{\mathrm{att}}$, sees B as the head (B is the only slot-N block in view), and casts a same-slot attestation with `data.beacon_block_root = B.root` and `data.index = 0`.

By Algorithm 4 (`is_supporting_vote`) Case 1 lines 5–6, every such attestation supports the node $(B.\mathit{root}, \mathrm{PENDING})$. (By Lemma 2, these same attestations do *not* support $(B.\mathit{root}, \mathrm{FULL})$ or $(B.\mathit{root}, \mathrm{EMPTY})$ — but that is irrelevant to the present comparison, which is at the PENDING level.)

Writing $\beta < 0.2$ for the per-committee Byzantine bound, the honest contribution to $\mathit{weight}((B.\mathit{root}, \mathrm{PENDING}))$ is at least $(1 - \beta)\, W$, so:

$$
\mathit{weight}((B.\mathit{root}, \mathrm{PENDING})) \;\geq\; (1 - \beta)\, W \;>\; 0.8\, W. \qquad (*)
$$

H7's "$\geq 40\%$ at reveal time" is consistent with $(*)$ but strictly weaker than it; the bound that drives the arithmetic below is $(*)$, which follows from synchrony + no-equivocation (the latter is exactly what H7 enforces as a precondition for revealing).

**Step 2: Upper bound on $\mathit{weight}((B'.\mathit{root}, \mathrm{PENDING}))$ at slot N+1.**

$B'$ is proposed at slot N+1 with `slot(B') = N+1` and `B'.parent_root = B.parent_root`. We bound the support for `(B'.root, PENDING)` at the moment slot N+1 starts (i.e., right after $B'$ has been broadcast and proposer-boosted, but before any slot-N+1 attestations have been cast at $T_{\mathrm{att}}$ of slot N+1):

- **Proposer boost.** $B'$ receives proposer boost. By Algorithm 5 lines 13–20, the synthetic boost vote `(slot = N+1, root = B'.root, payload_present = False)` targets `B'.root`, and Algorithm 4 Case 1 line 5 confirms it supports `(B'.root, PENDING)`. The contribution is `get_proposer_score(store) = PROPOSER_SCORE_BOOST/100 · W = 0.4 W`.
- **Slot-N attestations.** No slot-N attestation supports the node `(B'.root, PENDING)`. We check both cases of Algorithm 4 (`is_supporting_vote`):

  - **Case 1** (line 4) requires `vote.root == B'.root`. Slot-N attestations have `vote.root == B.root` (the slot-N committee voted for $B$ per Step 1), and `B.root ≠ B'.root` (different blocks at different slots). So Case 1 does not match for $B'$.
  - **Case 2** (line 11) computes `(anc_root, _) = get_ancestor(B.root, slot(B'))` and requires `anc_root == B'.root`. By Algorithm 3 line 3, since $\mathrm{slot}(B) = N \leq N+1 = \mathrm{slot}(B')$, `get_ancestor` returns `(B.root, PENDING)` — i.e., `anc_root == B.root ≠ B'.root`. So Case 2 does not support `(B'.root, PENDING)` either.

  Hence no slot-N attestation contributes to the weight of `(B'.root, PENDING)`.
- **Slot-N+1 attestations.** No slot-N+1 attestations have been cast yet — they fire at $T_{\mathrm{att}}$ of slot N+1, after the moment we are analysing. Their contribution is 0.

Therefore:

$$
\mathit{weight}((B'.\mathit{root}, \mathrm{PENDING})) \;=\; 0.4\, W. \qquad (**)
$$

**Step 3: Compare.**

From $(*)$ and $(**)$:

$$
\mathit{weight}((B.\mathit{root}, \mathrm{PENDING})) - \mathit{weight}((B'.\mathit{root}, \mathrm{PENDING})) \;\geq\; (1 - \beta)\,W - 0.4\, W \;=\; (0.6 - \beta)\,W \;>\; 0.4\, W \;>\; 0.
$$

Hence $\mathit{weight}((B.\mathit{root}, \mathrm{PENDING})) > \mathit{weight}((B'.\mathit{root}, \mathrm{PENDING}))$, and the slot-N+1 fork-choice at $\mathit{parent}(B)$'s child level selects $(B.\mathit{root}, \mathrm{PENDING})$ over $(B'.\mathit{root}, \mathrm{PENDING})$. The boost-only reorg fails.

**Step 4: Persistence past $T_{\mathrm{att}}$ of slot N+1.**

After slot-N+1 attestations are cast, the gap can only widen or stay positive under $\beta < 0.2$. In what follows, the slot-N+1 committee contributes a fresh $W_{N+1}$ worth of weight; under the per-slot stability convention from line 895, $W_{N+1} \approx W_N = W$, so we add a second $W$ on top of the slot-N contribution from $(*)$:

- Honest slot-N+1 attesters run `get_head`, observe (per Step 3) that $(B.\mathit{root}, \mathrm{PENDING})$ wins, traverse into B's branch, and attest on B's chain. Two cases: if no slot-N+1 block extends B, they attest for $B$ directly (Case 1, supporting $(B.\mathit{root}, \mathrm{PENDING})$); if an honest extending block exists at slot N+1, they attest for that block (Case 2, descendant attribution to $(B.\mathit{root}, \mathrm{PENDING})$ via the `s == PENDING` disjunct on line 13 of Algorithm 4). Either way, the contribution to $\mathit{weight}((B.\mathit{root}, \mathrm{PENDING}))$ is at least $(1 - \beta)\, W_{N+1}$, layered on top of the slot-N contribution $(1-\beta)\, W_N$ from Step 1.
- Adversarial slot-N+1 attesters can in the worst case all attest for $B'$ (Case 1 Pending), adding at most $\beta\, W_{N+1}$ to $\mathit{weight}((B'.\mathit{root}, \mathrm{PENDING}))$.

Thus, summing the slot-N committee contribution from $(*)$ with the new slot-N+1 contribution and using $W_N \approx W_{N+1} = W$, after slot N+1's $T_{\mathrm{att}}$:

$$
\mathit{weight}((B.\mathit{root}, \mathrm{PENDING})) \;\geq\; \underbrace{(1 - \beta)\, W_N}_{\text{slot-}N\text{, from }(*)} + \underbrace{(1 - \beta)\, W_{N+1}}_{\text{slot-}N{+}1\text{, this step}} \;\approx\; 2\,(1 - \beta)\, W,
$$

$$
\mathit{weight}((B'.\mathit{root}, \mathrm{PENDING})) \;\leq\; \underbrace{0.4\, W}_{\text{boost, from }(**)} + \underbrace{\beta\, W_{N+1}}_{\text{adv. slot-}N{+}1} \;\approx\; (0.4 + \beta)\, W.
$$

Under $\beta < 0.2$: $2(1-\beta)\,W > 1.6\, W > 0.6\, W \geq 0.4\,W + \beta\,W$. B remains canonical at the PENDING level. Combined with Part (a) (which establishes that the FULL/EMPTY tiebreaker for B selects FULL), $(B.\mathit{root}, \mathrm{FULL})$ is the canonical head for every honest validator running `get_head` at slot N+1. ∎

*Remark (the role of H7, precisely).* The arithmetic above relies on three quantitative inputs:
(i) *synchrony at slot N* — every honest slot-N validator sees B by $T_{\mathrm{att}}$ (delay $\Delta < T_{\mathrm{att}}$);
(ii) *no proposer equivocation visible at $T_{\mathrm{att}}$ of slot N* — there is no rival slot-N block $B''$ that could split the honest vote;
(iii) the per-committee Byzantine bound $\beta < 0.2$.

Inputs (i) and (iii) come from the network/adversary model directly. **Input (ii) is delivered by the combination of H7's "no proposer equivocation" precondition and synchrony.** Specifically: H7 requires the builder to observe no equivocation in its local view at reveal time $t_{\mathrm{rev}} \geq T_{\mathrm{att}}$; under synchrony with delay $\Delta$, anything broadcast before $t_{\mathrm{rev}} - \Delta$ is in the builder's view at $t_{\mathrm{rev}}$. Setting $\Delta < T_{\mathrm{att}}$, this forces "no equivocation broadcast before $T_{\mathrm{att}} - \Delta$", which by synchrony in turn implies "no equivocation visible to honest validators at $T_{\mathrm{att}}$". Hence (ii).

Without H7, the builder might reveal under H5's loose rule even when an equivocation $B''$ has surfaced at $\mathit{slot}(B)$; honest votes would then converge on the lex-tiebreak winner of $\{B, B''\}$, which may be $B''$ rather than $B$. In that case $\mathit{weight}((B.\mathit{root}, \mathrm{PENDING})) = 0$ from honest, and $\mathit{weight}((B'.\mathit{root}, \mathrm{PENDING})) = 0.4\, W$ from boost — so the boost-only reorg succeeds and B is orphaned. The 40% threshold inside H7 is the "reveal-time" check that makes the no-equivocation invariant testable by the builder; the actual inequality that proves no-reorg is $(*) + (**) \Rightarrow$ Step 3, with input (ii) supplied by H7.

H7's threshold of 40% is also calibrated to `PROPOSER_SCORE_BOOST = 40` for a complementary reason: if H7's "$\geq 40\%$ real attestations at reveal time" is the only thing the builder can rely on (e.g., genuine network-asynchrony at slot N degrading the synchrony input (i) to a partial-synchrony scenario where only a 40% subset of honest validators saw B in time), then in the worst case $\mathit{weight}((B.\mathit{root}, \mathrm{PENDING})) = 0.4\, W$ and $\mathit{weight}((B'.\mathit{root}, \mathrm{PENDING})) = 0.4\, W + \beta\, W = 0.6\, W > 0.4\, W$, and a reorg succeeds. This is the limit of what H7 alone (without the full synchrony input) can guarantee — the 40% threshold is exactly the boundary at which the boost+adversary cannot reorg if synchrony also delivers the missing 40% of honest weight by slot N+1. Under the document's full-synchrony assumption, that missing 40% does arrive, and $(*)$ delivers the $(1 - \beta)$ bound that closes the proof.

---

**Lemma 8** (Attestation fallback for missed slots). *Let $B$ be a block with root $r$ at slot $N$, and let $W$ be the slot-N+1 committee's effective balance. Assume: (1) `r ∈ store.payloads` (the builder revealed a valid execution payload); (2) slot $N + 1$ is missed (no block is proposed at slot $N+1$); (3) under the network synchrony assumption, every honest validator in slot-N+1 committee has `r ∈ store.payloads` in its local view by $T_{\mathrm{att}}$ of slot $N+1$, and therefore (by Lemma H2) sets `data.index = 1` when it casts its non-same-slot attestation for $B$; (4) the per-committee Byzantine bound $\beta < 1/2$ holds for slot $N+1$. Then, when `get_head` is executed at slot $N + 2$, `get_weight(store, (r, FULL)) > get_weight(store, (r, EMPTY))` (using only the contribution of slot-N+1 committee attestations).*

This lemma covers the scenario where PTC votes cannot reach the tiebreaker because the next slot was missed: PTC votes can only be included on-chain in a slot $N + 1$ block (Helper 7 asserts `data.slot + 1 = state.slot`), and they influence the tiebreaker only when `slot(B) + 1 = current_slot` (Lemma 4). When slot $N + 1$ is missed and `get_head` runs at slot $N + 2$, neither mechanism applies. Instead, the attestation weight from slot $N + 1$'s non-same-slot attesters takes over.

*Proof.* At slot $N + 2$, `current_slot = N + 2`. Block $B$ has `slot(B) = N`, so `slot(B) + 1 = N + 1 ≠ N + 2 = current_slot`. The zero-return rule (Algorithm 5, lines 3–4) does not apply, and `get_weight` computes the full attestation score for both `(r, FULL)` and `(r, EMPTY)`.

Consider the attesters from slot $N + 1$. Since no block was proposed in slot $N + 1$, these are non-same-slot attestations: their head is $B$ (from slot $N$), with `data.slot = N + 1 > slot(B) = N`. By Algorithm 4 Case 1, line 7 asserts `vote.slot ≥ block.slot` (satisfied: $N+1 \geq N$); line 8 checks `vote.slot == block.slot` (false: $N+1$ ≠ $N$), so the same-slot exclusion at line 9 does not fire. Line 10 attributes the vote: `payload_present = True` supports `(r, FULL)`; `False` supports `(r, EMPTY)`.

By assumption (3), every honest validator in slot-N+1 committee sets `payload_present = True`. Therefore the honest contribution to `get_weight(store, (r, FULL))` is at least $(1-\beta)\, W$. Adversarial validators in slot-N+1 committee can in the worst case all vote with `payload_present = False`, contributing at most $\beta\, W$ to `get_weight(store, (r, EMPTY))`. Hence:

- `get_weight(store, (r, FULL))` $\;\geq\; (1-\beta)\, W$,
- `get_weight(store, (r, EMPTY))` $\;\leq\; \beta\, W$.

Under (4), $\beta < 1/2 < 1 - \beta$, so $(1-\beta)\, W > \beta\, W$. Therefore `get_weight(store, (r, FULL)) > get_weight(store, (r, EMPTY))`, and `get_head` (Algorithm 1, lines 8–10) selects `(r, FULL)`. ∎

*Timing note for hypothesis (3).* The hypothesis "every honest validator in slot-N+1 committee has `r ∈ store.payloads` in its local view by $T_{\mathrm{att}}$ of slot N+1" follows directly from the standing synchrony assumption with $\Delta < T_{\mathrm{att}}$. Concretely: an honest builder broadcasts the payload no later than $T_{\mathrm{ptc}} = 9\,\text{s}$ of slot $N$. Under synchrony with delay $\Delta$, all honest nodes (including all slot-N+1 committee members) receive it by $T_{\mathrm{ptc}} + \Delta$ of slot $N$. Since $T_{\mathrm{att}}$ of slot $N+1$ corresponds to `slot_duration + T_att = 12 s + 3 s = 15 s` of slot $N$, the timing requirement is $T_{\mathrm{ptc}} + \Delta \leq 15\,\text{s}$, i.e., $\Delta \leq 6\,\text{s}$. The standing synchrony assumption $\Delta < T_{\mathrm{att}} = 3\,\text{s}$ comfortably satisfies this with $3\,\text{s}$ of margin, so the payload is in every honest slot-N+1 attester's view well before they vote.

*Remark.* This lemma explains why `data.index` payload signaling exists for non-same-slot attestations (Section 12.2, Attester lines 11–12): it is the fallback mechanism for when PTC votes cannot influence the fork-choice because the immediately next slot was missed. Lemma 2 established that same-slot attesters are payload-neutral; this lemma shows that non-same-slot attesters fill the gap. The two mechanisms are complementary: PTC votes decide the payload status at the tiebreaker when the next slot is on time (Lemma 7), and attestation weight decides it when the next slot is missed (this lemma).

---

#### Payment mechanism properties

The unconditional payment guarantee is the core economic property of ePBS: the proposer receives the builder's bid regardless of whether the builder reveals the execution payload. This guarantee is what makes it rational for proposers to commit to a builder's bid without seeing the payload first.

The payment mechanism has three stages. First, when the beacon block is processed, `process_execution_payload_bid` (Helper 8) records a `BuilderPendingPayment` — an IOU saying "builder owes proposer X." No money moves yet. Second, the payment is **approved** into a withdrawal queue (`builder_pending_withdrawals`). This approval happens through one of two paths:

- **Path A (builder reveals):** In the *next* block's `process_block`, `process_parent_execution_payload` (Helper 12c) calls `apply_parent_execution_payload` (Helper 12b), which calls `settle_builder_payment` (Helper 12a) to queue the payment and clear the pending entry. Payment happens in the next block, not during payload receipt.
- **Path B (builder withholds):** The pending entry remains. Same-slot attesters accumulate weight on it via `process_attestation` (Helper 15). At the epoch boundary, `process_builder_pending_payments` (Helper 14) checks whether the accumulated weight meets the 60% quorum — if so, the payment is approved.

Third, the actual balance debit (builder's `balance` decreased, proposer's `fee_recipient` credited) happens in a subsequent block's `process_withdrawals` (Helper 13), which processes the `builder_pending_withdrawals` queue. The difference between Path A and Path B is not when the money moves — it is when the payment gets approved into the queue.

The following lemmas formalize each stage.

---

**Lemma 9** (Path A: builder reveals implies payment in next block). *If an honest builder reveals a valid execution payload via the cautious strategy (Assumption H7) for a block at slot N with bid.value > 0, and a subsequent block at slot M > N declares parentStatus = FULL for block N, then `process_parent_execution_payload` (Helper 12c) settles the builder payment via `settle_builder_payment` (Helper 12a), queuing a BuilderPendingWithdrawal for the proposer and clearing the pending payment entry. Furthermore, under H7 and assuming an honest slot-N+1 proposer, M = N + 1 — the FULL declaration happens at the immediately next slot.*

This is the "happy path": the builder delivered cautiously, and the next block applies the parent's execution effects, which includes settling the payment. H7 is what makes Path A *fire* (not just *would fire if some block declared FULL*) — it ensures that the slot-N+1 honest proposer's `should_extend_payload` returns True and that the block survives any boost-only reorg attempt (Lemma 7, Part (b)).

*Proof.* By Lemma H3 (bid-reveal consistency), the honest builder's revealed execution payload has `payload.block_hash == bid.block_hash`. Therefore, `verify_execution_payload_envelope` (Helper 12) does not fail at line 16 (`assert payload.block_hash == bid.block_hash`). Assuming all other assertions pass, `on_execution_payload_envelope` (Algorithm 8) stores the payload in `store.payloads[root]`, enabling the FULL node.

When block M arrives and declares `parentStatus = FULL` for block N, `on_block` (Algorithm 7) verifies `is_payload_verified(store, N.root)` (line 5) — which holds because the payload was stored. The state transition calls `process_block` (Helper 11), whose first step is `process_parent_execution_payload` (Helper 12c). Since `bid.parent_block_hash == parent_bid.block_hash` (the block declares FULL), the function calls `apply_parent_execution_payload` (Helper 12b). Inside, `settle_builder_payment` (Helper 12a) is called with the appropriate payment index. The pending entry was created by `process_execution_payload_bid` (Helper 8, lines 17–26) during block N's processing, with `payment.withdrawal.amount = bid.value > 0`. `settle_builder_payment` queues the payment (line 4) and clears the entry (line 5), preventing double payment at the epoch boundary.

For the second claim (M = N + 1), assume the slot-N+1 proposer is honest. By H7, B has ≥ 40% real attestation weight at payload-broadcast time. By Lemma 7 Part (b), no boost-only reorg can happen at slot N+1. The honest slot-N+1 proposer therefore runs `get_head`, which selects (r, FULL) (Lemma 7 Part (a)), and proposes its block extending B-FULL. The proposer's `propose` handler (Section 12.2, lines 26–30) sets `body.parent_execution_requests = store.payloads[r].execution_requests` because `should_extend_payload(store, r)` returned True. The slot-N+1 block thus declares `parentStatus = FULL` for B, and the proof above applies with M = N + 1. ∎

---

**Lemma 10** (Path B: builder withholds with sufficient attestation implies epoch-boundary payment). *If the builder for a block at slot N with bid.value > 0 does not reveal the execution payload, and the accumulated attester weight for that slot meets the quorum threshold (payment.weight ≥ get_builder_payment_quorum_threshold(state)), then process_builder_pending_payments queues a BuilderPendingWithdrawal for the proposer at the epoch boundary.*

This is the "unhappy path": the builder did not deliver, but enough attesters supported the beacon block to trigger payment anyway.

*Proof.* Since the builder does not reveal, `verify_execution_payload_envelope` (Helper 12) is never called for this slot. `store.payloads[root]` is never populated, so no subsequent block can declare `parentStatus = FULL` for this slot (Algorithm 7, line 5 would fail). Therefore, `process_parent_execution_payload` (Helper 12c) never calls `settle_builder_payment` for this slot. The `BuilderPendingPayment` created by `process_execution_payload_bid` (Helper 8, lines 17–26) remains in `state.builder_pending_payments[SLOTS_PER_EPOCH + slot % SLOTS_PER_EPOCH]` with `payment.withdrawal.amount = bid.value > 0`.

During the epoch, `process_attestation` (Helper 15) processes same-slot attestations for this block. By lines 28–32, for each attesting validator that sets at least one new participation flag (`will_set_new_flag = True`), is a same-slot attester (`is_attestation_same_slot(state, data) = True`), and the payment amount is positive (`payment.withdrawal.amount > 0`), the validator's `effective_balance` is added to `payment.weight`.

At the epoch boundary, `process_builder_pending_payments` (Helper 14) runs. By lines 3–5, the function iterates over the previous epoch's pending payments. For our slot's entry, `payment.weight ≥ quorum` by the lemma's assumption, so line 5 executes: `state.builder_pending_withdrawals.append(payment.withdrawal)`. The payment is queued. ∎

---

**Lemma 11** (Unconditional payment — formal proof of P1). *For any block at slot N with bid.value > 0 that is attested by honest validators with total effective balance meeting the quorum threshold: the proposer's payment is queued regardless of whether the builder reveals the execution payload. **This lemma is the formal proof of P1 (Unconditional payment to the proposer) from §4 of the overview, supplying the "settlement happens" half of P1's claim. Combined with Lemma 12 (single payment) and the 2-epoch ring-buffer layout of `state.builder_pending_payments` (Helper 8 line 26 writes to the upper half; Helper 14 examines the lower half at the epoch boundary), it also supplies the "within at most two epochs" timing bound stated in P1.** Lemma 12b supplies the solvency precondition that the bid is admittable only if the balance covers it.* The hypothesis `bid.value > 0` automatically excludes self-builds, where Helper 8 line 6 forces `bid.value = 0` (so the proposer would only ever "owe itself" zero, and no IOU is recorded by Helper 8 lines 17–26 — see also Lemma 12b's non-self-build scoping). The lemma is therefore vacuous for self-builds; its content is the unconditional-payment guarantee for external-builder bids only.

*Proof.* There are two cases, based on whether `settle_builder_payment` is called (i.e., whether some subsequent block declares `parentStatus = FULL` for this slot).

**Case 1: A subsequent block declares FULL (Path A fires).** This happens when the builder reveals via the cautious strategy of Assumption H7 and an honest proposer builds on B-FULL. By Lemma 9 (second claim), under H7 and an honest slot-N+1 proposer, the FULL declaration happens at slot N + 1, Path A fires, `settle_builder_payment` queues the payment, and the pending entry is cleared. The proposer gets paid in the slot-N+1 block.

**Case 2: No subsequent block declares FULL (Path A does not fire).** This includes three sub-cases: (a) the builder withholds entirely, (b) the builder reveals an invalid execution payload (rejected by `verify_execution_payload_envelope`, so `store.payloads` is never populated), or (c) the builder reveals a valid execution payload, but no subsequent block builds on B-FULL — this last sub-case is the PTC + proposer collusion scenario analyzed in detail by Lemma 17 below. In all three sub-cases, `settle_builder_payment` is never called, so the `BuilderPendingPayment` persists. Same-slot attesters accumulate `payment.weight` via `process_attestation` (Helper 15). By Lemma 20 (quorum reachability), under the lemma's hypothesis — total honest effective balance attesting for B reaches the quorum threshold — and assuming attestations are included on-chain by the next-epoch boundary, the accumulated `payment.weight` meets the quorum. By Lemma 10, `process_builder_pending_payments` queues the payment at the epoch boundary. The proposer gets paid.

In both cases, the proposer receives a `BuilderPendingWithdrawal` with amount `bid.value`. The actual balance debit from the builder occurs when `apply_withdrawals` (Helper 21) processes the `builder_pending_withdrawals` queue during a subsequent block's `process_withdrawals` (Helper 13). ∎

*Remark.* The quorum threshold (60% of per-slot active balance, Helper 18) exists to protect the builder: if the beacon block was not widely attested (e.g., the proposer equivocated, the block arrived late, or it was on a minority fork), the builder should not be forced to pay for a block the network did not accept. Without the quorum, a malicious proposer could broadcast a block that is consensus-valid but unsupported by the network, have it attested by a small minority, and still extract payment from the builder.

---

**Lemma 12** (Path A clears Path B). *If `settle_builder_payment` (called by `apply_parent_execution_payload` in the next block's `process_parent_execution_payload`) completes for slot N, then process_builder_pending_payments at the epoch boundary does not produce a second payment for slot N.*

This ensures that the two paths are mutually exclusive: the proposer gets paid once, not twice.

*Proof.* By Helper 12a (`settle_builder_payment`), counting from `def settle_builder_payment(...)` as line 1: line 5 queues the payment to `state.builder_pending_withdrawals` (when `withdrawal.amount > 0`), and line 6 replaces the pending payment entry with a fresh `BuilderPendingPayment()` carrying `weight = 0` and `withdrawal.amount = 0`. The epoch-boundary check then fails: when `process_builder_pending_payments` (Helper 14) runs, the entry's `weight` is 0, the quorum check `payment.weight >= quorum` fails, and the entry is discarded during the epoch shift without producing a second payment. (An ancillary effect of the clear is that any further `process_attestation` calls referencing this slot are no-ops at the weight-accumulation step: the guard `payment.withdrawal.amount > 0` in Helper 15 fails. Either fact alone suffices.) ∎

*Remark on Helper 12b Case 3.* There is a third path inside `apply_parent_execution_payload` (Helper 12b, lines 19–26) that appends a `BuilderPendingWithdrawal` directly when `parent_epoch < get_previous_epoch(state)`, bypassing `settle_builder_payment`. One might worry that this third path could double-fire with Path B, queuing a second withdrawal. It cannot, due to a mutual exclusion: Case 3 requires `state.latest_execution_payload_bid` to still point to a bid more than one epoch old, which by Helper 8 line 27 (the unconditional cache update on every `process_block`) means no intervening block was processed. But Path B's quorum-met condition (`payment.weight ≥ quorum`) requires same-slot attestations to slot N's block to be **included on-chain** via subsequent blocks' `process_attestation`. With no intervening blocks, no attestation inclusions occur, `payment.weight` stays at 0, and Path B does not fire. Conversely, if any intervening block was processed, `state.latest_execution_payload_bid` is no longer slot N's bid, so Case 3 cannot fire for slot N's entry. The two paths are reachable from disjoint chain histories.

---

**Lemma 12b** (Builder solvency at bid time). *If `process_execution_payload_bid` (Helper 8) accepts a non-self-build bid with `bid.value > 0` for builder index $i$, then immediately after the call:*

$$
\texttt{state.builders}[i].\texttt{balance} \;\geq\; \texttt{MIN\_DEPOSIT\_AMOUNT} \;+\; \sum_{w \,\in\, \texttt{state.builder\_pending\_withdrawals},\; w.\texttt{builder\_index} \,=\, i} w.\texttt{amount} \;+\; \sum_{p \,\in\, \texttt{state.builder\_pending\_payments},\; p.\texttt{withdrawal.builder\_index} \,=\, i} p.\texttt{withdrawal.amount}
$$

*That is, the builder's balance covers `MIN_DEPOSIT_AMOUNT` plus every outstanding obligation — both already-approved withdrawals (Path A or Path B output) and pending IOUs from concurrent slots — including the new bid just accepted.*

This is the solvency precondition for P1 (Unconditional payment to the proposer) from §4 of the overview: a builder cannot commit to a bid it cannot pay, even across concurrent slots. The check is what makes Lemma 11 (unconditional payment) economically sound: when the proposer is paid, the builder's balance is guaranteed to cover the payment. The overview treats this as a remark in §5 Phase 1b rather than as a standalone property.

*Proof.* By Helper 8 (line 10), the function asserts `can_builder_cover_bid(state, builder_index, amount)`. By Helper 6, this assertion is equivalent to `balance - MIN_DEPOSIT_AMOUNT - pending >= bid_amount`, where `pending = get_pending_balance_to_withdraw_for_builder(state, builder_index)`. By Helper 20, `pending` is exactly the sum on the right-hand side **minus** the new bid (since at the time of the check, the new IOU has not yet been written). Helper 8 then writes the new `BuilderPendingPayment` at lines 17–26. After the write, the right-hand side sum increases by `bid.value = amount`, and the inequality `balance ≥ MIN_DEPOSIT_AMOUNT + pending + amount` holds with equality in the worst case. ∎

*Remark.* The check is conservative: the builder must reserve `MIN_DEPOSIT_AMOUNT` against every concurrent obligation, so a builder with balance $B$ can have at most $B - \texttt{MIN\_DEPOSIT\_AMOUNT}$ in total outstanding bids. Helper 21 (`apply_withdrawals`) further protects against edge cases via `min(amount, balance)` — but as Helper 6's commentary notes, that is a defensive safety net, not the primary protection. The primary protection is this invariant.

---

**Lemma 12c** (Builder exit does not interfere with pending obligations). *Let builder index $i$ have its exit initiated at epoch $e$ via `initiate_builder_exit` (Helper 28). Let $\mathcal{P}_i$ denote the set of `BuilderPendingPayment` entries with `withdrawal.builder_index = i` outstanding in `state.builder_pending_payments` at epoch $e$, and let $\mathcal{W}_i$ denote the set of `BuilderPendingWithdrawal` entries with `builder_index = i` outstanding in `state.builder_pending_withdrawals` at epoch $e$. Then:*

1. *(No new bids accepted.) For every slot $s$ with $\mathit{epoch}(s) \geq e$, `process_execution_payload_bid` (Helper 8, line 9) rejects any non-self-build bid for builder $i$, because `is_active_builder` (Helper 1) returns False once `withdrawable_epoch \neq \texttt{FAR\_FUTURE\_EPOCH}`.*
2. *(Pending obligations settle on the normal schedule.) Every entry in $\mathcal{P}_i$ either (a) settles via Path A through `settle_builder_payment` (Helper 12a) — appending to `state.builder_pending_withdrawals` — within at most one slot of the exit, or (b) settles via Path B through `process_builder_pending_payments` (Helper 14) by the epoch boundary $e+1$ if `payment.weight ≥ quorum`, or (c) is silently discarded at the epoch boundary if the quorum is not met. In all three sub-cases, the entry is removed from `state.builder_pending_payments` by epoch $e+2$ (the 2-epoch sliding-window bound of Lemma 21).*
3. *(IOUs are paid before the exit sweep.) The 2-epoch payment window is strictly shorter than $\texttt{MIN\_BUILDER\_WITHDRAWABILITY\_DELAY} = 64$ epochs. Therefore, by epoch $e + 2$, every entry of $\mathcal{P}_i$ that resolves into an IOU has already been written to `state.builder_pending_withdrawals`. From epoch $e+2$ through epoch $e+63$, `process_withdrawals` (Helper 13) drains those IOUs via `get_builder_withdrawals`, which runs strictly before `get_builders_sweep_withdrawals` in `get_expected_withdrawals` (`beacon-chain.md`). Each IOU debits builder $i$'s balance via `apply_withdrawals` (Helper 21) using `min(withdrawal.amount, builder_balance)`. From epoch $e + \texttt{MIN\_BUILDER\_WITHDRAWABILITY\_DELAY} = e + 64$ onward, `get_builders_sweep_withdrawals` becomes eligible to fire (the guard `builder.withdrawable_epoch \leq epoch` succeeds) and the residual balance is withdrawn to `builder.execution_address`.*
4. *(Solvency at exit.) By Lemma 12b, at the time of every accepted bid prior to epoch $e$, builder $i$'s balance covered every outstanding obligation plus `MIN_DEPOSIT_AMOUNT`. Since no new bids are accepted from epoch $e$ onward (item 1), the obligation set $\mathcal{P}_i \cup \mathcal{W}_i$ does not grow. Therefore the balance available to honor pending IOUs at the time of `apply_withdrawals` is at least the original obligation total, and `min(withdrawal.amount, builder_balance)` reduces to `withdrawal.amount` in every case where Lemma 12b's invariant has been preserved (i.e., in the absence of out-of-band balance reductions such as a slashing event between bid acceptance and IOU processing).*

This formalises the interaction between the builder lifecycle (Helpers 26–28) and the unconditional payment mechanism (Lemmas 9–11). The spec relies on a *temporal separation* — the 2-epoch payment window vs. the 64-epoch withdrawability delay — rather than an explicit inhibition of `initiate_builder_exit` while obligations are pending. The temporal separation is sufficient because the payment window is 32× shorter than the withdrawability delay.

*Proof.*

*Item 1.* Helper 28 sets `state.builders[i].withdrawable_epoch = get_current_epoch(state) + MIN_BUILDER_WITHDRAWABILITY_DELAY`. After this assignment, `withdrawable_epoch ≠ FAR_FUTURE_EPOCH`, so by Helper 1, `is_active_builder(state, i)` returns False. Helper 8 line 9 asserts `is_active_builder(state, builder_index)` for non-self-build bids; the assertion fails for builder $i$ from epoch $e$ onward, so no `BuilderPendingPayment` for builder $i$ is created post-exit.

*Item 2.* The case split (a)/(b)/(c) is exactly the case split of Lemma 11 (unconditional payment) and its proof in Cases 1, 2: Path A fires iff a subsequent block declares `parentStatus = FULL` for the slot of the pending payment; Path B fires iff `payment.weight ≥ quorum` at the epoch boundary; otherwise the entry is discarded. None of these mechanisms depend on `is_active_builder` — `process_attestation` (Helper 15, lines 28–32) accumulates weight on `payment.withdrawal.amount > 0` regardless of builder activity status, and `process_builder_pending_payments` (Helper 14) and `settle_builder_payment` (Helper 12a) operate on `state.builder_pending_payments[*]` directly. The 2-epoch sliding-window bound is established by Lemma 21 (the index written by Helper 8 is read by Helper 12b within the same or next epoch; older entries are evicted by Helper 14 lines 6–8).

*Item 3.* `MIN_BUILDER_WITHDRAWABILITY_DELAY = 2^6 = 64` (Section 8 / `beacon-chain.md`), which is strictly greater than 2.[^lem12c-constants] The `get_builders_sweep_withdrawals` guard at `beacon-chain.md` line 1239 checks `builder.withdrawable_epoch <= epoch and builder.balance > 0`; since `withdrawable_epoch = e + 64`, the guard remains False through epoch $e + 63$ and only succeeds from epoch $e + 64$. The ordering of `get_builder_withdrawals` (which drains the IOU queue) before `get_builders_sweep_withdrawals` (which sweeps the residual) inside `get_expected_withdrawals` is a fixed property of the spec function (`beacon-chain.md`).

[^lem12c-constants]: The Item 3 argument relies on the temporal separation between **two distinct constants**: the 2-epoch payment-settlement window (Lemma 21, fixed by the size of `state.builder_pending_payments`) and the 64-epoch withdrawability delay (`MIN_BUILDER_WITHDRAWABILITY_DELAY`). If either constant is changed in a future spec revision, Item 3 of this proof must be revisited: the lemma requires the payment-settlement window to be **strictly shorter** than the withdrawability delay so that every IOU resolves before the exit sweep activates. The current 32× margin is well-padded, but the proof is not a function of the gap — it is a function of the ordering.
    
*Item 4.* Lemma 12b establishes the invariant `balance ≥ MIN_DEPOSIT_AMOUNT + pending` at every bid acceptance for builder $i$. Item 1 implies no new bids increase `pending` after epoch $e$. The set of obligations therefore monotonically shrinks (as IOUs are paid and discharged) over $[e, e+64]$. At the moment each IOU is presented to `apply_withdrawals`, the inequality `builder_balance ≥ withdrawal.amount` holds — provided no slashing, `seq_payment_underflow`, or out-of-band balance change has occurred (the standard caveat noted in Lemma 15's proof and Helper 6's commentary). The `min(amount, balance)` clause in `apply_withdrawals` line 1309 is the spec's defensive fallback for those edge cases. ∎

*Remark.* The lemma answers an audit-level question: *can a builder exit while it has outstanding obligations, and if so, are those obligations honored?* The answer is yes, and the spec's design relies on three interlocking mechanisms:

1. **Activity-based bid rejection** (Helper 1 → Helper 8 line 9) prevents new obligations after exit.
2. **Temporal separation** (`MIN_BUILDER_WITHDRAWABILITY_DELAY` ≫ 2-epoch payment window) ensures all pending obligations resolve into IOUs before the exit sweep activates.
3. **Withdrawal ordering** (`get_builder_withdrawals` runs before `get_builders_sweep_withdrawals` in `get_expected_withdrawals`) ensures that within any single block, IOUs are paid before the residual is swept.

The spec does not block `initiate_builder_exit` while $\mathcal{P}_i$ is non-empty; it does not need to, because (1)+(2)+(3) together guarantee that the obligations settle before the builder's balance is fully withdrawn.

---

#### Adversarial properties

The properties proved so far assume that the relevant actors follow the protocol. In practice, any subset of validators, PTC members, builders, or proposers may be Byzantine — deviating arbitrarily from the honest behavior specified in Section 12.2. This section establishes what the protocol guarantees *despite* such deviations. The key insight is that different actors have different attack surfaces, and the protocol's layered design limits the damage each can cause.

The interaction between the PTC and the next-slot proposer is central to understanding the adversarial landscape. The FULL/EMPTY tiebreaker for the immediately previous slot (Definition 13) is resolved by `should_extend_payload` (Algorithm 6), which first checks the hard guard (`is_payload_verified`, line 2), then the PTC primary path (line 6: `is_payload_timely and is_payload_data_available`), and then falls through to proposer-based fallbacks (lines 7–9). The following table provides a comprehensive analysis of all adversarial combinations. The first four rows cover the case where the builder **delivered** a valid execution payload (`r ∈ store.payloads`); the last four cover the case where the builder **withheld** (`r ∉ store.payloads`).

**Payload delivered** (`r ∈ store.payloads`, FULL node exists):

| PTC                     | Next-slot proposer         | Outcome         | Mechanism                                                                                                                                                                              |
| ----------------------- | -------------------------- | --------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Honest (votes True)     | Honest (declares FULL)     | FULL            | PTC primary path confirms; fallbacks also agree                                                                                                                                        |
| Honest (votes True)     | Malicious (declares EMPTY) | FULL            | **PTC primary path overrides** the proposer's false EMPTY declaration                                                                                                            |
| Malicious (votes False) | Honest (declares FULL)     | FULL            | Proposer sees `proposer_boost_root` is zero → fallback (a); attesters see proposer declares FULL → fallback (c) (Lemma 14)                                                         |
| Malicious (votes False) | Malicious (declares EMPTY) | **EMPTY** | **Only successful attack.** PTC primary path fails (PTC lied) and all fallbacks fail (proposer declared EMPTY). Recovery requires honest proposer in subsequent slot. (Lemma 17) |

**Payload NOT delivered** (`r ∉ store.payloads`, FULL node does not exist):

| PTC                    | Next-slot proposer        | Outcome | Mechanism                                                                                                                                                    |
| ---------------------- | ------------------------- | ------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| Honest (votes False)   | Honest (declares EMPTY)   | EMPTY   | Correct outcome — execution payload wasn't delivered, everyone agrees                                                                                       |
| Honest (votes False)   | Malicious (declares FULL) | EMPTY   | `on_block` (Algorithm 7, line 5) asserts `is_payload_verified(store, parent_root)` — assertion fails, block rejected by every honest node (Lemma 13)    |
| Malicious (votes True) | Honest (declares EMPTY)   | EMPTY   | FULL node doesn't exist (`store.payloads` not populated) — PTC votes never consulted. Tiebreaker not needed since only EMPTY child node exists (Lemma 13) |
| Malicious (votes True) | Malicious (declares FULL) | EMPTY   | Same as above —`on_block` rejects the block. `is_payload_verified` is the gate, PTC cannot create branches (Lemma 13)                                   |

**Key takeaways:**

1. **The PTC exists to defend against a single malicious proposer.** Without PTC, a malicious proposer could force EMPTY by declaring it (row 4 of the first table would also apply to row 2). With PTC, the honest PTC majority overrides the malicious proposer via the primary path of `should_extend_payload`.
2. **The only successful attack requires two colluding parties**: a PTC majority voting False AND a malicious proposer declaring EMPTY. Even then, the damage is bounded (Lemma 17): `store.payloads` persists, the builder still gets paid (Path A), and honest proposers in subsequent slots can recover by building on B-FULL.
3. **PTC cannot create FULL nodes.** Regardless of PTC votes, the FULL node exists only if `store.payloads` is populated, which requires the execution engine to validate the execution payload (Lemma 1). A malicious PTC voting True for a withheld or invalid execution payload has zero effect — the bottom four rows all resolve to EMPTY.
4. **`store.payloads` is the hard gate; PTC is the soft signal.** Branch existence is controlled by execution validity (`store.payloads`). The PTC only influences which existing branch the tiebreaker favors. This layered design means the PTC can be Byzantine without compromising the integrity of the fork-choice tree structure — only the tiebreaker resolution is affected.

---

**Lemma 13** (Malicious PTC cannot force FULL for an invalid execution payload). *Let B be a block with root r. If the execution payload for B is invalid (rejected by the execution engine) or never revealed, then get_head cannot return (r, FULL), regardless of PTC votes.*

Even if all 512 PTC members are Byzantine and unanimously vote `payload_present = True`, the FULL node does not exist in the fork-choice tree for an invalid or missing payload. PTC votes influence the tiebreaker, but the tiebreaker only runs between child nodes that `get_node_children` actually returns — and the FULL child node is gated by `store.payloads`, not by PTC votes.

*Proof.* By Lemma 1, (r, FULL) is a child of (r, PENDING) if and only if `r ∈ store.payloads`. By Algorithm 8 (on_execution_payload_envelope), `store.payloads[r]` is written only at line 7, which is reached only after `verify_execution_payload_envelope` completes successfully at line 6. If the execution payload is invalid, `verify_execution_payload_envelope` fails at one of its assertions (beacon block consistency, bid consistency, execution engine validation) and line 7 of Algorithm 8 is never reached. If the execution payload is never revealed, Algorithm 8 is never called at all. In either case, `r ∉ store.payloads`, and by Lemma 1 the FULL child does not exist.

PTC votes are stored in `store.payload_timeliness_vote[r]` (Algorithm 9, line 13), which is read by `is_payload_timely` (Helper 4) and `should_extend_payload` (Algorithm 6). However, `should_extend_payload` is called by the tiebreaker (Definition 13) only to decide between FULL and EMPTY children that already exist. Since (r, FULL) does not exist, `get_node_children` (Algorithm 2, lines 3–7) returns only {(r, EMPTY)}, the tiebreaker is never consulted for this block, and `get_head` (Algorithm 1) proceeds down the EMPTY path. ∎

*Remark.* This is a stronger result than Lemma 6 (which assumed the builder withholds). Lemma 13 covers the case where the builder actively reveals an *invalid* execution payload. The protection is the same: `store.payloads` is the sole gatekeeper, and it requires execution engine approval. This separation between the PTC (observational, gossip-level) and `store.payloads` (validity-enforcing, execution-level) is a fundamental design property of ePBS.

---

**Lemma 14** (Malicious PTC cannot force EMPTY for a valid execution payload — bounded by fallbacks). *Let B be a block with root r at slot N. Assume: (1) the builder reveals a valid execution payload, so that r ∈ store.payloads; (2) at least PTC_SIZE / 2 = 256 out of 512 PTC members vote payload_present = False, so that sum(True votes) ≤ 256 and `is_payload_timely` fails (Helper 4 returns True only if sum > PAYLOAD_TIMELY_THRESHOLD = 256). Then, when get_head is executed at slot N + 1, should_extend_payload(store, r) still returns True if any of the following holds:*

- *(a) No block has been proposed in slot N + 1 (proposer_boost_root is the zero root).*
- *(b) A block has been proposed in slot N + 1 but it is not a child of B (boost_block.parent_root ≠ r).*
- *(c) A block has been proposed in slot N + 1 that is a child of B and declares parentStatus = FULL.*

The PTC timeliness check (Helper 4) requires a strict majority: more than `PAYLOAD_TIMELY_THRESHOLD` votes for `payload_present = True`. If this majority is not reached — whether because Byzantine PTC members lied, or because some honest members genuinely did not observe the payload due to network delays — the primary path of `should_extend_payload` fails. However, the fallback conditions in Algorithm 6 (lines 7–9) provide three independent paths to FULL that bypass PTC votes entirely. A PTC failure can only force EMPTY when *none* of these fallbacks apply — which requires a specific combination: a block was proposed in slot N+1, it is a child of B, and it declares EMPTY.

*Proof.* The hard guard at Algorithm 6 line 2 passes because `is_payload_verified(store, r)` is True by assumption (1). By assumption (2), `sum(store.payload_timeliness_vote[r]) ≤ PTC_SIZE / 2`. By Helper 4 (`is_payload_timely`), the timeliness check fails. The PTC primary path at line 6 evaluates to False, and the function does not return True via it.

The function then evaluates the remaining fallback conditions in the disjunction:

- **Line 7:** If `proposer_boost_root` is the zero root (no block proposed in slot N+1), return True. This is condition (a).
- **Line 8:** If `store.blocks[proposer_boost_root].parent_root != root` (the proposed block is not a child of B), return True. This is condition (b).
- **Line 9:** If `is_parent_node_full(store, store.blocks[proposer_boost_root])` (the proposed block declares its parent was FULL), return True. This is condition (c).

In each of these cases, `should_extend_payload` returns True despite the PTC voting False. By Definition 13, the tiebreaker assigns FULL priority 2 and EMPTY priority 1, so `get_head` selects (r, FULL).

The only scenario where `should_extend_payload` returns False is when either the hard guard fails (`is_payload_verified` returns False — the payload was never verified locally) or all of the following hold simultaneously: the payload is verified but a block was proposed in slot N+1 (`proposer_boost_root ≠ Root()`), it is a child of B (`boost_block.parent_root == r`), and it declares `parentStatus = EMPTY`. The latter case requires the slot N+1 proposer to actively collude with the PTC by choosing the EMPTY path despite having access to the valid execution payload. ∎

---

**Lemma 15** (Malicious builder cannot avoid payment). *If a builder submits a bid with bid.value > 0 that is included in a beacon block, and that beacon block is attested by honest validators with total effective balance meeting the quorum threshold, then the builder's balance is debited by bid.value regardless of the builder's subsequent behavior.*

A builder cannot commit to a bid and then avoid paying. The unconditional payment mechanism ensures that the builder pays whether it reveals (Path A — settled by `settle_builder_payment` in the next block), withholds (Path B — settled at epoch boundary), or reveals an invalid execution payload (also Path B, since the invalid execution payload fails `verify_execution_payload_envelope`, the payload is never stored, and `settle_builder_payment` is never called).

*Proof.* There are three cases for the builder's behavior after the bid is included:

**Case 1: Builder reveals a valid execution payload.** By Lemma 9 (Path A), the next block's `process_parent_execution_payload` calls `settle_builder_payment`, which queues the payment and clears the pending entry. The builder pays.

**Case 2: Builder withholds the execution payload entirely.** By Lemma 10 (Path B), the pending payment remains, attester weight accumulates, and `process_builder_pending_payments` queues the payment at the epoch boundary if the quorum is met. By the lemma's assumption, the quorum is met. The builder pays.

**Case 3: Builder reveals an invalid execution payload.** `verify_execution_payload_envelope` (Helper 12) fails at one of its assertions. The payload is never stored in `store.payloads`, so no subsequent block can declare `parentStatus = FULL` for this slot (`on_block` Algorithm 7, line 5 would fail). Therefore, `settle_builder_payment` is never called for this slot. The pending payment entry created by `process_execution_payload_bid` (Helper 8, lines 17–26) is never cleared. This is identical to Case 2 from the payment mechanism's perspective: the entry remains, weight accumulates, and Path B queues the payment at the epoch boundary. The builder pays.

In all three cases, the payment is queued in `builder_pending_withdrawals` and eventually debited from `state.builders[builder_index].balance` during `apply_withdrawals` (Helper 21) in a subsequent block's `process_withdrawals` (Helper 13). At bid time, `can_builder_cover_bid` (Helper 6) verified that `balance - MIN_DEPOSIT_AMOUNT - pending_obligations >= bid.value`, where `pending_obligations` (Helper 20) includes all outstanding payments and bids. This ensures sufficient balance for the bid provided no out-of-band balance reduction (such as slashing) occurs between bid verification and withdrawal processing. The spec's `apply_withdrawals` includes a `min(amount, balance)` safeguard that prevents underflow in edge cases. ∎

---

**Lemma 16** (Malicious proposer cannot extract payment without network support — formal proof of P3). *If a proposer includes a builder's bid in a beacon block, but the beacon block does not accumulate enough same-slot attestation weight to meet the quorum threshold, and the builder does not reveal the execution payload, then the builder is not charged. **This lemma is the formal proof of P3 (Builder withholding protection) from §4 of the overview.** Combined with Lemma 20 case (c), the 60% = 40% (real-reveal threshold) + 20% (adversary budget) calibration ensures that Byzantine validators alone cannot push the quorum over 60% — honest participation is required, and a builder whose block lacks honest support is therefore protected when withholding.*

The 60% quorum threshold (Helper 18) protects builders from malicious proposers. A proposer who broadcasts a late, equivocating, or otherwise poorly-supported beacon block cannot force the builder to pay for a block that the network did not accept.

*Proof.* Since the builder does not reveal, `store.payloads` is never populated, no subsequent block can declare FULL, and `settle_builder_payment` (Path A — Lemma 9) is never called. The pending payment entry remains in `state.builder_pending_payments` with whatever weight was accumulated by `process_attestation` (Helper 15, lines 28–32).

At the epoch boundary, `process_builder_pending_payments` (Helper 14) checks `payment.weight >= quorum` (line 4). By the lemma's assumption, `payment.weight < quorum`. The condition fails, line 5 does not execute, and the payment is not queued. The entry is discarded during the epoch shift (Helper 14, lines 6–8). The builder's balance is never debited. ∎

*Remark on coverage of P3.* Lemma 16 covers Case 1 of P3 — the *no-weight withhold* path, where the honest builder withholds because Assumption H7 precondition (ii) fails (< 40% real attestation accumulated). The complementary Case 2 of P3 — *equivocation-driven withhold*, where H7 precondition (iii) fails because the builder observed a `ProposerSlashing` — is handled by Lemma 22 part (iv): when the slashing evidence is included on-chain within the 2-epoch `builder_pending_payments` window, `process_proposer_slashing` (Helper 22, lines 11–17) clears the `BuilderPendingPayment` entry entirely, overriding whatever weight had accumulated. Under β < 20%, honest proposers control > 80% of slots, so the slashing evidence is reliably included within a few slots. Together, Lemma 16 (Case 1) and Lemma 22 part (iv) (Case 2) prove P3 across all H7-driven withhold causes.

*Remark 1.* This protection has a subtle interaction with proposer equivocation. If the proposer broadcasts two conflicting beacon blocks, honest attesters will set `equivocating` flags and their attestations may not count toward the payment weight (since equivocating attesters are excluded from weight computation in Algorithm 5, and the attester may be confused about which block to attest to). This makes it harder for an equivocating proposer to reach the quorum, providing additional protection for the builder.

*Remark 2 (Attestation weight serves two distinct roles).* Same-slot attestations play different roles in the fork-choice and the payment mechanism, and it is important not to confuse them:

- **Fork-choice weight** (Algorithm 5, `get_weight`): same-slot attestations count for the PENDING node of their block (helping it win against competing blocks at the same slot), but they are excluded from FULL and EMPTY weight (Lemma 2). The FULL/EMPTY decision for the immediately previous slot is delegated to the PTC tiebreaker (Lemma 4).
- **Payment quorum weight** (Helper 15, `process_attestation`, lines 28–32): same-slot attestations accumulate `payment.weight` for the builder pending payment. This is the mechanism that powers Path B of unconditional payment. The payment quorum measures "did enough attesters support this beacon block?", which is a separate question from "was the execution payload FULL or EMPTY?".

PTC votes are irrelevant to the payment mechanism. They go into `store.payload_timeliness_vote` (read by the fork-choice tiebreaker via Helper 4 → Algorithm 6 → Definition 13) and never into `payment.weight` (which is accumulated by `process_attestation` from same-slot attesters only). The two data paths are completely independent.

---

**Lemma 17** (PTC + proposer collusion — bounded damage; attacker requirements under H7). *Let B be a block with root r at slot N. Assume: (1) the builder reveals a valid execution payload for B (under H5; H7 not required); (2) at least PTC_SIZE / 2 = 256 out of 512 PTC members vote payload_present = False, so that sum(True votes) ≤ 256 and the timeliness check fails; (3) the slot N+1 proposer is Byzantine and builds on B-EMPTY (declares parentStatus = EMPTY despite having access to the valid execution payload). Then:*

- *(i) At slot N+1, get_head returns (r, EMPTY) — the colluders succeed in suppressing FULL for one slot.*
- *(ii) The FULL branch remains structurally available — `store.payloads[r]` is never removed (Lemma 23) — so recovery is possible whenever an honest proposer at some later slot M > N+1 chooses to build on B-FULL. Recovery is not guaranteed in any fixed number of slots and depends on proposer-side choice rather than on the fork-choice rule alone.*

*Note on hypothesis realisability.* Hypothesis (2) requires at least 256 PTC members voting False. Under the document's standing β < 0.2 per-committee bound, the adversary controls at most $\lfloor 0.2 \cdot 512 \rfloor = 102$ PTC members — far below 256 — so hypothesis (2) **cannot be met by Byzantine PTC voting alone within the standard adversarial model**. The lemma therefore applies in an *extended* threat model where one of the following additionally holds: (a) per-PTC β is locally above $\sim 0.5$ in this slot (a deviation that is overwhelmingly unlikely under Lemma 26 unless global β' is itself near 0.5), or (b) an asynchronous adversary prevents enough honest PTC members from observing the payload by $T_{\mathrm{ptc}}$ (so they vote False for lack of observation, per Lemma H4, even though they are honest). The lemma's value under standing assumptions is as a *worst-case bound* on damage *if* such an extended threat materialises, not as a description of an attack feasible under β < 0.2 alone.

*Why hypothesis (3) is formulated as "declares EMPTY" rather than "reorgs B."* Under H7 (cautious-reveal builder) and the network-model synchrony assumption, a slot-$N+1$ proposer cannot substitute a boost-only reorg of $B$ for the EMPTY declaration. By Lemma 7 Part (b), every honest slot-$N$ validator has voted for $B$ by $T_{\mathrm{att}}$, and the resulting bound $\mathit{weight}((B.\mathit{root}, \mathrm{PENDING})) \geq (1 - \beta)\, W > 0.8\, W$ at slot $N+1$ strictly exceeds the boost-induced upper bound $\mathit{weight}((B'.\mathit{root}, \mathrm{PENDING})) \leq 0.4\, W + \beta\, W \leq 0.6\, W$. Any sibling-reorg block $B'$ at slot $N+1$ therefore cannot outweigh $B$ at the PENDING level. The colluder is forced onto the alternative path of declaring $\mathit{parentStatus} = \mathrm{EMPTY}$ on $B$, which in turn requires the PTC primary path of `should_extend_payload` to fail — exactly hypothesis (2). The combined resource requirement (a bribed PTC majority *plus* a colluding next-slot proposer) is what makes the attack expensive to mount under balance-weighted PTC sampling (Lemma 26).*

This is the worst-case scenario for PTC failure combined with proposer collusion: the PTC timeliness check fails (whether due to Byzantine PTC members or network issues) AND the next proposer colludes by building on the wrong branch. The colluders can suppress FULL for exactly one slot, but they cannot permanently alter the chain's view of B's execution payload status. The damage is bounded because the two-chain structure records B's execution payload delivery in `store.payloads` (which cannot be undone), and honest attesters in subsequent slots provide the corrective weight.

*Proof.*

*Part (i):* The hard guard at Algorithm 6 line 2 passes because the builder revealed a valid execution payload (`is_payload_verified(store, r)` is True). By assumption (2), more than half of PTC members voted False, so `sum(store.payload_timeliness_vote[r]) ≤ PTC_SIZE / 2` and `is_payload_timely` returns False (Helper 4). The PTC primary path at line 6 fails. By assumption (3), a block exists in slot N+1 (`proposer_boost_root ≠ Root()`, so line 7 fails), it is a child of B (`boost_block.parent_root == r`, so line 8 fails), and it declares `parentStatus = EMPTY` (`is_parent_node_full` returns False, so line 9 fails). All conditions in the disjunction are False, and `should_extend_payload` returns False. By Definition 13, tiebreaker for FULL = 0, tiebreaker for EMPTY = 1. `get_head` selects (r, EMPTY).

*Part (ii):* We characterise what the protocol guarantees once the colluders have succeeded at slot N+1. The honest framing is **damage bounded, not damage recovered**: the protocol does not enforce that the FULL branch will be re-selected by honest proposers, but it does guarantee that the FULL branch remains structurally available, that no honest validator is slashed, and that the slot-N proposer is still paid.

**Slot N+1 state.** `get_head` returned (r, EMPTY) → (C, PENDING), where C is the colluding proposer's block built on B-EMPTY. Honest attesters at slot N+1 run `get_head` at $T_{\text{att}}$, see C as the head, and cast a same-slot attestation for C with `data.index = 0` (Lemma H1: same-slot voters carry no payload opinion). The vote's direct contribution to fork-choice weight is via Case 1 to $(C.\mathit{root}, \mathrm{PENDING})$; via Case 2 of `is_supporting_vote`, `get_ancestor(C.root, slot(B))` reads `parentStatus(C) = EMPTY` (from C's bid commitment, by Definition 4) and attributes the vote to $(r, \mathrm{EMPTY})$ at the ancestor level — a *structural* attribution from the chain's parentStatus declarations, not from the voter's payload opinion.

**Slot N+2 fork-choice.** The zero-return rule does not apply to B (`slot(B) + 1 = N+1 ≠ N+2`), so `get_weight` computes full attestation scores. At this point `get_weight((r, EMPTY))` includes slot N+1's honest attestation weight (attributed via C's EMPTY declaration); `get_weight((r, FULL))` has no attestation weight (no block has been built on B-FULL). An honest proposer at slot N+2 running `get_head` sees B-EMPTY → C as the winning chain and extends C.

**What the protocol guarantees from slot N+2 onward.** The honest reading is the following:

- **The FULL branch persists structurally.** `store.payloads[r]` was populated when B's execution payload was verified and is never removed (Lemma 23). $(r, \mathrm{FULL})$ remains in the fork-choice tree as a valid child of $(r, \mathrm{PENDING})$ (Lemma 1).
- **Proposer payment is unaffected.** The slot-N proposer is paid by one of two mutually exclusive paths. If a later block (at any slot $\geq N + 2$) eventually declares FULL for B, Path A fires (Lemma 9). Otherwise — including the case where the EMPTY chain remains canonical indefinitely — Path B fires at the next epoch boundary, provided slot N's beacon block reached the 60% quorum (Lemma 20 case (a)). The colluders cannot prevent payment via either path.
- **No honest validator is slashed.** Honest slot-N+1 attesters voted for C following `get_head` honestly. The protocol does not slash for following the fork-choice rule. No equivocation occurred (each honest validator cast one attestation).

**What the protocol does NOT guarantee.** The protocol does **not** force honest proposers to re-build on $(r, \mathrm{FULL})$ once slot N+1's EMPTY branch has accumulated weight. `get_head` follows the heaviest chain; an honest proposer following it will extend C, not B-FULL. Recovery to FULL therefore depends on an out-of-protocol decision (e.g., an off-chain coordination heuristic, or an implementation that breaks ties differently). The lemma's "bounded damage" framing should be read as: the damage is bounded to *the FULL/EMPTY decision at slot N+1 and downstream blocks following the EMPTY chain*, not as a guarantee that the FULL branch will eventually be re-selected by honest proposers. ∎

*Remark on the damage profile.* The damage from PTC + proposer collusion at slot N is:

- **The FULL/EMPTY decision for slot N is decided EMPTY by the canonical chain**, even though B's execution payload was validly revealed and is locally available to every honest node. Subsequent honest proposers extend the EMPTY chain.
- **No permanent chain damage.** `store.payloads[r]` exists in every honest node's store and cannot be removed. The $(r, \mathrm{FULL})$ node remains structurally available; whether honest proposers ever re-select it is outside the protocol's enforcement scope.
- **Proposer payment delivered.** Path B settles the payment if the slot-N beacon block reached the quorum (Lemma 20 case (a)); the colluders cannot suppress this.
- **No slashing risk for honest validators.** Following `get_head` honestly is never a slashable offense, regardless of which branch wins.

The colluders therefore extract no economic advantage at the proposer's expense — they merely produce a wrong FULL/EMPTY decision at slot N+1 and force the canonical chain onto the EMPTY branch. Whether this matters in practice depends on the application (e.g., whether the slot-N execution payload contained transactions that downstream applications expected to be canonical).

---

#### Consistency, persistence, and liveness properties

The properties proved so far establish what the fork-choice rule produces in specific scenarios (honest case, adversarial case, payment settlement). The following lemmas close the remaining gaps: they prove that the protocol's internal bookkeeping is consistent, that structural invariants survive network-level events (reorgs, out-of-order delivery), and that the chain makes progress under honest majority.

---

**Lemma 18** (Honest builder parentStatus consistency). *If an honest builder constructs a bid for slot $N+1$ on top of block $B$ at slot $N$, and $B$'s builder revealed a valid execution payload (i.e., `B.root ∈ store.payloads` at the bid-construction time), then the honest builder's bid has `bid.parent_block_hash = bid(B).block_hash`, and therefore `parentStatus(B') = FULL` (Definition 4) for the resulting block $B'$. Conversely, if $B$'s payload was not revealed (i.e., `B.root ∉ store.payloads`), the honest builder's bid has `parentStatus(B') = EMPTY`.*

This ensures that honest builders' FULL/EMPTY declarations match reality: if the parent's execution payload was delivered, the honest builder declares FULL.

*Proof.* The honest builder constructs its bid by invoking the `submit_bid` handler (Section 12.2). Line 3 reads `parent_block_hash ← state.latest_block_hash`. The subtlety: at the time the slot-$N+1$ builder constructs its bid, no block has yet executed `apply_parent_execution_payload` for $B$'s execution payload — that helper runs *inside* `process_block` of the slot-$N+1$ block, which doesn't exist yet (the builder is constructing the bid for it). So `state.latest_block_hash` as held in `store.block_states[B.root]` has *not* been updated to `bid(B).block_hash` yet.

This is resolved by the spec function `prepare_execution_payload` (`validator.md`), which the builder calls before constructing the bid. `prepare_execution_payload` operates on a *copy* of `store.block_states[B.root]`, and on that copy it locally simulates `apply_parent_execution_payload` whenever `should_extend_payload(store, B.root)` returns True (i.e., when the builder is building on the FULL branch). The simulation calls Helper 12b, which at line 29 sets `state.latest_block_hash = parent_bid.block_hash` where `parent_bid = state.latest_execution_payload_bid` (cached as `bid(B)` by Helper 8 line 27 during slot-$N$'s `process_block`).

We split into two cases on the builder's intent:

**Case 1: The builder is building on $B$-FULL.** This is rational precisely when `B.root ∈ store.payloads` (otherwise the FULL node does not exist by Lemma 1, the resulting block would fail `on_block`'s assertion at Algorithm 7 line 5, and the builder gains nothing). The builder calls `prepare_execution_payload` with the FULL branch selected, the simulation runs Helper 12b on the state copy, and on the copy `state.latest_block_hash = bid(B).block_hash`. Line 3 of `submit_bid` reads this value, so `bid.parent_block_hash = bid(B).block_hash`. By Definition 4, `parentStatus(B') = FULL`.

**Case 2: The builder is building on $B$-EMPTY.** An honest builder runs `get_head` (via `prepare_execution_payload` in the `submit_bid` handler) immediately before constructing the bid, and extends whatever payload status `get_head` selects — the honest-behavior code does not branch on builder policy. So the honest builder reaches Case 2 if and only if `get_head` returned $(B.\mathit{root}, \mathrm{EMPTY})$ when run on the local store. By Lemma 1, $(B.\mathit{root}, \mathrm{FULL})$ exists as a child of $(B.\mathit{root}, \mathrm{PENDING})$ if and only if $B.\mathit{root} \in \mathit{store.payloads}$; by Lemma 13, `get_head` cannot return $(B.\mathit{root}, \mathrm{FULL})$ when $B.\mathit{root} \notin \mathit{store.payloads}$. Therefore Case 2 corresponds exactly to $B.\mathit{root} \notin \mathit{store.payloads}$. In that case, `should_extend_payload(store, B.root)` returns False (the hard guard at Algorithm 6 line 2 fails), the simulation skips the `apply_parent_execution_payload` call, and the state copy retains its pre-slot-$N$ value of `latest_block_hash`. Specifically, `latest_block_hash` was last updated by some earlier block's `apply_parent_execution_payload`, to that earlier block's `bid.block_hash` — call it $h_{\mathrm{prev}}$ with $h_{\mathrm{prev}} \neq$ `bid(B).block_hash` (different by the Definition 4 disjunction). Line 3 reads $h_{\mathrm{prev}}$, so `bid.parent_block_hash` $= h_{\mathrm{prev}} \neq$ `bid(B).block_hash`, and by Definition 4, `parentStatus(B') = EMPTY`.

Eventually, when slot-$N+1$'s block $B'$ is processed by all honest nodes, `process_parent_execution_payload` (Helper 12c) runs deterministically: it compares `bid(B').parent_block_hash` against the on-state `state.latest_execution_payload_bid.block_hash = bid(B).block_hash`, and either applies $B$'s execution effects (Case 1, FULL match) or asserts emptiness of `parent_execution_requests` and returns (Case 2, EMPTY match). The simulation result and the network's deterministic execution agree because both operate on the same state copy and bid data. ∎

---

**Lemma 18b** (Data availability for chain inclusion). *Let $B$ be a beacon block with root $r$ at slot $N$ that is canonical in the consensus chain, and let $h := \mathit{bid}(B).\mathit{block\_hash}$. Under (i) the per-committee Byzantine bound $\beta < 20\%$, (ii) network synchrony with delay $\Delta < T_{\mathrm{att}}$, and (iii) an honest super-majority of slot-$N+1$ proposers and attesters (in particular, an honest slot-$N+1$ proposer):*

*$h$ is in the payload hash chain of the canonical beacon chain (Definition 4c) — equivalently, $\mathit{block\_status}(\mathit{canonical}, B) = \text{FULL}$ — if and only if the execution payload corresponding to $h$ and its associated blob data were both delivered to the honest super-majority (so that $r \in \mathit{store.payloads}$ at every honest node, including the slot-$N+1$ proposer).*

**The (⇒) direction of this lemma is the formal proof of P4 (Data availability for chain inclusion) from §4 of the overview** — P4 itself claims "$h$ on chain ⇒ payload + blob data available and valid". The reverse direction (⇐: DA delivered ⇒ $h$ on chain) is also proved here as the stronger biconditional fact: under the same hypotheses, the data-availability gate is *equivalent* to membership in the payload hash chain, not merely necessary for it. This is the biconditional between the hash-chain membership of $h$ (Definition 4c) and the data-availability gate (which requires both payload and blob data to populate `store.payloads`).

*Proof.* We prove both directions.

*Forward direction (DA missing $\Rightarrow$ B is EMPTY on chain).* Suppose either the payload or the sampled blob data is unavailable to the honest super-majority. By Algorithm 8 (`on_execution_payload_envelope`) lines 4–7, populating `store.payloads[r]` requires (i) the payload to arrive on gossip (without which the handler is never invoked) and (ii) `is_data_available(envelope.beacon_block_root)` to pass before `verify_execution_payload_envelope` is called. If either condition fails for an honest node, that node never populates `store.payloads[r]`. By assumption, this holds for the honest super-majority — in particular for the honest slot-$N+1$ proposer (assumption (iii)).

The honest slot-$N+1$ proposer runs the `propose` handler (§12.2, Proposer), which consults `should_extend_payload(store, r)` to decide what `parent_execution_requests` to include and which bid to select. By Lemma 18 Case 2, when $r \notin \mathit{store.payloads}$ at the proposer's node, `should_extend_payload(store, r)` returns False at the hard guard (Algorithm 6 line 2 fails because `is_payload_verified(store, r)` evaluates `r ∈ store.payloads`, which is False). The proposer therefore selects a bid whose `bid.parent_block_hash` equals the previous execution payload tip $h_{\mathrm{prev}} \neq \mathit{bid}(B).\mathit{block\_hash}$ (consistent with what `prepare_execution_payload` simulates locally for the EMPTY branch). By Definition 4, the slot-$N+1$ block's $\mathit{parentStatus}(B'_{N+1}) = \mathrm{EMPTY}$.

No later honest proposer can declare $B$ FULL either: by Lemma 13, a malicious proposer cannot force FULL for an execution payload absent from `store.payloads` (the `on_block` assertion at Algorithm 7 line 5 fails at every honest node admitting the block), and an honest proposer following Lemma 18 declares EMPTY whenever $r \notin \mathit{store.payloads}$ locally. Therefore no canonical successor $C$ satisfies $\mathit{bid}(C).\mathit{parent\_block\_hash} = \mathit{bid}(B).\mathit{block\_hash}$, and by the §3 definition $B$ is EMPTY on chain.

*Reverse direction (B is FULL on chain $\Rightarrow$ DA was confirmed).* Suppose $B$ is FULL on chain: there exists a canonical block $C$ at some slot $M > N$ with $\mathit{bid}(C).\mathit{parent\_block\_hash} = \mathit{bid}(B).\mathit{block\_hash}$.

*Step 1 — B's direct canonical consensus child declared B FULL.* Let $B^*$ be the direct canonical consensus child of $B$ (i.e., $B^*.\mathit{parent\_root} = B.\mathit{root}$ and $B^*$ is on the canonical chain — necessarily exists, since $C$ extends from $B$ in the consensus chain and there is a unique canonical descendant at every slot after $B$). We claim $\mathit{bid}(B^*).\mathit{parent\_block\_hash} = \mathit{bid}(B).\mathit{block\_hash}$, equivalently $\mathit{parentStatus}(B^*) = \mathrm{FULL}$. Suppose for contradiction that $B^*$ declared EMPTY for $B$. Then in $B^*$'s `process_block`, Helper 12c's empty-parent branch runs and `state.latest_block_hash` is *not* updated. Subsequent canonical descendants of $B^*$ each read `state.latest_execution_payload_bid` cached by their parent (Helper 8 line 27 stores the parent's own bid), and Helper 8 line 21 asserts each descendant's `bid.parent_block_hash == state.latest_block_hash`. Tracing forward from $B^*$: `state.latest_block_hash` never equals $\mathit{bid}(B).\mathit{block\_hash}$ on the canonical chain past $B^*$ (because no apply call ever sets it to that value — only $B^*$ would have been able to, and we assumed it did not). But then $C$'s admission via Helper 8 line 21 must fail: $\mathit{bid}(C).\mathit{parent\_block\_hash} = \mathit{bid}(B).\mathit{block\_hash} \neq \mathit{state.latest\_block\_hash}$ at C's processing time. Contradiction with $C$ canonical. Therefore $\mathit{parentStatus}(B^*) = \mathrm{FULL}$.

*Step 2 — Algorithm 7 line 5 fires at $B^*$'s admission.* Because $\mathit{parentStatus}(B^*) = \mathrm{FULL}$, `is_parent_node_full(store, B*)` returns True at every honest node, and Algorithm 7 line 5 asserts `is_payload_verified(store, B^*.parent_root) = is_payload_verified(store, B.root)` — i.e., $B.\mathit{root} \in \mathit{store.payloads}$. Since $B^*$ is canonical, the honest super-majority of attesters voted in $B^*$'s branch; in particular, the honest super-majority admitted $B^*$, and therefore had $B.\mathit{root} \in \mathit{store.payloads}$ at $B^*$'s admission time. By Lemma 23 (store persistence), this remains true at every later time, including time $M$.

The only path to populating $\mathit{store.payloads}[B.\mathit{root}]$ is Algorithm 8 line 7, reached only after Algorithm 8 line 4 (`is_data_available(envelope.beacon_block_root)`) and line 6 (`verify_execution_payload_envelope`) both succeed. Both checks are honest-node-local; the first asserts blob data availability via PeerDAS sampling and KZG verification, and the second verifies the execution payload. Hence the honest super-majority observed both the payload and the blob data. ∎

*Remark on the role of the PTC.* Neither direction of the biconditional relies on PTC voting directly. The load-bearing structural fact is Algorithm 7's `is_payload_verified` assertion (forward direction's honest-proposer step and reverse direction's block-admission step), not the PTC tiebreaker. PTC majority is, however, *implicitly* consistent with assumption (iii) — Lemma H4 (A4) ensures that honest PTC members vote `payload_present` and `blob_data_available` reflecting their local DA state, so under honest super-majority the PTC tally tracks the underlying data-availability picture. If the reader prefers to make the PTC role explicit, it can be added as a strengthening of assumption (iii) without weakening the conclusion.

*Remark on PeerDAS partial availability.* The phrase "delivered to the honest super-majority" abstracts the PeerDAS sampling model: under PeerDAS, each node samples a column subset of the full data, and the *aggregate* honest super-majority's sampled columns are reconstructible (the column sampling sizes are calibrated so that an honest super-majority's union meets the reconstruction threshold). The local `is_data_available(envelope.beacon_block_root)` check is per-node: it returns True if the node's *own* sampled columns arrived and pass KZG verification. Lemma 18b's premise is that this local check succeeds for every honest super-majority node; that premise is in turn delivered by PeerDAS reconstructibility under the synchrony and Byzantine bounds (i)–(ii) when the builder honestly distributes the columns. A formal treatment of the PeerDAS reconstruction layer is outside this document's scope (it lives in `fulu/peer-data-availability-sampling.md`); the lemma takes the per-node `is_data_available` outcome as input.

*Remark on the role of fallbacks.* The fallback conditions (a), (b), (c) in `should_extend_payload` do not weaken P4. Under an honest slot-$N+1$ proposer (assumption (iii)), only fallback (c) — "the proposer declares the parent FULL" — can fire while $r \notin \mathit{store.payloads}$ at the proposer's node, and that fallback fires only if the proposer itself declares FULL, which by Lemma 18 they will not do without `store.payloads[r]`. Under a *malicious* slot-$N+1$ proposer (a case not covered by assumption (iii)), fallback (c) could in principle fire dishonestly, but the resulting block would not satisfy Algorithm 7 line 5 at honest nodes (since `is_payload_verified` returns False), and would not be admitted by the honest super-majority. P4's honest-majority assumption is what closes this loop.

---

**Lemma 19** (Honest proposer includes correct parent execution requests). *If an honest proposer builds a block B' on top of block B, and B' declares `parentStatus = FULL`, then `B'.body.parent_execution_requests` equals the execution requests from B's verified payload. If B' declares `parentStatus = EMPTY`, then `B'.body.parent_execution_requests` is an empty `ExecutionRequests()`.*

This ensures that `process_parent_execution_payload` (Helper 12c) will not reject an honest proposer's block at the `hash_tree_root` check.

*Proof.* By the `propose` handler (Section 12.2, Proposer), lines 27–30: if `should_extend_payload(store, parent_root)` is true (the proposer is building on B-FULL), the proposer sets `body.parent_execution_requests = store.payloads[parent_root].execution_requests`. The payload in `store.payloads[parent_root]` was stored by `on_execution_payload_envelope` (Algorithm 8, line 7) only after `verify_execution_payload_envelope` (Helper 12) verified the payload. During that verification, Helper 12 line 17 asserted `hash_tree_root(envelope.execution_requests) == bid.execution_requests_root`. Therefore, the execution requests in the stored payload satisfy the root commitment. When `process_parent_execution_payload` (Helper 12c) runs for B', line 9 checks `hash_tree_root(requests) == parent_bid.execution_requests_root` — this holds because the requests are the same object whose root was verified at payload-verification time.

If `should_extend_payload(store, parent_root)` is false (building on EMPTY), the proposer sets `body.parent_execution_requests = ExecutionRequests()`. When Helper 12c runs, it detects the empty-parent case at line 5 and asserts `requests == ExecutionRequests()` at line 6 — which holds. ∎

---

**Lemma 20** (Payment quorum reachability). *Let β < 20% denote the per-committee Byzantine bound. Let p denote the fraction of honest committee members that are online and attest for the block. The payment quorum is met if:*

$$
p \geq \frac{3/5 - \alpha}{1 - \beta}
$$

*where α is the fraction of the committee weight contributed by Byzantine validators who attest for the block (0 ≤ α ≤ β). In particular: (a) if all honest attesters participate (p = 1) and β < 20%, the quorum is met regardless of adversarial behavior; (b) if Byzantine validators attest normally (α = β = 20%), only `p ≥ (3/5 - 1/5) / (4/5) = 50%` of honest attesters need participate; (c) if Byzantine validators withhold attestations (α = 0, β = 20%), the requirement is `p ≥ (3/5) / (4/5) = 75%`.*

*Proof.* The quorum threshold is `(total_active_balance / SLOTS_PER_EPOCH) × 6 / 10` (Helper 18). Let `W_slot = total_active_balance / SLOTS_PER_EPOCH` denote the average per-slot committee weight. By `process_attestation` (Helper 15, lines 28–32), each same-slot attester — honest or Byzantine — that sets at least one new participation flag contributes their `effective_balance` to `payment.weight`. The total `payment.weight` is therefore:

`payment.weight = (p × (1 - β) + α) × W_slot`

where `p × (1 - β) × W_slot` is the weight from online honest attesters, and `α × W_slot` is the weight from Byzantine attesters who choose to attest. The quorum is met when:

`(p × (1 - β) + α) × W_slot ≥ (3/5) × W_slot`

which simplifies to `p ≥ (3/5 - α) / (1 - β)`.

For case (a): if p = 1 and α = 0 (worst case — adversary withholds), the honest contribution is `(1 - β) × W_slot`. Since β < 20%, this is at least `0.8 × W_slot > (3/5) × W_slot = 0.6 × W_slot`. The quorum is met.

For case (b): with α = β = 1/5, the requirement is `p ≥ (3/5 - 1/5) / (4/5) = (2/5) / (4/5) = 1/2 = 50%`.

For case (c): with α = 0 and β = 1/5, the requirement is `p ≥ (3/5) / (4/5) = 3/4 = 75%`. ∎

*Remark 1.* The β < 20% bound is a per-committee assumption. With Ethereum's current validator set (~1M validators) and `SLOTS_PER_EPOCH = 32`, each committee has ~31,250 validators, and the Byzantine fraction in any committee concentrates tightly around the global Byzantine fraction by standard concentration inequalities.

*Remark 2 (Interpretation).* The quorum threshold (60%) is a design parameter that balances two concerns. Setting it lower makes the unconditional payment more robust to low participation but makes it easier for a malicious proposer to extract payment from the builder with less network support. Setting it higher protects the builder better but requires more participation for the guarantee to hold. The 60% value is deliberately below the 80% honest lower bound (under β < 20%), ensuring the quorum is always reachable when all honest validators participate (case a), while providing meaningful protection against poorly-supported blocks. Under β < 20%, the quorum equals the builder's reveal threshold (40%) plus the adversary budget (20%): if the builder sees < 40% and withholds, the adversary cannot fill the gap to reach 60%. Case (c) — where Byzantine validators actively withhold attestations to prevent the quorum — represents adversarial collusion with the builder and is itself a form of builder protection: the builder's allies prevent payment for a block the builder does not want to pay for.

---

**Lemma 21** (Payment index correctness). *For a block at slot N with `bid.value > 0`: the protocol's index arithmetic in `process_execution_payload_bid` (Helper 8) and `apply_parent_execution_payload` (Helper 12b) is consistent — Path A settles the correct IOU in the same-epoch and previous-epoch cases, and falls back to a direct withdrawal append in the older-than-previous-epoch case, never confusing one slot's payment with another's.*

*Proof.* Helper 8 (line 26) writes to index `SLOTS_PER_EPOCH + bid.slot % SLOTS_PER_EPOCH` in the upper (current-epoch) half of the 2-epoch `builder_pending_payments` array. When a subsequent block processes the parent's execution payload via Helper 12b, the payment-handling branch is selected by the parent's epoch:

- **Case 1 (parent in current epoch):** `parent_epoch == get_current_epoch(state)`. Helper 12b computes `payment_index = SLOTS_PER_EPOCH + parent_slot % SLOTS_PER_EPOCH`. Since `parent_slot = bid.slot`, this is the same index Helper 8 wrote to. `settle_builder_payment` clears and settles the correct entry.
- **Case 2 (parent in previous epoch — one epoch boundary crossed):** `parent_epoch == get_previous_epoch(state)`. At the epoch boundary, `process_builder_pending_payments` (Helper 14) shifted the array: the entry has moved from the upper half (index `SLOTS_PER_EPOCH + bid.slot % SLOTS_PER_EPOCH`) to the lower half (index `bid.slot % SLOTS_PER_EPOCH`). Helper 12b computes `payment_index = parent_slot % SLOTS_PER_EPOCH`, which is the shifted position of the same entry. `settle_builder_payment` clears and settles the correct entry.
- **Case 3 (parent older than previous epoch):** `parent_epoch < get_previous_epoch(state)`. By this point the original IOU entry has been evicted from `builder_pending_payments` by two or more epoch-boundary shifts. Helper 12b bypasses the array and appends a `BuilderPendingWithdrawal` directly, reading `fee_recipient`, `amount = parent_bid.value`, and `builder_index` from the still-cached `state.latest_execution_payload_bid` (the bid was cached unconditionally by Helper 8 line 27 and has not been overwritten because — by definition of Case 3 reachability — no intervening block was processed; see the mutual-exclusion remark in Lemma 12). The withdrawal queued in this case is for the same builder, fee recipient, and amount as the original bid would have settled to, so the payment delivered to the proposer is unchanged.

In all three cases the payment delivered for slot N's bid is correctly attributed to slot N's `(builder_index, fee_recipient, value)` — no slot's payment is confused with another's. ∎

---

**Lemma 22** (Proposer equivocation bounds). *If a proposer broadcasts two conflicting blocks B and B' for the same slot, each containing a different builder's bid, then: (i) honest nodes detect the equivocation and the proposer is subject to slashing; (ii) the payment quorum for each block's bid is harder to reach because honest attesters split across the two blocks; (iii) neither builder can be charged more than `bid.value` — each bid's payment is independent; (iv) if the `ProposerSlashing` is included on-chain within the 2-epoch window, the pending payment is cleared entirely.*

*Proof.*

*(i) Equivocation detection:* When an honest node receives two `SignedBeaconBlock` messages for the same slot from the same `proposer_index`, it detects the equivocation (the two blocks have different roots but the same slot and proposer). The evidence constitutes a valid `ProposerSlashing` — the two conflicting `signed_header` messages satisfy all verification conditions in `process_proposer_slashing` (Helper 22): same slot (line 5), same proposer index (line 6), different headers (line 7), and valid signatures (line 9, abstracted as `# Verify both signatures ...`; verified explicitly in the spec via per-header BLS checks). The equivocating proposer is slashed and eventually exited.

*(ii) Payment quorum impact:* With two competing blocks B and B' at slot N, honest attesters run `get_head` and each selects one — let $h_B$ and $h_{B'}$ denote the honest fractions (of total committee weight $W$) that select B and B' respectively, with $h_B + h_{B'} \leq 1 - \beta$ (the honest total fraction; equality holds only if every honest attester votes for one of $\{B, B'\}$, no abstentions). Adversarial attesters can each contribute to at most one of $\{B, B'\}$ without equivocating; let $\alpha_B$ and $\alpha_{B'}$ be their contributions, with $\alpha_B, \alpha_{B'} \geq 0$ and $\alpha_B + \alpha_{B'} \leq \beta$.

When `process_attestation` (Helper 15, lines 28–32) runs for an attestation referencing `root(B)`, it accumulates weight only on B's `BuilderPendingPayment` entry (indexed by `SLOTS_PER_EPOCH + slot % SLOTS_PER_EPOCH` in B's state). The payment weights are therefore $\mathit{weight}_B = (h_B + \alpha_B)\, W$ and $\mathit{weight}_{B'} = (h_{B'} + \alpha_{B'})\, W$.

The quorum threshold is $0.6\, W$ (Helper 18). For both blocks to reach quorum we need:

$$
h_B + \alpha_B \;\geq\; 0.6 \quad \text{AND} \quad h_{B'} + \alpha_{B'} \;\geq\; 0.6.
$$

Adding the two:

$$
(h_B + h_{B'}) + (\alpha_B + \alpha_{B'}) \;\geq\; 1.2.
$$

Substituting the upper bounds $h_B + h_{B'} \leq 1 - \beta$ and $\alpha_B + \alpha_{B'} \leq \beta$:

$$
1.2 \;\leq\; (h_B + h_{B'}) + (\alpha_B + \alpha_{B'}) \;\leq\; (1 - \beta) + \beta \;=\; 1.
$$

But $1.2 \leq 1$ is a contradiction. **Therefore at most one of B, B' can have its quorum met, regardless of the value of $\beta$.** (The strict bound $\beta < 0.2$ used elsewhere in the document is not even needed for this part of the argument; the result follows from the structural fact that an honest validator does not double-vote and an adversarial validator does not contribute to both sides simultaneously without equivocating.)

If neither block individually reaches the $0.6\, W$ quorum threshold, neither builder is charged via Path B (Lemma 16).

*(iii) Payment independence:* On any canonical chain, exactly one of B or B' is included (they compete for the same slot). The block that is included has its `BuilderPendingPayment` created by `process_execution_payload_bid` (Helper 8). The other block's bid is never processed in the canonical state, so no IOU exists for it. A builder whose bid was not included in the canonical chain is not charged — no entry was ever created.

*(iv) Payment clearing via slashing:* If the `ProposerSlashing` evidence is included in a block within the 2-epoch `builder_pending_payments` window, `process_proposer_slashing` (Helper 22, lines 11--18) clears the `BuilderPendingPayment` for the equivocating slot. This erases the IOU entirely — even if the quorum was met, the builder is not charged. Under $\beta < 20\%$, honest proposers control $> 80\%$ of slots, so the slashing evidence is included within a few slots. ∎

---

**Lemma 23** (Fork-choice store persistence). *Once `store.payloads[r]` is populated for a root r, it is never removed. Once `store.blocks[r]` and `store.block_states[r]` are populated, they are never removed. Consequently, the structural invariants of Lemmas 1–5 are monotonic: if a FULL node exists at time t, it exists at all times t' > t.*

This ensures that the protocol's structural guarantees are permanent — a validated execution payload cannot be "unvalidated" by later events.

*Proof.* The store is modified by four handlers: `on_block` (Algorithm 7), `on_execution_payload_envelope` (Algorithm 8), `on_payload_attestation_message` (Algorithm 9), and `update_checkpoints` / `compute_pulled_up_tip` (called at the end of `on_block`).

- `on_block` (Algorithm 7) writes to `store.blocks[root]`, `store.block_states[root]`, `store.payload_timeliness_vote[root]`, and `store.payload_data_availability_vote[root]`. It never deletes any existing entry.
- `on_execution_payload_envelope` (Algorithm 8) writes to `store.payloads[root]`. It never deletes any existing entry.
- `on_payload_attestation_message` (Algorithm 9) writes to individual PTC vote entries. It never deletes blocks, states, or payloads.
- `update_checkpoints` and `compute_pulled_up_tip` modify checkpoint fields but not the block/state/payload dictionaries.

No handler contains a `del store.payloads[r]`, `del store.blocks[r]`, or `del store.block_states[r]` operation. Therefore, once an entry is written, it persists.

By Lemma 1, the FULL node for root r exists if and only if `r ∈ store.payloads`. Since entries are never removed from `store.payloads`, if the FULL node exists at time t, it exists at all t' > t. The same monotonicity applies to `get_node_children` (Algorithm 2), `is_supporting_vote` (Algorithm 4), and `get_weight` (Algorithm 5), which read from but never delete store entries. ∎

*Verification note.* The proof's claim that the four listed handlers exhaust all writers to `store.{blocks, block_states, payloads, payload_timeliness_vote, payload_data_availability_vote}` should be cross-checked against the upstream spec ([`fork-choice.md`](https://github.com/ethereum/consensus-specs/blob/master/specs/gloas/fork-choice.md)). A useful spot-check is `grep -nE "^\\s*store\\.(blocks|block_states|payloads|payload_timeliness_vote|payload_data_availability_vote)\\[" fork-choice.md` — every write site should land inside one of the four handlers above. If a future spec revision adds a new handler that mutates these fields (e.g., for fork transition, garbage collection, or finality-driven cleanup), Lemma 23 must be revisited. As of the spec version targeted by this document (`fa8bb08`), no such additional writer exists.

*Remark.* This persistence property ensures that after PTC+proposer collusion forces EMPTY for one slot, `store.payloads[r]` still exists and the FULL node remains structurally available. Whether the FULL branch actually recovers depends on whether an honest proposer builds on it (Lemma 17, Part (ii)). Regardless, the proposer's payment is guaranteed via Path B if the quorum was met (Lemma 10).

---

**Lemma 24** (Liveness under β < 20%). *If (i) β < 20% (the per-committee Byzantine bound), (ii) at least one honest proposer exists in every epoch, (iii) the network delivers messages within the slot timing bounds (T_att = 25%, T_ptc = 75%) — i.e., the synchrony delay $\Delta$ satisfies $\Delta < T_{\mathrm{att}}$ — and (iv) every honest validator follows the protocol per Section 12.2: in particular, every honest validator assigned to a slot's committee attests at $T_{\mathrm{att}}$ for the `get_head` head, every honest PTC member votes at $T_{\mathrm{ptc}}$ per Lemma H4, and every honest builder reveals per Assumption H7 when its preconditions are met; then: the consensus chain grows by at least one block per epoch, the execution chain grows whenever the slot's builder is honest and the block is canonical, and every honest builder's payment is settled within the 2-epoch `builder_pending_payments` window.*

*Proof.* By induction on epochs. Let epoch $e$ satisfy assumptions (i)--(iii), and let slot $N$ be a slot in epoch $e$ with an honest proposer (exists by assumption (ii)).

**Step 1: Block production.** The honest proposer at slot $N$ runs `get_head`, selects a valid bid (or self-builds with `BUILDER_INDEX_SELF_BUILD`), runs `state_transition` locally to compute the state root, and broadcasts the `SignedBeaconBlock` at $t = 0$.

**Step 2: Block becomes canonical.** By assumption (iii), all honest attesters receive block $B$ before $T_{\text{att}} = 3\text{s}$. Each honest attester runs `get_head` and sees $B$ as the head — it is either the only block for slot $N$, or the heaviest (honest validators control $\geq 80\%$ of the committee weight by (i), and the proposer boost of 40% further favors $B$). They attest with `data.beacon_block_root = root(B)`. The accumulated honest weight ($\geq 80\%$) exceeds any competing block's weight (at most $20\%$ Byzantine + proposer boost if the competing block arrived first, which it did not since the honest proposer broadcasts at $t = 0$). Block $B$ is canonical.

**Step 3: Execution chain progress (if builder is honest).** If $B$'s builder is honest, it reveals the `SignedExecutionPayloadEnvelope` after the cautious-reveal preconditions of Assumption H7 are met: (a) $B$ is locally canonical (H5), and (b) at least $\texttt{PROPOSER\_SCORE\_BOOST} = 40\%$ of the slot-N committee's effective balance has been seen attesting for $B$.

Under assumption (i) ($\beta < 0.2$): the honest fraction of committee weight is $1 - \beta > 0.8$. By Step 2, all honest slot-N validators vote for $B$ at $T_{\mathrm{att}} = 3\,\text{s}$. By assumption (iii), these attestations are received by $T_{\mathrm{att}} + \Delta < T_{\mathrm{ptc}} = 9\,\text{s}$. The accumulated honest attestation weight for $B$ thus crosses the 40% threshold strictly before $T_{\mathrm{ptc}}$ (in fact, well before, since $0.8 > 0.4$). The cautious-reveal preconditions are therefore satisfiable, and the honest builder broadcasts the payload at some time $t_{\mathrm{rev}} \in [T_{\mathrm{att}}, T_{\mathrm{ptc}})$.

By assumption (iii) (synchrony), the payload reaches all honest nodes by $t_{\mathrm{rev}} + \Delta \leq T_{\mathrm{ptc}}$. Honest PTC members observe it and vote `payload_present = True` and `blob_data_available = True` (Lemma H4). The number of honest PTC members is at least $\lceil (1 - \beta) \cdot 512 \rceil > 0.8 \cdot 512 = 409.6$, i.e., at least $410$. Each casts a `True` vote on both arrays. Since $\texttt{PAYLOAD\_TIMELY\_THRESHOLD} = \lfloor 512 / 2 \rfloor = 256$ and $410 > 256$, both `is_payload_timely` (Helper 4) and `is_payload_data_available` (Helper 5) return True.

At slot $N+1$, `should_extend_payload` (Algorithm 6) returns True via the PTC primary path (line 6). The tiebreaker (Definition 13) favors FULL: the priority key for $(B, \text{FULL})$ is 2, vs. 1 for $(B, \text{EMPTY})$. By Lemma 7 Part (b), no slot-$N+1$ proposer can reorg $B$ via boost alone, so $(B, \text{FULL})$ is canonical for every honest validator at slot $N+1$. The next honest proposer builds on $(B, \text{FULL})$. Inside `process_block` (Helper 11), `process_parent_execution_payload` (Helper 12c) detects FULL, verifies the execution requests root, and calls `apply_parent_execution_payload` (Helper 12b), which processes execution requests, settles the payment, and updates `state.latest_block_hash = bid(B).block_hash`. The execution chain advances.

**Step 4: Payment settlement.** Two cases:

*(a) Builder reveals (Path A).* By Step 3, a subsequent block builds on $(B, \text{FULL})$. By Lemma 18, the honest builder's `parentStatus` declaration is correct. By Lemma 21, the payment index computed by `process_execution_payload_bid` (Helper 8, line 26: `SLOTS_PER_EPOCH + slot % SLOTS_PER_EPOCH`) matches the index read by `settle_builder_payment` (Helper 12a) inside `apply_parent_execution_payload` (Helper 12b). The payment is queued in `builder_pending_withdrawals` and the `BuilderPendingPayment` is cleared (Lemma 9). By Lemma 12, the epoch-boundary check produces no second payment.

*(b) Builder withholds (Path B).* The `BuilderPendingPayment` persists. Same-slot attesters for slot $N$ contribute their effective balances to `payment.weight` via `process_attestation` (Helper 15, lines 28--32). By Lemma 20, under $\beta < 20\%$ and assuming $\geq 75\%$ of honest validators are online, the accumulated weight meets the 60% quorum. At the epoch boundary, `process_builder_pending_payments` (Helper 14) checks `payment.weight >= quorum` and queues the payment. The payment settles within the current or next epoch — well within the 2-epoch `builder_pending_payments` window.

**Step 5: Balance debit.** In both cases, the payment is queued in `builder_pending_withdrawals`. During a subsequent block's `process_withdrawals` (Helper 13), `apply_withdrawals` (Helper 21) debits the builder's balance: `state.builders[builder_index].balance -= min(amount, balance)`. The builder had sufficient balance at bid time (Helper 6 verified solvency via Helper 20, which accounts for all outstanding obligations).

**Inductive step.** By assumption (ii), epoch $e+1$ also has at least one honest proposer. Steps 1--5 apply to that epoch, producing at least one more canonical block. The consensus chain grows by at least one block per epoch. If the builder at the honest proposer's slot is also honest, the execution chain grows at that slot. Payments settle within the 2-epoch window. ∎

*Remark.* The assumption of at least one honest proposer per epoch is standard in Ethereum's security model. Under $\beta < 20\%$ and uniformly random proposer selection across $\sim 1\text{M}$ validators, the probability that all 32 slots in an epoch have Byzantine proposers is $(0.2)^{32} \approx 4 \times 10^{-23}$ — negligible.

---

#### Adversarial model: β < 20% per-committee bound

The properties above assume **β < 20%**: the Byzantine fraction in any single slot's committee is less than 20% of the committee weight. This section establishes why 20% is the relevant threshold for ePBS and how the spec parameters are calibrated.

**Three parameters, one constraint.** The ePBS builder safety mechanism involves three interlocking parameters:

| Parameter       | Value | Spec constant                                                                                | Role                                                  |
| --------------- | ----- | -------------------------------------------------------------------------------------------- | ----------------------------------------------------- |
| Proposer boost  | 40%   | `PROPOSER_SCORE_BOOST = 40`                                                                | Extra fork-choice weight for timely block             |
| Reorg threshold | 20%   | `REORG_HEAD_WEIGHT_THRESHOLD = 20`                                                         | Below this, equivocation protection activates         |
| Payment quorum  | 60%   | `BUILDER_PAYMENT_THRESHOLD_NUMERATOR = 6` / `BUILDER_PAYMENT_THRESHOLD_DENOMINATOR = 10` | Minimum attestation weight for epoch-boundary payment |

These are calibrated to each other via: **payment_quorum = reveal_threshold + adversary_budget = 40% + 20% = 60%**.

The reveal threshold (40%) is the attestation weight at which a block is stable against the next proposer's boost. The adversary budget (20%) is the maximum Byzantine attestation weight per committee. Their sum (60%) is the quorum: if the quorum is met, honest attestation must be ≥ 40% (since the adversary contributes at most 20%), meaning the builder should have seen enough support to reveal. If the quorum is not met, the adversary alone could not have filled the gap, so the builder's withholding was justified and it is not charged.

---

#### Attack analysis: proposer equivocation + boost

The most dangerous attack against builder payment safety is proposer equivocation combined with the next-slot proposer boost:

1. Slot N: adversarial proposer broadcasts block B (containing honest builder's bid) at t = 0. B is timely, receives the proposer boost (40%).
2. The builder sees B as the head (boost applied) and reveals the execution payload.
3. The same proposer broadcasts B' (equivocation). Adversarial attesters (β%) attest for B'.
4. Slot N+1: colluding proposer extends B' and receives the new proposer boost.

At slot N+1, the fork-choice compares:

- B: $(1 - \beta)\%$ honest attestation weight (no boost — boost moved to N+1)
- B': $\beta\%$ adversarial attestation + 40% boost = $(\beta + 40)\%$

B' wins when $\beta + 40 > 100 - \beta$, i.e., $\beta > 30\%$. Under $\beta < 20\%$, the attack fails: $20 + 40 = 60 < 80 = 100 - 20$.

**The `REORG_HEAD_WEIGHT_THRESHOLD = 20` defense.** The spec's `should_apply_proposer_boost` (`fork-choice.md`) adds an additional protection layer. When the boosted block's parent is from the immediately previous slot, the function checks `is_head_weak(store, parent_root)`: if the parent has < 20% of committee weight AND an equivocation from the parent's proposer is detected, the **boost is not applied**.

In the table below, the rows track the slot-N equivocation block $B'$ (the parent of the slot-N+1 boost-attacker block), and "B' weight" is the quantity `is_head_weak(store, parent_root)` examines — i.e., the attestation weight of $B'$ itself. The "B' wins?" column asks whether the branch rooted at $B'$ (with the slot-N+1 block extending it) overtakes the honest block $B$.

| $\beta$ range            | B' weight  | `is_head_weak`?            | Boost applied? | B' wins?                                                                                                                                                                                                     |
| -------------------------- | ---------- | ---------------------------- | -------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| $\beta < 20\%$           | < 20%      | Yes → equivocation detected | **No**   | **No** — B stays canonical                                                                                                                                                                            |
| $20\% \leq \beta < 30\%$ | ≥ 20%     | No                           | Yes            | **No** — $\beta + 40 < 100 - \beta$ strictly                                                                                                                                                        |
| $\beta = 30\%$           | $= 20\%$ | No                           | Yes            | **Lex tiebreak** — equal weights $\beta + 40 = 70 = 100 - \beta$; root with greater hash wins, so B' wins with probability $\approx \tfrac{1}{2}$ over a uniformly random adversarial root choice |
| $\beta > 30\%$           | ≥ 20%     | No                           | Yes            | **Yes** — $\beta + 40 > 100 - \beta$ strictly                                                                                                                                                       |

The attack thus succeeds deterministically only when $\beta > 30\%$; at the exact boundary $\beta = 30\%$ it succeeds with probability ${\sim}\tfrac{1}{2}$ over the lex tiebreak. The 20% threshold is where the spec's protection mechanism provides the first line of defense; the mathematical bound ($\beta < 30\%$ for deterministic safety) provides the second.

**Proposer equivocation and payment suppression.** If the proposer equivocates and a `ProposerSlashing` is included on-chain (within the 2-epoch `builder_pending_payments` window), `process_proposer_slashing` (Helper 22) clears the `BuilderPendingPayment` for that slot. This protects the builder: if it detected the equivocation and withheld, it is not charged at the epoch boundary. Under $\beta < 20\%$, honest proposers control > 80% of slots, so the `ProposerSlashing` is included within a few slots — well within the 2-epoch window.

---

#### Non-normative builder reveal strategy

The spec (`builder.md`) prescribes only the minimal honest rule: reveal when the block is "timely and the head of the builder's chain". This document strengthens that rule into the **cautious-reveal strategy** formalised as Assumption H7: reveal only after observing ≥ 40% real attestation weight AND no proposer equivocation. Assumption H7 contains the full statement, proof, and remarks; Lemma 7 Part (b) and Lemma 9 (second claim) show how the 40% threshold yields concrete anti-reorg and Path-A-fires guarantees under $\beta < 20\%$. The fork-choice mechanism (`should_apply_proposer_boost`, `is_head_weak`, dual timeliness tracking in `record_block_timeliness`) is calibrated to support this strategy.

---

### 12.5 Beacon-Chain Helpers and Processing Functions

The algorithms in Section 12.3 rely on several helper functions defined in `beacon-chain.md` and `fork-choice.md`. This section provides pseudocode and Python for each, grouped by size: tiny helpers first, then the processing functions that call them.

#### Helper 1: is_active_builder

```python
def is_active_builder(state, builder_index):
    builder = state.builders[builder_index]
    return (
        builder.deposit_epoch < state.finalized_checkpoint.epoch
        and builder.withdrawable_epoch == FAR_FUTURE_EPOCH
    )
```

A builder is active if two conditions hold: (1) the builder's deposit has been finalized — meaning `deposit_epoch` is strictly before the finalized checkpoint epoch, so the deposit cannot be reverted — and (2) the builder has not initiated a voluntary exit (`withdrawable_epoch` is still set to the far-future sentinel). Unlike validators, builders do not attest or propose; they only construct execution payloads. This function is called by `process_execution_payload_bid` (Helper 8) to verify that the builder behind a bid is eligible.

*Spec reference: `is_active_builder` in `beacon-chain.md`.*

#### Helper 2: parent_payload_revealed_in_state (pedagogical)

```python
def parent_payload_revealed_in_state(state):
    return state.latest_execution_payload_bid.block_hash == state.latest_block_hash
```

This is a pedagogical helper — **not a spec function** — used to make the FULL/EMPTY check inside `process_withdrawals` readable. The spec writes the equivalent comparison inline (see *Spec reference* below). We choose the name `parent_payload_revealed_in_state` to avoid collision with the real spec helper `is_parent_node_full(store, block)` (Helper 3 below, defined at `fork-choice.md` line 318), which has a different signature and operates on the fork-choice store rather than the BeaconState.

The comparison works because `latest_block_hash` is only updated by `apply_parent_execution_payload` (Helper 12b), which is called inside `process_parent_execution_payload` (Helper 12c) at the beginning of `process_block` (Helper 11) — if the parent's builder revealed and the current block declares FULL, `latest_block_hash` was set to the parent's `bid.block_hash`, so they match. If the builder withheld, `latest_block_hash` still holds a stale value from an earlier slot, so they differ. It is used by `process_withdrawals` (Helper 13) to decide whether to skip withdrawal processing (if the parent was empty, the withdrawal list depends on execution state that was never finalized).

*Spec reference: inline check in `process_withdrawals` in `beacon-chain.md`: `if state.latest_block_hash != state.latest_execution_payload_bid.block_hash: return`.*

#### Helper 3: get_parent_payload_status

```python
def get_parent_payload_status(store, block):
    parent = store.blocks[block.parent_root]
    parent_hash = block.body.signed_execution_payload_bid.message.parent_block_hash
    committed_hash = parent.body.signed_execution_payload_bid.message.block_hash
    return FULL if parent_hash == committed_hash else EMPTY
```

This is the fork-choice store version of the same FULL/EMPTY determination. It compares the block's declared `parent_block_hash` (the execution chain tip the builder read from state when constructing the bid) against the parent block's committed `block_hash` (the execution payload the parent's builder promised to deliver). If they match, the parent's execution payload was revealed and the execution chain advanced — FULL. If they differ, the parent's execution payload was never processed and the execution chain did not advance — EMPTY. This function is called by `get_node_children` (Algorithm 2, lines 8–9) to route blocks to the correct FULL or EMPTY branch, and by `get_ancestor` (Algorithm 3, line 9) to read the FULL/EMPTY declaration when walking back through the chain. It corresponds to `parentStatus` in Definition 4.

*Spec reference: `get_parent_payload_status` in `fork-choice.md`.*

#### Helper 4: is_payload_timely

```python
def is_payload_timely(store, root):
    assert root in store.payload_timeliness_vote
    if not is_payload_verified(store, root):
        return False                                               # Must have validated locally
    votes = store.payload_timeliness_vote[root]
    return sum(vote is True for vote in votes) > PAYLOAD_TIMELY_THRESHOLD
```

This function checks whether the execution payload for a given block was delivered on time. It requires two independent conditions: (1) the node itself must have locally received and validated the payload (`is_payload_verified(store, root)` — this prevents a node from favoring FULL for a payload it cannot verify), and (2) a majority of PTC members must have voted `payload_present = True` (more than `PAYLOAD_TIMELY_THRESHOLD`). The PTC votes are read from the scorecard populated by `on_payload_attestation_message` (Algorithm 9). This function is called by `should_extend_payload` (Algorithm 6, line 6) and corresponds to the first condition in Definition 11.

*Spec reference: `is_payload_timely` in `fork-choice.md`.*

#### Helper 5: is_payload_data_available

```python
def is_payload_data_available(store, root):
    assert root in store.payload_data_availability_vote
    if not is_payload_verified(store, root):
        return False                                               # Must have validated locally
    votes = store.payload_data_availability_vote[root]
    return sum(vote is True for vote in votes) > DATA_AVAILABILITY_TIMELY_THRESHOLD
```

Identical structure to `is_payload_timely` (Helper 4), but checks blob data availability instead of payload presence. PTC members vote on both dimensions independently: `payload_present` (did the execution payload arrive?) and `blob_data_available` (is the blob data available?). Both must pass for `should_extend_payload` (Algorithm 6, line 6) to return True via the primary path. This corresponds to the second condition in Definition 11.

*Spec reference: `is_payload_data_available` in `fork-choice.md`.*

#### Helper 6: can_builder_cover_bid

```python
def can_builder_cover_bid(state, builder_index, bid_amount):
    balance = state.builders[builder_index].balance
    pending = get_pending_balance_to_withdraw_for_builder(state, builder_index)
    min_balance = MIN_DEPOSIT_AMOUNT + pending
    if balance < min_balance:
        return False
    return balance - min_balance >= bid_amount
```

This function checks whether a builder has enough balance to cover a bid. The check is conservative: the builder must have enough to cover the bid amount *after* reserving `MIN_DEPOSIT_AMOUNT` (the minimum stake to remain active) and any already-queued pending withdrawals. This prevents a builder from bidding more than it can actually pay, which would break the unconditional payment guarantee (Lemma 11). The function is called by `process_execution_payload_bid` (Helper 8, line 10) during bid verification.

The `pending` amount (line 3) is computed by `get_pending_balance_to_withdraw_for_builder`, which sums **both** `builder_pending_withdrawals` (already-approved payments queued by Path A or Path B, waiting for `apply_withdrawals`) **and** `builder_pending_payments` (pending bids from the current and previous epoch that have not yet been settled). This means the check accounts for all of the builder's outstanding obligations, including bids for other slots that have been committed but not yet debited. A builder cannot overbid across concurrent slots: if a builder bids 80 ETH in slot N, then attempts to bid 80 ETH in slot N+1 with only 100 ETH balance, the second bid sees `pending = 80` (from slot N's entry in `builder_pending_payments`), computes `min_balance = MIN_DEPOSIT_AMOUNT + 80`, and rejects the bid because `100 - min_balance < 80`.

The `min(withdrawal.amount, builder_balance)` in `apply_withdrawals` (beacon-chain.md) is a defensive safety net that should never trigger given these upstream checks. It exists as a guard against edge cases (e.g., rounding or future spec changes) but is not the primary protection against overbidding.

*Spec reference: `can_builder_cover_bid` and `get_pending_balance_to_withdraw_for_builder` in `beacon-chain.md`.*

#### Helper 7: process_payload_attestation

```python
def process_payload_attestation(state, payload_attestation):
    data = payload_attestation.data
    assert data.beacon_block_root == state.latest_block_header.parent_root
    assert data.slot + 1 == state.slot                             # Must be from previous slot
    indexed = get_indexed_payload_attestation(state, payload_attestation)
    assert is_valid_indexed_payload_attestation(state, indexed)    # Verify aggregate signature
```

This function validates PTC votes that are included on-chain in a beacon block. It is called during `process_operations` (inside `process_block`) for each `PayloadAttestation` in the block body. Two key constraints are enforced: (1) the PTC vote must reference the parent block (`data.beacon_block_root = parent_root`), and (2) the vote must be from exactly the previous slot (`data.slot + 1 = state.slot`). The second constraint means PTC votes can only be included in the immediately next block — if that slot is missed, the votes are lost on-chain (though they still influenced fork-choice via gossip, as described in the Phase 4 timeline in Section 12.2). Line 5 converts the aggregated `PayloadAttestation` (with a bitfield over PTC positions) into an `IndexedPayloadAttestation` (with explicit validator indices), and line 6 verifies the aggregate BLS signature against those indices.

*Spec reference: `process_payload_attestation` in `beacon-chain.md`.*

#### Helper 8: process_execution_payload_bid

```python
def process_execution_payload_bid(state, block):
    bid = block.body.signed_execution_payload_bid.message
    builder_index = bid.builder_index
    amount = bid.value
    if builder_index == BUILDER_INDEX_SELF_BUILD:                  # Self-build path
        assert amount == 0
        assert block.body.signed_execution_payload_bid.signature == G2_POINT_AT_INFINITY
    else:                                                          # External builder path
        assert is_active_builder(state, builder_index)             # Helper 1
        assert can_builder_cover_bid(state, builder_index, amount) # Helper 6
        assert verify_execution_payload_bid_signature(state, block.body.signed_execution_payload_bid)
    assert len(bid.blob_kzg_commitments) <= get_blob_parameters(get_current_epoch(state)).max_blobs_per_block
    assert bid.slot == block.slot
    assert bid.parent_block_hash == state.latest_block_hash
    assert bid.parent_block_root == block.parent_root
    assert bid.prev_randao == get_randao_mix(state, get_current_epoch(state))
    if amount > 0:                                                 # Record pending payment
        payment = BuilderPendingPayment(
            weight=0,
            withdrawal=BuilderPendingWithdrawal(
                fee_recipient=bid.fee_recipient,
                amount=amount,
                builder_index=builder_index,
            ),
        )
        state.builder_pending_payments[SLOTS_PER_EPOCH + bid.slot % SLOTS_PER_EPOCH] = payment
    state.latest_execution_payload_bid = bid                       # Cache for later use
```

Under ePBS, the beacon block no longer contains the execution payload. Instead, it contains a **bid** — a promise from a builder saying "I will deliver execution payload X, and I'll pay you Y ETH for the privilege." This function verifies that promise and sets up the payment mechanism. It runs inside `process_block` (the consensus-layer state transition) for every beacon block, regardless of whether the builder later reveals the execution payload.

**Lines 2–4:** Extract the bid fields from the block body.

**Lines 5–11: Who is bidding?** There are two possibilities. If `builder_index = BUILDER_INDEX_SELF_BUILD`, the proposer is acting as its own builder — no external builder is involved. In this case, the bid value must be zero (the proposer does not pay itself) and the signature is the BLS infinity point (a sentinel meaning "no real signer"). Otherwise, an external builder submitted the bid. The function checks: is the builder active (Helper 1)? Can the builder afford the bid (Helper 6)? Is the signature valid? If any check fails, the bid is rejected and the entire beacon block is invalid.

**Lines 12–16: Is the bid consistent with the current chain state?** Four checks ensure the bid matches reality:

- Line 13: the bid is for this slot, not some other slot.
- Line 14: the builder is building on the correct execution chain tip (`bid.parent_block_hash == state.latest_block_hash`). This is the assertion that enforces consistency between the builder's declared world and the actual state.
- Line 15: the builder is building on the correct beacon chain parent.
- Line 16: the RANDAO mix is consistent.

If any of these fail, the entire beacon block is invalid and rejected by the network.

**Lines 17–26: Set up the payment.** If the bid value is > 0, the function creates a `BuilderPendingPayment` with `weight = 0` and stores it. This is like writing an IOU: "builder owes proposer Y ETH." At this point, nobody has been paid yet. What happens next depends on whether the builder reveals:

- **Builder reveals** (Path A): the next block's `process_parent_execution_payload` (Helper 12c) calls `apply_parent_execution_payload` (Helper 12b), which calls `settle_builder_payment` (Helper 12a) to queue the payment and clear the IOU.
- **Builder withholds** (Path B): The pending payment sits there. Meanwhile, as attesters vote for this beacon block, `process_attestation` (Helper 15) accumulates their effective balances into `payment.weight`. At the epoch boundary, if enough weight accumulated (60% quorum), `process_builder_pending_payments` (Helper 14) queues the payment — the proposer gets paid even though the builder didn't deliver.

**Line 27:** Caches the bid in state for later use by `verify_execution_payload_envelope` (Helper 12, which verifies the revealed execution payload matches the committed bid), `process_parent_execution_payload` (Helper 12c, which reads the parent bid to apply execution effects), and `parent_payload_revealed_in_state` (Helper 2).

In one sentence: this function verifies the builder's promise is valid and arms the unconditional payment mechanism.

*Spec reference: `process_execution_payload_bid` in `beacon-chain.md`.*

#### Helper 9: compute_ptc

```python
def compute_ptc(state, slot):
    epoch = compute_epoch_at_slot(slot)
    seed = hash(get_seed(state, epoch, DOMAIN_PTC_ATTESTER) + uint_to_bytes(slot))
    indices = []
    for i in range(get_committee_count_per_slot(state, epoch)):
        indices.extend(get_beacon_committee(state, slot, CommitteeIndex(i)))
    return compute_balance_weighted_selection(
        state, indices, seed, size=PTC_SIZE, shuffle_indices=False
    )
```

This function computes the Payload Timeliness Committee for a given slot from scratch. It concatenates all beacon committees for the slot into a single ordered list (lines 4–6), then selects `PTC_SIZE = 512` members via balance-weighted sampling (line 7): the list is traversed in order, and each validator is accepted with probability proportional to their effective balance, using a deterministic seed derived from the slot number. The `shuffle_indices=False` parameter means the traversal order follows the committee ordering, not a random shuffle. PTC members are therefore a subset of the slot's attestation committees and still perform their normal attester duties. The result is cached in `state.ptc_window` (see Helper 10) to avoid recomputation.

*Spec reference: `compute_ptc` in `beacon-chain.md`.*

#### Helper 10: get_ptc

```python
def get_ptc(state, slot):
    epoch = compute_epoch_at_slot(slot)
    state_epoch = get_current_epoch(state)
    if epoch < state_epoch:                                        # Previous epoch
        assert epoch + 1 == state_epoch
        return state.ptc_window[slot % SLOTS_PER_EPOCH]
    assert epoch <= state_epoch + MIN_SEED_LOOKAHEAD               # Current or next epoch
    offset = (epoch - state_epoch + 1) * SLOTS_PER_EPOCH
    return state.ptc_window[offset + slot % SLOTS_PER_EPOCH]
```

This function retrieves the precomputed PTC for a given slot from the cached `ptc_window` array in the BeaconState. The `ptc_window` is a flat array of `(2 + MIN_SEED_LOOKAHEAD) × SLOTS_PER_EPOCH` entries covering the previous epoch, current epoch, and lookahead epochs. The function maps the requested slot to the correct index: previous-epoch slots map to indices `0` through `SLOTS_PER_EPOCH - 1`, current-epoch slots to `SLOTS_PER_EPOCH` through `2 × SLOTS_PER_EPOCH - 1`, and next-epoch slots to `2 × SLOTS_PER_EPOCH` through `3 × SLOTS_PER_EPOCH - 1`. The function is called by `on_payload_attestation_message` (Algorithm 9, line 6) to verify that a PTC voter is actually a PTC member for that slot, and by `process_payload_attestation` (Helper 7) during on-chain validation.

*Spec reference: `get_ptc` in `beacon-chain.md`.*

#### Helper 11: process_block

```python
def process_block(state, block):
    process_parent_execution_payload(state, block)                 # New: apply parent's EL effects
    process_block_header(state, block)
    process_withdrawals(state)                                     # Helper 13; may return early
    process_execution_payload_bid(state, block)                    # Helper 8
    process_randao(state, block.body)
    process_eth1_data(state, block.body)
    process_operations(state, block.body)                          # Includes Helper 7
    process_sync_aggregate(state, block.body.sync_aggregate)
```

This is the top-level state transition function. It is called by `on_block` (Algorithm 7, line 11) every time a beacon block is received. Under ePBS, three changes are visible compared to pre-ePBS:

1. **`process_parent_execution_payload` is added as the first step** (line 2). This applies the *parent's* execution effects — EL requests (deposits, withdrawals, consolidations), builder payment settlement via `settle_builder_payment`, and `latest_block_hash` update — if the block declares FULL. It must run before `process_execution_payload_bid` (line 5) because the bid verification reads `state.latest_execution_payload_bid`, which `process_parent_execution_payload` leaves unchanged but whose consistency depends on `latest_block_hash` being up to date.
2. **`process_execution_payload` is removed.** Pre-ePBS, the execution payload was embedded in the beacon block and processed here. Under ePBS, the current slot's execution payload is verified separately by `on_execution_payload_envelope` (Algorithm 8) in the fork-choice layer.
3. **`process_execution_payload_bid` is added** (line 5). This verifies the builder's bid and arms the unconditional payment mechanism (Helper 8).

The remaining calls are unchanged from pre-ePBS: block header validation, withdrawals (modified — see Helper 13), RANDAO, Eth1 data, operations (which now includes `process_payload_attestation` for on-chain PTC votes — Helper 7), and sync committee processing. The order matters: `process_parent_execution_payload` (line 2) must run before `process_execution_payload_bid` (line 5) because the latter overwrites `state.latest_execution_payload_bid`.

*Spec reference: `process_block` in `beacon-chain.md`.*

#### Helper 12: verify_execution_payload_envelope

```python
def verify_execution_payload_envelope(state, signed_envelope, execution_engine):
    envelope = signed_envelope.message
    payload = envelope.payload
    # Verify signature
    assert verify_execution_payload_envelope_signature(state, signed_envelope)
    # Verify consistency with the beacon block
    header = copy(state.latest_block_header)
    header.state_root = hash_tree_root(state)
    assert envelope.beacon_block_root == hash_tree_root(header)
    assert envelope.parent_beacon_block_root == state.latest_block_header.parent_root
    # Verify consistency with the committed bid
    bid = state.latest_execution_payload_bid                       # Cached by Helper 8
    assert envelope.builder_index == bid.builder_index
    assert payload.prev_randao == bid.prev_randao
    assert payload.gas_limit == bid.gas_limit
    assert payload.block_hash == bid.block_hash                    # Payload matches commitment
    assert hash_tree_root(envelope.execution_requests) == bid.execution_requests_root
    # Verify the execution payload is valid
    assert payload.slot_number == state.slot
    assert payload.parent_hash == state.latest_block_hash          # Execution chain continuity
    assert payload.timestamp == compute_time_at_slot(state, state.slot)
    assert hash_tree_root(payload.withdrawals) == hash_tree_root(state.payload_expected_withdrawals)
    versioned_hashes = [kzg_commitment_to_versioned_hash(c) for c in bid.blob_kzg_commitments]
    assert execution_engine.verify_and_notify_new_payload(
        NewPayloadRequest(payload, versioned_hashes, envelope.parent_beacon_block_root,
                          envelope.execution_requests)
    )
```

This is the execution payload verification function — the second phase of the two-phase model. It is called by `on_execution_payload_envelope` (Algorithm 8, line 6) when the builder's payload arrives separately from the beacon block. This function **only verifies** the payload — it does not mutate state, process EL requests, settle payments, or update `latest_block_hash`. Those effects are deferred to the *next* block's `process_parent_execution_payload` (Helper 12c). If this function completes successfully, Algorithm 8 stores the payload in `store.payloads`, which enables the FULL branch in the fork-choice tree. If it fails at any assertion, `store.payloads` is never populated and the FULL branch never exists.

**Lines 4–5: Signature verification.** The payload signature is verified against the builder's public key.

**Lines 6–10: Verification against the beacon block.** A copy of the block header is created with the state root filled in (lines 7–8), and the payload must reference this exact block (line 9). The payload's `parent_beacon_block_root` must match the block header's `parent_root` (line 10), ensuring the payload is tied to the correct chain position.

**Lines 11–17: Verification against the committed bid.** The function reads the bid that was cached by `process_execution_payload_bid` (Helper 8, line 27) during beacon block processing. It verifies that the revealed execution payload matches what the builder committed to: same builder (line 13), same RANDAO (line 14), same gas limit (line 15), same block hash (line 16 — the core commitment check), and same execution requests root (line 17).

**Lines 18–22: Execution payload validity.** The payload's `slot_number` must match the state slot (line 19). Execution chain continuity is verified: `parent_hash` must equal `state.latest_block_hash` (line 20). Timestamp must be correct (line 21). Withdrawals must match the state's expected withdrawals (line 22).

**Lines 23–27: Execution engine.** The execution payload is sent to the EL for full EVM validation — valid transactions, correct state root, etc. The `NewPayloadRequest` uses `envelope.parent_beacon_block_root` (not directly from the state header) to pass the parent beacon block root to the execution engine. If the EL rejects it, the assertion fails and the function aborts.

*Spec reference: `verify_execution_payload_envelope` in `fork-choice.md`.*

#### Helper 12a: settle_builder_payment

```python
def settle_builder_payment(state, payment_index):
    assert payment_index < len(state.builder_pending_payments)
    payment = state.builder_pending_payments[payment_index]
    if payment.withdrawal.amount > 0:
        state.builder_pending_withdrawals.append(payment.withdrawal)
    state.builder_pending_payments[payment_index] = BuilderPendingPayment()
```

This function settles a builder's pending payment by moving it from the `builder_pending_payments` array to the `builder_pending_withdrawals` queue. It is called by `apply_parent_execution_payload` (Helper 12b) when the next block processes the parent's execution effects. If the payment amount is positive (line 3–4), the withdrawal is queued for later processing by `apply_withdrawals`. Regardless of the amount, the pending payment entry is cleared (line 5) — this prevents double payment at the epoch boundary (Lemma 12).

*Spec reference: `settle_builder_payment` in `beacon-chain.md`.*

#### Helper 12b: apply_parent_execution_payload

```python
def apply_parent_execution_payload(state, requests):
    parent_bid = state.latest_execution_payload_bid
    parent_slot = parent_bid.slot
    parent_epoch = compute_epoch_at_slot(parent_slot)
    # Process execution requests from parent's execution payload
    for deposit in requests.deposits:
        process_deposit_request(state, deposit)
    for withdrawal in requests.withdrawals:
        process_withdrawal_request(state, withdrawal)
    for consolidation in requests.consolidations:
        process_consolidation_request(state, consolidation)
    # Settle the builder payment
    if parent_epoch == get_current_epoch(state):
        payment_index = SLOTS_PER_EPOCH + parent_slot % SLOTS_PER_EPOCH
        settle_builder_payment(state, payment_index)               # Helper 12a
    elif parent_epoch == get_previous_epoch(state):
        payment_index = parent_slot % SLOTS_PER_EPOCH
        settle_builder_payment(state, payment_index)               # Helper 12a
    elif parent_bid.value > 0:
        state.builder_pending_withdrawals.append(
            BuilderPendingWithdrawal(
                fee_recipient=parent_bid.fee_recipient,
                amount=parent_bid.value,
                builder_index=parent_bid.builder_index,
            )
        )
    # Update parent execution payload availability and latest block hash
    state.execution_payload_availability[parent_slot % SLOTS_PER_HISTORICAL_ROOT] = 1
    state.latest_block_hash = parent_bid.block_hash
```

This function applies the execution-layer effects of the parent block's execution payload. It reads `parent_bid` from `state.latest_execution_payload_bid` internally (line 2), rather than receiving it as a parameter. It has two callers:

1. **During block processing:** called by `process_parent_execution_payload` (Helper 12c) inside `process_block` (Helper 11) when a block declares FULL. This is the "real" execution that all nodes perform when processing a beacon block.
2. **During block production:** called by `prepare_execution_payload` (spec: `validator.md`) on a **copy** of the state by builders and self-building proposers *before* the block is proposed. This local simulation determines the correct execution chain head (`latest_block_hash`) and withdrawals for the new execution payload. The simulation and the real execution produce identical results because both operate on the same inputs deterministically.

These effects were previously applied inside `verify_execution_payload_envelope` (the old Helper 12), but are now deferred to the *next* block's `process_block`.

**Lines 5–10: Execution-layer requests.** Processes deposits, withdrawals, and consolidation requests embedded in the parent's execution payload. These are the EL-originated state changes that modify the validator set and pending queues.

**Lines 12–26: Builder payment settlement (Path A).** Settles the builder's pending payment via `settle_builder_payment` (Helper 12a). The payment index depends on whether the parent slot is in the current or previous epoch. If the parent is older than the previous epoch (an edge case at fork boundaries), the payment is queued directly. This is the mechanism that implements Path A of unconditional payment: the builder revealed, so the proposer is paid in the next block without waiting for the epoch boundary.

**Lines 28–29: State updates.** Marks the parent's slot as having a revealed execution payload in `execution_payload_availability` (line 28), and advances `latest_block_hash` to the parent's `block_hash` (line 29). The `latest_block_hash` update is critical: it is read by the next bid's consistency check in `process_execution_payload_bid` (Helper 8, line 14).

*Spec reference: `apply_parent_execution_payload` in `beacon-chain.md`.*

#### Helper 12c: process_parent_execution_payload

```python
def process_parent_execution_payload(state, block):
    bid = block.body.signed_execution_payload_bid.message
    parent_bid = state.latest_execution_payload_bid
    requests = block.body.parent_execution_requests
    if bid.parent_block_hash != parent_bid.block_hash:
        assert requests == ExecutionRequests()                      # No EL requests for empty parent
        return
    # Parent was FULL — verify commitment and apply effects
    assert hash_tree_root(requests) == parent_bid.execution_requests_root
    apply_parent_execution_payload(state, requests)                # Helper 12b
```

This function is the first step of `process_block` (Helper 11, line 2). It determines whether the parent block was FULL or EMPTY, and if FULL, applies the parent's execution effects.

**Lines 2–4:** Extract the current block's bid, the parent's cached bid (`state.latest_execution_payload_bid`), and the `parent_execution_requests` field from the block body. The `parent_execution_requests` is a new `BeaconBlockBody` field that carries the EL requests from the parent's execution payload — it is included by the proposer (see Section 12.2, Proposer).

**Lines 5–7: Empty parent.** If the parent was EMPTY (the block's `parent_block_hash` does not match the parent's committed `block_hash`), the function asserts that no execution requests are included (line 6) and returns. No execution effects are applied.

**Lines 9–10: Full parent.** The execution requests root is verified against the parent's committed `execution_requests_root` (line 9), and `apply_parent_execution_payload` (Helper 12b) is called to process the EL requests, settle the builder payment, and update `latest_block_hash`.

*Spec reference: `process_parent_execution_payload` in `beacon-chain.md`.*

#### Helper 13: process_withdrawals

```python
def process_withdrawals(state):
    if not parent_payload_revealed_in_state(state):
        return                                                     # Helper 2; skip if parent empty
    expected = get_expected_withdrawals(state)
    apply_withdrawals(state, expected.withdrawals)
    update_next_withdrawal_index(state, expected.withdrawals)
    update_payload_expected_withdrawals(state, expected.withdrawals)
    update_builder_pending_withdrawals(state, expected.processed_builder_withdrawals_count)
    update_pending_partial_withdrawals(state, expected.processed_partial_withdrawals_count)
    update_next_withdrawal_builder_index(state, expected.processed_builders_sweep_count)
    update_next_withdrawal_validator_index(state, expected.withdrawals)
```

This function processes validator and builder withdrawals during `process_block` (Helper 11, line 4). The key ePBS modification is **line 2: the early return**. If the parent block's execution payload was not revealed (`parent_payload_revealed_in_state` returns False — Helper 2), the function returns immediately and does nothing. This is because the withdrawal list depends on execution state (account balances, pending partial withdrawals) that was never finalized when the parent's execution payload was missing. Processing withdrawals against a stale execution state would produce incorrect results, so the protocol skips withdrawal processing entirely for blocks that build on an empty parent.

When the parent was full (line 2 passes), the function proceeds normally: it computes the expected withdrawals (line 3), applies them — debiting builder and validator balances (line 4) — and updates the various sweep indices and caches (lines 5–10). Line 6 is new to ePBS: it caches the expected withdrawals in `state.payload_expected_withdrawals`, which `verify_execution_payload_envelope` (Helper 12, line 22) later checks against the revealed execution payload's actual withdrawals to ensure consistency.

*Spec reference: `process_withdrawals` in `beacon-chain.md`.*

#### Helper 14: process_builder_pending_payments

```python
def process_builder_pending_payments(state):
    quorum = get_builder_payment_quorum_threshold(state)           # 60% of per-slot active balance
    for payment in state.builder_pending_payments[:SLOTS_PER_EPOCH]:
        if payment.weight >= quorum:                               # Enough attesters supported block
            state.builder_pending_withdrawals.append(payment.withdrawal)
    old_payments = state.builder_pending_payments[SLOTS_PER_EPOCH:]
    new_payments = [BuilderPendingPayment() for _ in range(SLOTS_PER_EPOCH)]
    state.builder_pending_payments = old_payments + new_payments
```

This function runs at the epoch boundary (during `process_epoch`) and implements **Path B** of the unconditional payment mechanism (Lemma 10). It handles the case where the builder withheld the execution payload: `settle_builder_payment` (Helper 12a) was never called (because no subsequent block declared FULL for this slot), so the pending payment was never cleared by Path A.

**Lines 2–5: Quorum check.** The function iterates over the previous epoch's pending payments (indices `0` through `SLOTS_PER_EPOCH - 1`). For each payment still pending (not already cleared by Path A), it checks whether enough attester weight accumulated (`payment.weight ≥ quorum`). The quorum threshold is 60% of the average per-slot active balance (`total_active_balance / SLOTS_PER_EPOCH × 6 / 10`). The weight was accumulated by `process_attestation` (Helper 15): each time a same-slot attester voted for the beacon block and set at least one new participation flag, their effective balance was added to `payment.weight`. If the quorum is met, the payment is queued — the proposer gets paid even though the builder did not deliver. If the quorum is not met (e.g., the beacon block was not widely attested, perhaps because it was equivocating or late), the payment is silently discarded — neither the proposer nor the builder pays.

**Lines 6–8: Epoch shift.** The `builder_pending_payments` array is a 2-epoch sliding window. The current epoch's entries (indices `SLOTS_PER_EPOCH` through `2 × SLOTS_PER_EPOCH - 1`) become the previous epoch's entries (indices `0` through `SLOTS_PER_EPOCH - 1`), and fresh empty entries are appended for the new epoch.

*Spec reference: `process_builder_pending_payments` in `beacon-chain.md`.*

#### Helper 15: process_attestation (modified)

```python
def process_attestation(state, attestation):
    data = attestation.data
    assert data.target.epoch in (get_previous_epoch(state), get_current_epoch(state))
    assert data.slot + MIN_ATTESTATION_INCLUSION_DELAY <= state.slot
    assert data.index < 2                                          # ePBS: only 0 or 1
    # ... committee/aggregation validation and the slot-vs-target consistency assert
    # `data.target.epoch == compute_epoch_at_slot(data.slot)` (unchanged from pre-ePBS;
    # see beacon-chain.md:1682 and the per-committee loop at 1687–1701). None of these
    # checks is load-bearing for the lemmas in §12.4; we elide them for clarity ...
    participation_flag_indices = get_attestation_participation_flag_indices(
        state, data, state.slot - data.slot                        # Helper 16
    )
    assert is_valid_indexed_attestation(state, get_indexed_attestation(state, attestation))
    # Select the correct payment slot
    if data.target.epoch == get_current_epoch(state):
        epoch_participation = state.current_epoch_participation
        payment = state.builder_pending_payments[SLOTS_PER_EPOCH + data.slot % SLOTS_PER_EPOCH]
    else:
        epoch_participation = state.previous_epoch_participation
        payment = state.builder_pending_payments[data.slot % SLOTS_PER_EPOCH]
    proposer_reward_numerator = 0
    for index in get_attesting_indices(state, attestation):
        will_set_new_flag = False
        for flag_index, weight in enumerate(PARTICIPATION_FLAG_WEIGHTS):
            if flag_index in participation_flag_indices and not has_flag(
                epoch_participation[index], flag_index
            ):
                epoch_participation[index] = add_flag(epoch_participation[index], flag_index)
                proposer_reward_numerator += get_base_reward(state, index) * weight
                will_set_new_flag = True
        # ePBS: accumulate payment weight for same-slot attestations
        if (will_set_new_flag
                and is_attestation_same_slot(state, data)
                and payment.withdrawal.amount > 0):
            payment.weight += state.validators[index].effective_balance
    # Reward proposer
    proposer_reward = proposer_reward_numerator // (
        (WEIGHT_DENOMINATOR - PROPOSER_WEIGHT) * WEIGHT_DENOMINATOR // PROPOSER_WEIGHT
    )
    increase_balance(state, get_beacon_proposer_index(state), Gwei(proposer_reward))
    # Store updated payment weight back
    if data.target.epoch == get_current_epoch(state):
        state.builder_pending_payments[SLOTS_PER_EPOCH + data.slot % SLOTS_PER_EPOCH] = payment
    else:
        state.builder_pending_payments[data.slot % SLOTS_PER_EPOCH] = payment
```

This function processes attestations included in beacon blocks. Most of the logic is unchanged from pre-ePBS; the ePBS modifications are:

**Line 5: Restricted `data.index`.** Pre-ePBS, `data.index` identified a committee (values 0 through 63). Under ePBS, it is restricted to `{0, 1}` and repurposed to signal payload status: `0` = empty/no opinion, `1` = full. Committee identification now uses `attestation.committee_bits` instead.

**Lines 7–9: Modified participation flags.** `get_attestation_participation_flag_indices` (Helper 16) now checks whether the attester's payload signal (`data.index`) matches the actual on-chain record (`execution_payload_availability`). If it doesn't match, the head vote is not considered correct and the attester misses the timely head reward.

**Lines 28–32: Builder payment weight accumulation.** This is entirely new to ePBS and is the mechanism that powers Path B of unconditional payment (Lemma 10). For each attesting validator, if (1) they set at least one new participation flag (`will_set_new_flag = True`, ensuring each validator contributes weight at most once per slot), (2) the attestation is a same-slot attestation (`is_attestation_same_slot(state, data)` — the attester voted for a block in their own slot), and (3) there is actually a pending payment for that slot (`payment.withdrawal.amount > 0`), then the validator's effective balance is added to `payment.weight`. This weight is what `process_builder_pending_payments` (Helper 14) checks against the 60% quorum at the epoch boundary.

*Spec reference: `process_attestation` in `beacon-chain.md`.*

#### Helper 16: get_attestation_participation_flag_indices (modified)

```python
def get_attestation_participation_flag_indices(state, data, inclusion_delay):
    if data.target.epoch == get_current_epoch(state):
        justified_checkpoint = state.current_justified_checkpoint
    else:
        justified_checkpoint = state.previous_justified_checkpoint
    is_matching_source = (data.source == justified_checkpoint)
    target_root = get_block_root(state, data.target.epoch)
    is_matching_target = is_matching_source and (data.target.root == target_root)
    # ePBS: check payload status signal
    if is_attestation_same_slot(state, data):
        assert data.index == 0                                     # Must signal "no opinion"
        payload_matches = True                                     # Always matches
    else:
        slot_index = data.slot % SLOTS_PER_HISTORICAL_ROOT
        payload_matches = (data.index == state.execution_payload_availability[slot_index])
    # Matching head now requires payload match too
    head_root = get_block_root_at_slot(state, data.slot)
    is_matching_head = (
        is_matching_target
        and data.beacon_block_root == head_root
        and payload_matches                                        # ePBS addition
    )
    assert is_matching_source
    participation_flag_indices = []
    if is_matching_source and inclusion_delay <= integer_squareroot(SLOTS_PER_EPOCH):
        participation_flag_indices.append(TIMELY_SOURCE_FLAG_INDEX)
    if is_matching_target:
        participation_flag_indices.append(TIMELY_TARGET_FLAG_INDEX)
    if is_matching_head and inclusion_delay == MIN_ATTESTATION_INCLUSION_DELAY:
        participation_flag_indices.append(TIMELY_HEAD_FLAG_INDEX)
    return participation_flag_indices
```

This function determines which participation rewards an attester earns (timely source, timely target, timely head). Most of the logic is unchanged from pre-ePBS. The ePBS modification is in **lines 4–11: payload status matching**.

**Lines 10–12: Same-slot attestations.** If the attester voted for a block in their own slot (`is_attestation_same_slot(state, data)` returns True), `data.index` must be `0` (line 11) — the attester has no payload opinion because attestation happens at 25% of the slot while the builder has until 75% (Definition 1). Since there is no opinion to check, `payload_matches` is set to `True` unconditionally (line 12). The attester is not penalized for something they cannot know.

**Lines 13–15: Non-same-slot attestations.** If the attester voted for a block from a previous slot (their slot had no block, so their head fell back to an older block), they can observe whether that block's execution payload was revealed. `data.index` signals their observation (`0` = empty, `1` = full). Line 15 checks this against `execution_payload_availability` — a bitvector in the BeaconState that records, for each recent slot, whether `verify_execution_payload_envelope` (Helper 12) actually ran. If the attester signals `1` (full) but the payload was never revealed (availability is `0`), the signals don't match.

**Lines 18–21: Modified head matching.** Pre-ePBS, a correct head vote required matching source, target, and head root. Under ePBS, it also requires `payload_matches`. An attester who signals the wrong payload status does not receive the timely head reward — even if their source, target, and head root are all correct.

*Spec reference: `get_attestation_participation_flag_indices` in `beacon-chain.md`.*

#### Helper 17: is_attestation_same_slot

```python
def is_attestation_same_slot(state, data):
    if data.slot == 0:
        return True
    block_root = data.beacon_block_root
    slot_block_root = get_block_root_at_slot(state, data.slot)
    prev_block_root = get_block_root_at_slot(state, Slot(data.slot - 1))
    return block_root == slot_block_root and block_root != prev_block_root
```

This function determines whether an attestation is for a block that was proposed in the attester's own slot (a same-slot attestation) or for an older block (a non-same-slot attestation). It checks two conditions: (1) the attested block root matches the block root at the attestation's slot, and (2) the attested block root differs from the block root at the previous slot — meaning a new block was actually proposed in this slot, not just inherited from the previous slot. If both hold, the attester is voting for a block from their own slot. This function is called by `process_attestation` (Helper 15) to determine whether to accumulate builder payment weight, and by `get_attestation_participation_flag_indices` (Helper 16) to determine whether to enforce `data.index == 0`.

*Spec reference: `is_attestation_same_slot` in `beacon-chain.md`.*

#### Helper 18: get_builder_payment_quorum_threshold

```python
def get_builder_payment_quorum_threshold(state):
    per_slot_balance = get_total_active_balance(state) // SLOTS_PER_EPOCH
    quorum = per_slot_balance * BUILDER_PAYMENT_THRESHOLD_NUMERATOR
    return quorum // BUILDER_PAYMENT_THRESHOLD_DENOMINATOR
```

This function computes the quorum threshold for Path B of the unconditional payment mechanism (Lemma 10). The threshold is 60% of the average per-slot active balance: `(total_active_balance / SLOTS_PER_EPOCH) × 6 / 10`, where `BUILDER_PAYMENT_THRESHOLD_NUMERATOR = 6` and `BUILDER_PAYMENT_THRESHOLD_DENOMINATOR = 10`. If the accumulated `payment.weight` from same-slot attestations (Helper 15, lines 28–32) meets or exceeds this threshold, the payment is queued at the epoch boundary (Helper 14, lines 3–5) — the proposer gets paid even though the builder withheld. The 60% threshold ensures that a builder cannot be forced to pay for a beacon block that the network did not widely support.

*Spec reference: `get_builder_payment_quorum_threshold` in `beacon-chain.md`.*

#### Helper 19: process_ptc_window

```python
def process_ptc_window(state):
    # Shift all epochs forward by one
    state.ptc_window[:len(state.ptc_window) - SLOTS_PER_EPOCH] = (
        state.ptc_window[SLOTS_PER_EPOCH:]
    )
    # Fill in the last epoch with freshly computed PTC assignments
    next_epoch = Epoch(get_current_epoch(state) + MIN_SEED_LOOKAHEAD + 1)
    start_slot = compute_start_slot_at_epoch(next_epoch)
    state.ptc_window[len(state.ptc_window) - SLOTS_PER_EPOCH:] = [
        compute_ptc(state, Slot(slot))
        for slot in range(start_slot, start_slot + SLOTS_PER_EPOCH)
    ]
```

This function runs at the epoch boundary (during `process_epoch`) and updates the cached PTC assignment window in the BeaconState. The `ptc_window` is a flat array of `3 × SLOTS_PER_EPOCH = 96` entries covering the previous, current, and next epoch. At each epoch transition, the function shifts the array left by one epoch (lines 3–5) — dropping the oldest epoch's assignments and moving the current epoch into the "previous" position — then fills in the newly visible lookahead epoch by calling `compute_ptc` (Helper 9) for each of its slots (lines 7–12). This ensures that validators always have PTC assignments available for the previous, current, and next epoch, which is what `get_ptc` (Helper 10) reads from.

**Example.** When `current_epoch = 10`, the `ptc_window` contains:

| Indices 0–31      | Indices 32–63     | Indices 64–95       |
| ------------------ | ------------------ | -------------------- |
| Epoch 9 (previous) | Epoch 10 (current) | Epoch 11 (lookahead) |

To read the PTC for slot 325 (epoch 10, slot index `325 % 32 = 5`), `get_ptc` (Helper 10) returns `ptc_window[32 + 5]`.

When epoch 11 starts, `process_ptc_window` shifts the array and computes new assignments:

| Indices 0–31 | Indices 32–63 | Indices 64–95              |
| ------------- | -------------- | --------------------------- |
| Epoch 10      | Epoch 11       | Epoch 12 (freshly computed) |

Epoch 9's assignments are dropped, epoch 10 moves to the "previous" position, epoch 11 moves to "current", and epoch 12 is freshly computed via `compute_ptc` (Helper 9) for each of its 32 slots.

*Spec reference: `process_ptc_window` in `beacon-chain.md`.*

#### Helper 20: get_pending_balance_to_withdraw_for_builder

```python
def get_pending_balance_to_withdraw_for_builder(state, builder_index):
    pending_withdrawals = sum(
        w.amount for w in state.builder_pending_withdrawals
        if w.builder_index == builder_index
    )
    pending_payments = sum(
        p.withdrawal.amount for p in state.builder_pending_payments
        if p.withdrawal.builder_index == builder_index
    )
    return pending_withdrawals + pending_payments
```

This function computes the total amount a builder owes across all outstanding obligations: both already-approved withdrawals (in `builder_pending_withdrawals`, queued by `settle_builder_payment` or `process_builder_pending_payments`) and pending IOUs (in `builder_pending_payments`, recorded by `process_execution_payload_bid`). It is called by `can_builder_cover_bid` (Helper 6) to ensure a builder cannot commit to a bid it cannot pay. The sum covers both Path A obligations (already settled into the withdrawal queue) and pending Path B obligations (IOUs that may or may not be settled at the epoch boundary).

*Spec reference: `get_pending_balance_to_withdraw_for_builder` in `beacon-chain.md`.*

#### Helper 21: apply_withdrawals (modified)

```python
def apply_withdrawals(state, withdrawals):
    for withdrawal in withdrawals:
        if is_builder_index(withdrawal.validator_index):
            builder_index = convert_validator_index_to_builder_index(withdrawal.validator_index)
            builder_balance = state.builders[builder_index].balance
            state.builders[builder_index].balance -= min(withdrawal.amount, builder_balance)
        else:
            decrease_balance(state, withdrawal.validator_index, withdrawal.amount)
```

This function performs the actual balance debit for both validator and builder withdrawals. Under ePBS, the function is modified to handle builder withdrawals: if `withdrawal.validator_index` falls in the builder index range (checked by `is_builder_index`), the withdrawal is applied to the builder's consensus-layer balance (`state.builders[builder_index].balance`) rather than a validator balance. The `min(withdrawal.amount, builder_balance)` prevents underflow. This is the final step of both Path A and Path B: `settle_builder_payment` (Helper 12a) and `process_builder_pending_payments` (Helper 14) queue payments into `builder_pending_withdrawals`, and `apply_withdrawals` debits them here during `process_withdrawals` (Helper 13).

*Spec reference: `apply_withdrawals` in `beacon-chain.md`.*

#### Helper 22: process_proposer_slashing (modified)

```python
def process_proposer_slashing(state, proposer_slashing):
    header_1 = proposer_slashing.signed_header_1.message
    header_2 = proposer_slashing.signed_header_2.message
    # Standard slashing checks (unchanged)
    assert header_1.slot == header_2.slot
    assert header_1.proposer_index == header_2.proposer_index
    assert header_1 != header_2
    assert is_slashable_validator(state.validators[header_1.proposer_index], get_current_epoch(state))
    # Per-header BLS signature verification (unchanged from pre-ePBS;
    # beacon-chain.md:1799–1805 iterates over both signed_headers and calls
    # bls.Verify with DOMAIN_BEACON_PROPOSER). Not load-bearing for the lemmas
    # in §12.4; elided here for readability.
    # ... verify both signatures ...
    # [New in Gloas:EIP7732] Clear the builder's pending payment
    slot = header_1.slot
    proposal_epoch = compute_epoch_at_slot(slot)
    if proposal_epoch == get_current_epoch(state):
        payment_index = SLOTS_PER_EPOCH + slot % SLOTS_PER_EPOCH
        state.builder_pending_payments[payment_index] = BuilderPendingPayment()
    elif proposal_epoch == get_previous_epoch(state):
        payment_index = slot % SLOTS_PER_EPOCH
        state.builder_pending_payments[payment_index] = BuilderPendingPayment()
    slash_validator(state, header_1.proposer_index)
```

The ePBS modification to `process_proposer_slashing` is the payment clearing logic (lines 11--18). When a proposer is caught equivocating (two different blocks signed for the same slot), the `BuilderPendingPayment` for that slot is cleared — replaced with an empty entry. This protects the builder: if the proposer equivocated and the builder withheld (because it detected the equivocation), the builder should not be charged at the epoch boundary. The clearing uses the same index arithmetic as `process_execution_payload_bid` (Helper 8, line 26) and `settle_builder_payment` (Helper 12a): current-epoch slots map to `SLOTS_PER_EPOCH + slot % SLOTS_PER_EPOCH`, previous-epoch slots map to `slot % SLOTS_PER_EPOCH`. If the equivocation is for a slot older than the previous epoch, the entry has already been evicted by the epoch shift, so no clearing is needed.

This is the mechanism that makes the cautious builder reveal strategy safe under equivocation: the builder detects the equivocation, withholds, and when the `ProposerSlashing` evidence is included on-chain, the IOU is erased.

*Spec reference: `process_proposer_slashing` in `beacon-chain.md`.*

#### Builder lifecycle helpers (Helpers 23–28)

Builders enter and leave the registry through a lifecycle distinct from validators. The helpers below formalize this lifecycle. They are referenced throughout the proofs (especially Helper 6's solvency check and the discussion of who is allowed to bid), but their internal logic is straightforward.

#### Helper 23: is_builder_index

```python
def is_builder_index(validator_index):
    return (validator_index & BUILDER_INDEX_FLAG) != 0
```

Distinguishes builder indices from validator indices in the unified withdrawal index space. Builders and validators share the type `ValidatorIndex` for withdrawals (so that `apply_withdrawals` can dispatch to either code path), but builder indices carry the bitwise marker `BUILDER_INDEX_FLAG = uint64(2**40)` (bit 40, *not* the high bit of a uint64; spec value at `beacon-chain.md` line 138). This single-bit test is used by `apply_withdrawals` (Helper 21) to route withdrawals to `state.builders` rather than `state.validators`. The unrelated sentinel `BUILDER_INDEX_SELF_BUILD = BuilderIndex(UINT64_MAX)` lives in `BuilderIndex` space and is never run through Helper 23 or Helper 24 — it is handled at the bid layer by `process_execution_payload_bid` (Helper 8 line 5), which short-circuits the self-build branch before any flag-arithmetic is needed.

*Spec reference: `is_builder_index` in `beacon-chain.md`.*

#### Helper 24: convert_builder_index_to_validator_index / convert_validator_index_to_builder_index

```python
def convert_builder_index_to_validator_index(builder_index):
    return ValidatorIndex(builder_index | BUILDER_INDEX_FLAG)

def convert_validator_index_to_builder_index(validator_index):
    return BuilderIndex(validator_index & ~BUILDER_INDEX_FLAG)
```

The two halves of the encoding established by Helper 23. Setting `BUILDER_INDEX_FLAG` (bit 40) produces a `ValidatorIndex` that `is_builder_index` recognizes; clearing it recovers the original `BuilderIndex`. Used by `get_builder_withdrawals` (when constructing withdrawals from `state.builders`) and `apply_withdrawals` (when dispatching back to `state.builders`). Note that these helpers operate on `BuilderIndex` values that originate from `state.builders` (i.e., real on-chain builders), never on the `BUILDER_INDEX_SELF_BUILD` sentinel, which is filtered out upstream.

*Spec reference: `convert_builder_index_to_validator_index` and `convert_validator_index_to_builder_index` in `beacon-chain.md`.*

#### Helper 25: is_builder_withdrawal_credential

```python
def is_builder_withdrawal_credential(withdrawal_credentials):
    return withdrawal_credentials[:1] == BUILDER_WITHDRAWAL_PREFIX
```

A deposit's `withdrawal_credentials` field selects whether the deposit creates a validator or a builder. The `BUILDER_WITHDRAWAL_PREFIX` (byte `0x03`) marks builder deposits. Used by `apply_pending_deposit` and `onboard_builders_from_pending_deposits` to route deposits to the correct registry.

*Spec reference: `is_builder_withdrawal_credential` in `beacon-chain.md`.*

#### Helper 26: get_index_for_new_builder

```python
def get_index_for_new_builder(state):
    for index, builder in enumerate(state.builders):
        if builder.withdrawable_epoch <= get_current_epoch(state) and builder.balance == 0:
            return BuilderIndex(index)
    return BuilderIndex(len(state.builders))
```

Builder indices are reusable: when an exited builder's balance reaches zero, its slot in `state.builders` becomes available for a new builder. This function scans the registry for the first reusable slot and returns its index; if none exists, it returns `len(state.builders)` (signaling that a new entry should be appended). Reusing indices keeps the `state.builders` array bounded.

*Spec reference: `get_index_for_new_builder` in `beacon-chain.md`.*

#### Helper 27: add_builder_to_registry / apply_deposit_for_builder

```python
def add_builder_to_registry(state, pubkey, withdrawal_credentials, amount, slot):
    set_or_append_list(
        state.builders,
        get_index_for_new_builder(state),
        Builder(
            pubkey=pubkey,
            version=uint8(withdrawal_credentials[0]),
            execution_address=ExecutionAddress(withdrawal_credentials[12:]),
            balance=amount,
            deposit_epoch=compute_epoch_at_slot(slot),
            withdrawable_epoch=FAR_FUTURE_EPOCH,
        ),
    )

def apply_deposit_for_builder(state, pubkey, withdrawal_credentials, amount, signature, slot):
    builder_pubkeys = [b.pubkey for b in state.builders]
    if pubkey not in builder_pubkeys:
        if is_valid_deposit_signature(pubkey, withdrawal_credentials, amount, signature):
            add_builder_to_registry(state, pubkey, withdrawal_credentials, amount, slot)
    else:
        builder_index = builder_pubkeys.index(pubkey)
        state.builders[builder_index].balance += amount
```

`apply_deposit_for_builder` handles both registration (new pubkey) and top-up (existing pubkey). For new pubkeys, the signature must be valid (proof-of-possession) — invalid signatures cause the deposit to be silently dropped. For existing pubkeys, the deposit is added to the existing balance with no signature check (the original registration already proved possession). `add_builder_to_registry` constructs the `Builder` entry with `withdrawable_epoch = FAR_FUTURE_EPOCH` (active state) and `balance = amount`.

The interaction with index reuse (Helper 26) is subtle: a newly registered builder may receive a `BuilderIndex` previously held by an exited builder with a different pubkey. This is safe because the registration overwrites the entire entry, but cached references to old `(BuilderIndex, pubkey)` mappings can become stale.

*Spec reference: `add_builder_to_registry` and `apply_deposit_for_builder` in `beacon-chain.md`.*

#### Helper 28: initiate_builder_exit

```python
def initiate_builder_exit(state, builder_index):
    builder = state.builders[builder_index]
    builder.withdrawable_epoch = get_current_epoch(state) + MIN_BUILDER_WITHDRAWABILITY_DELAY
```

Sets the exit timer for a builder. Once `withdrawable_epoch` is in the past, the builder's balance becomes available for the withdrawal sweep (`get_builders_sweep_withdrawals`), and after the balance reaches zero, the index becomes reusable (Helper 26). Crucially, setting `withdrawable_epoch` to a finite value also makes `is_active_builder` return False, which prevents the builder from being included in any new bids (`process_execution_payload_bid` at Helper 8 asserts `is_active_builder` for non-self-build bids).

This is the mechanism by which Lemma 15's solvency argument holds: an exiting builder cannot create new obligations, so the obligations tracked by `get_pending_balance_to_withdraw_for_builder` (Helper 20) are a complete picture at exit time.

*Spec reference: `initiate_builder_exit` in `beacon-chain.md`.*

#### Helper 29: is_pending_validator

```python
def is_pending_validator(state, pubkey):
    for pending_deposit in state.pending_deposits:
        if pending_deposit.pubkey != pubkey:
            continue
        if is_valid_deposit_signature(
            pending_deposit.pubkey,
            pending_deposit.withdrawal_credentials,
            pending_deposit.amount,
            pending_deposit.signature,
        ):
            return True
    return False
```

Used during deposit processing to check whether a pubkey is already claimed by a pending validator deposit. This prevents a builder deposit from creating a builder with a pubkey that a pending validator deposit will later try to use — the validator deposit takes precedence, and the corresponding builder deposit is rejected (or refunded). Relevant for the fork transition's `onboard_builders_from_pending_deposits` (`fork.md`), which handles the edge case where deposits accumulated before the Gloas fork need to be sorted into validators and builders.

*Spec reference: `is_pending_validator` in `beacon-chain.md`.*

#### Helper 30: compute_balance_weighted_selection

```python
def compute_balance_weighted_selection(state, indices, seed, size, shuffle_indices):
    MAX_RANDOM_VALUE = 2**16 - 1
    total = uint64(len(indices))
    assert total > 0
    effective_balances = [state.validators[index].effective_balance for index in indices]
    selected = []
    i = uint64(0)
    while len(selected) < size:
        offset = i % 16 * 2
        if offset == 0:
            random_bytes = hash(seed + uint_to_bytes(i // 16))
        next_index = i % total
        if shuffle_indices:
            next_index = compute_shuffled_index(next_index, total, seed)
        weight = effective_balances[next_index] * MAX_RANDOM_VALUE
        random_value = bytes_to_uint64(random_bytes[offset : offset + 2])
        threshold = MAX_EFFECTIVE_BALANCE_ELECTRA * random_value
        if weight >= threshold:
            selected.append(indices[next_index])
        i += 1
    return selected
```

The core balance-weighted sampling primitive. Given a candidate set `indices`, it samples `size` indices using rejection sampling: at each step, a candidate is selected (either in order or via `compute_shuffled_index`, depending on `shuffle_indices`), and accepted with probability `effective_balance / MAX_EFFECTIVE_BALANCE_ELECTRA`. The acceptance test compares `effective_balance × MAX_RANDOM_VALUE >= MAX_EFFECTIVE_BALANCE_ELECTRA × random_value`, where `random_value` is a 16-bit value derived from `hash(seed || counter)`.

A validator with effective balance `b` is selected with probability proportional to `b / MAX_EFFECTIVE_BALANCE_ELECTRA`. A validator at the maximum effective balance is always accepted; one at half the maximum is accepted half the time, etc. Sampling continues until `size` indices have been collected.

This function is called by:

- `compute_proposer_indices` with `size=1` and `shuffle_indices=True` (one proposer per slot, randomly drawn from the active set)
- `compute_ptc` (Helper 9) with `size=PTC_SIZE` (= 512) and `shuffle_indices=False` (PTC drawn from the slot's committees, in committee order)
- `get_next_sync_committee_indices` (similar pattern)

*Spec reference: `compute_balance_weighted_selection` in `beacon-chain.md`.*

#### Properties of balance-weighted sampling

The PTC and proposer selections inherit two key statistical properties from `compute_balance_weighted_selection`. We state them as lemmas because some of the security arguments rely on them, but their proofs are standard probability arguments that we sketch rather than develop in full.

**Lemma 25** (Balance-weighted sampling is unbiased, up to discretization). *Each validator in the candidate set is selected with probability $b / \text{MEB} + O(2^{-16})$, where $b$ is the validator's effective balance, $\text{MEB} = \text{MAX\_EFFECTIVE\_BALANCE\_ELECTRA}$, and the additive error accounts for the discrete 16-bit `random_value` used in the acceptance test.*

*Proof sketch.* Within `compute_balance_weighted_selection` (Helper 30), each candidate `i` is reached at some iteration `j` and accepted iff `effective_balance(i) × MAX_RANDOM_VALUE >= MAX_EFFECTIVE_BALANCE_ELECTRA × random_value`, where `random_value` is drawn uniformly from the $2^{16}$-element set $\{0, 1, \ldots, \text{MAX\_RANDOM\_VALUE}\} = \{0, 1, \ldots, 2^{16}-1\}$. Writing $b$ for `effective_balance(i)`, the acceptance count equals $\lfloor b\,(2^{16}-1)/\text{MEB}\rfloor + 1$, divided by $2^{16}$:

$$
P_{\text{accept}}(b) \;=\; \frac{\lfloor b\,(2^{16}-1)/\text{MEB}\rfloor + 1}{2^{16}} \;=\; \frac{b}{\text{MEB}} \;+\; \epsilon_b, \qquad |\epsilon_b| \leq 2^{-16}.
$$

Thus the probability is proportional to effective balance up to an additive error of order $2^{-16} \approx 1.5 \times 10^{-5}$. For the downstream concentration bounds (Lemma 26 and its instantiations), this discretization is absorbed by the Hoeffding error term and is negligible at any committee size of practical interest. ∎

**Lemma 26** (Per-committee Byzantine concentration from balance-weighted sampling). *Let $\beta'$ be the Byzantine fraction of effective balance in the active validator set. Let $C$ be a committee of size $k$ drawn by `compute_balance_weighted_selection` (Helper 30) using a per-slot seed, and let $\beta_C$ denote the Byzantine fraction within $C$ (a random variable over the seed). For any $\epsilon > 0$,*

$$
\Pr[\beta_C > \beta' + \epsilon] \;\leq\; \exp(-2 k\, \epsilon^2),
$$

*and symmetrically for the lower tail $\Pr[\beta_C < \beta' - \epsilon] \leq \exp(-2k\, \epsilon^2)$.*

*Proof sketch.* The seed enters as the input to a hash function used as a random oracle (`hash(seed + uint_to_bytes(i // 16))` in Helper 30 line 8); we treat the resulting bytes as uniformly random and independent across iterations $i$. Under this random-oracle abstraction, each loop iteration $i$ draws an independent uniform `random_value`, and acceptance is conditionally independent of all prior iterations given the candidate identity.

Let $Y_1, \ldots, Y_k$ be the indicator variables for "the $j$-th accepted member is Byzantine." Each $Y_j$ corresponds to one accepted candidate, drawn from the active validator set (with replacement) and accepted with probability $b/\text{MEB} + O(2^{-16})$ proportional to its effective balance (Lemma 25). The marginal distribution of $Y_j$ is therefore Bernoulli($\beta'$), where $\beta'$ is the Byzantine fraction of effective balance.

Joint independence requires more care. Because `shuffle_indices=False` for the PTC and the loop sweeps `indices` in committee order, the $j$-th accepted member's identity is correlated with which prior validators were rejected; however, the *Byzantine indicator* $Y_j$ depends only on whether the accepted validator belongs to the Byzantine effective-balance mass, and by symmetry of the random `random_value` draws across iterations, the $Y_j$ are exchangeable with the same marginal $\beta'$. Hoeffding's inequality holds for exchangeable bounded random variables (Hoeffding 1963, §6) — in fact it holds for sampling without replacement with negative correlation, and a fortiori here. Thus

$$
\Pr[X/k > \beta' + \epsilon] \;\leq\; \exp(-2k\,\epsilon^2),
$$

and symmetrically for the lower tail. $\blacksquare$

[^lem26-iid]: Two subtleties: (i) The random-oracle abstraction of `hash(seed + ...)` is standard but means the bound is in a probabilistic model with one round of trusted randomness; RANDAO biasability by Byzantine proposers is bounded to at most one bit per slot of seed manipulation, which slightly weakens the effective uniformity but is negligible at the committee sizes used. (ii) Sampling with replacement is the literal behavior of `compute_balance_weighted_selection`, which can produce duplicates; Hoeffding's bound for sampling with replacement equals the iid bound and is tight; the with-replacement structure is what gives the iid form used here.
    
#### Two operative instantiations of Lemma 26

ePBS uses Lemma 26 at two committee sizes, with two different operative thresholds. **Each threshold is determined by the mechanism it serves, not by Lemma 26 itself.**

**Instantiation A: PTC (size $k = 512$, threshold $\beta_{\text{PTC}} < 0.5$).** The PTC needs honest majority on `is_payload_timely` (Helper 4): $\sum \text{True} > 256$. This holds whenever $\beta_{\text{PTC}} < 0.5$.

| $\beta'$ (global) | Required deviation$\epsilon = 0.5 - \beta'$ | $\Pr[\beta_{\text{PTC}} \geq 0.5]$                | Verdict                                                    |
| ------------------- | --------------------------------------------- | --------------------------------------------------- | ---------------------------------------------------------- |
| $0.10$            | $0.40$                                      | $\exp(-1024 \cdot 0.16) \approx 7\times 10^{-72}$ | Astronomical safety.                                       |
| $0.20$            | $0.30$                                      | $\exp(-1024 \cdot 0.09) \approx 9\times 10^{-41}$ | Astronomical safety.                                       |
| $0.30$            | $0.20$                                      | $\exp(-1024 \cdot 0.04) \approx 2\times 10^{-18}$ | Negligible.                                                |
| $0.40$            | $0.10$                                      | $\exp(-10.24) \approx 3.6\times 10^{-5}$          | Rare (~once per 28k slots ≈ 4 days).                      |
| $0.45$            | $0.05$                                      | $\exp(-2.56) \approx 0.077$                       | Problematic; would fail PTC majority once every ~13 slots. |

**Reading this table:** PTC majority safety is robust for any global $\beta'$ well below $0.5$. The PTC is a *robust* witness committee — the document's standing $\beta < 0.2$ per-committee bound is **not what makes the PTC work**; the PTC continues to work at much higher Byzantine fractions.

**Instantiation B: slot attestation committee (size $k \approx 31{,}250$, threshold $\beta_{\text{slot}} < 0.2$).** With Ethereum's ~1M active validators distributed across 32 slots per epoch, each slot's combined attestation committees contain $\approx 31{,}250$ members. The payment-safety calibration (Section 12.4: $0.4 + \beta < 0.6$) requires per-slot-committee $\beta < 0.2$. The Hoeffding exponent is $2 \cdot 31{,}250 = 62{,}500$ — about $61\times$ tighter than the PTC instantiation.

*Note on the underlying lemma.* Slot attestation committees are **not** produced by `compute_balance_weighted_selection` (Helper 30); they are produced by `get_beacon_committee`, which uses uniform shuffling via `compute_shuffled_index`. Lemma 26 as stated above is therefore not the correct tool to bound $\beta_{\text{slot}}$. The analogous concentration result for uniformly-shuffled committees is established below as Lemma 26b; the numbers in the table use Lemma 26b's bound, which has the same Hoeffding form.

**Lemma 26b** (Per-committee Byzantine concentration for uniform-shuffled committees). *Let the slot attestation committee for slot $N$ be the union of beacon committees returned by `get_beacon_committee`, formed by uniform shuffling (via `compute_shuffled_index`) of the active validator indices and partitioning into per-slot buckets. Let $\beta'$ denote the Byzantine fraction by **validator count** (not effective balance) in the active set, and let $\beta_{\text{slot}}$ denote the Byzantine fraction in the slot's attestation committee. For any $\epsilon > 0$,*

$$
\Pr[\beta_{\text{slot}} > \beta' + \epsilon] \;\leq\; \exp(-2 k\, \epsilon^2),
$$

*and symmetrically for the lower tail.*

*Proof sketch.* The slot attestation committee is a uniformly random size-$k$ subset of the active validator set, partitioned without replacement. Byzantine indicators on the committee positions are a sample-without-replacement from a finite population with success rate $\beta'$. By Hoeffding's bound for sampling without replacement (Hoeffding 1963, §6), $\Pr[\beta_{\text{slot}} > \beta' + \epsilon] \leq \exp(-2k\epsilon^2)$; the without-replacement bound is at least as strong as the with-replacement (iid) bound. $\blacksquare$

*Reconciling with Lemma 26.* Lemma 26 (balance-weighted, with replacement) and Lemma 26b (uniform-shuffled, without replacement) deliver the same Hoeffding form for slightly different reasons. The key distinction is whose "Byzantine fraction" is being bounded: Lemma 26 bounds the **effective-balance** fraction (relevant for PTC voting weight); Lemma 26b bounds the **validator-count** fraction (relevant for attestation committees, since each attester contributes one count regardless of balance). When the document writes "per-committee β < 20%" without qualification, the binding constraint is whichever is tighter; under near-uniform effective-balance distribution (post-Electra cap at MAX_EFFECTIVE_BALANCE_ELECTRA), the two fractions are close to each other and both deliver the Hoeffding bound above.

| $\beta'$ (global) | Required deviation$\epsilon = 0.2 - \beta'$ | $\Pr[\beta_{\text{slot}} \geq 0.2]$     | Verdict                                                                                  |
| ------------------- | --------------------------------------------- | ----------------------------------------- | ---------------------------------------------------------------------------------------- |
| $0.10$            | $0.10$                                      | $\exp(-625) \approx 0$                  | Astronomical safety.                                                                     |
| $0.15$            | $0.05$                                      | $\exp(-156) \approx 10^{-68}$           | Astronomical safety.                                                                     |
| $0.18$            | $0.02$                                      | $\exp(-25) \approx 1.4 \times 10^{-11}$ | Safety probability ~ 1.                                                                  |
| $0.19$            | $0.01$                                      | $\exp(-6.25) \approx 0.0019$            | $\approx 99.8\%$ slot reliability — rare failures (~once per 530 slots ≈ 1.8 hours). |
| $0.195$           | $0.005$                                     | $\exp(-1.56) \approx 0.21$              | Problematic: ~21% of slots could fail payment safety.                                    |
| $0.20$            | $0$                                         | (boundary)                                | Failing half the time.                                                                   |

**Reading this table:** the payment-safety threshold is *more sensitive* to global $\beta'$ than the PTC threshold, but the *concentration is also tighter* (committee 61× larger). Even a 1pp margin ($\beta' = 0.19$) yields ~99.8% per-slot reliability; a 2pp margin ($\beta' = 0.18$) brings the failure probability to $\sim 10^{-11}$. The danger zone is within ~1pp of $0.20$, not 5–10pp as the PTC-scale numbers might naively suggest.

#### Comparison with Gasper's standard $\beta' < 1/3$ assumption

Gasper's classical safety analysis assumes $\beta' < 1/3$. Under $\beta' = 1/3$, instantiation B's lower-tail bound gives $\Pr[\beta_{\text{slot}} < 0.2] \leq \exp(-62{,}500 \cdot (2/15)^2) = \exp(-1{,}111) \approx 0$ — i.e., per-slot-committee $\beta < 0.2$ is essentially impossible. So **the ePBS payment-safety assumption is strictly stronger than Gasper's $\beta' < 1/3$**: it requires global $\beta'$ to be comfortably below $0.20$, not just below $1/3$.

For instantiation A (PTC majority), the $0.5$ threshold is satisfied with comfortable margin even at $\beta' = 1/3$: $\Pr[\beta_{\text{PTC}} \geq 0.5] \leq \exp(-1024 \cdot (1/6)^2) = \exp(-28.4) \approx 4\times 10^{-13}$. So PTC operation alone is fine under Gasper's bound.

#### Summary remark

The numeric threshold $0.20$ in the document's standing assumption is a **design choice** — calibrated to the spec parameters $\mathtt{PROPOSER\_SCORE\_BOOST} = 40$ and $\mathtt{BUILDER\_PAYMENT\_THRESHOLD\_NUMERATOR}/\mathtt{DENOMINATOR} = 6/10$, which together yield the relationship "payment quorum = reveal threshold + adversary budget" $= 40\% + 20\% = 60\%$ (Section 12.4 adversarial model). Lemma 26 is the **statistical tool** that lifts this design threshold into a deployment requirement on the global Byzantine fraction:

- For **PTC operation alone**, the required margin is generous: any $\beta' < 0.4$ gives near-perfect per-PTC reliability for the $0.5$ honest-majority threshold.
- For **payment safety** (the binding constraint), the required margin is small in pp but tight in concentration: $\beta'$ within ~1pp of $0.20$ degrades per-slot reliability below $99\%$, while $\beta'$ at $0.18$ or below brings failure probability to ${\sim}10^{-11}$.

The document writes "$\beta < 0.2$ per committee" uniformly because it is the binding requirement; the PTC and other mechanisms inherit this bound with comfortable slack but do not strictly need it.

#### Helper 31: notify_ptc_messages

```python
def notify_ptc_messages(store, state, payload_attestations):
    if state.slot == 0:
        return
    for payload_attestation in payload_attestations:
        indexed = get_indexed_payload_attestation(state, payload_attestation)
        for idx in indexed.attesting_indices:
            on_payload_attestation_message(
                store,
                PayloadAttestationMessage(
                    validator_index=idx,
                    data=payload_attestation.data,
                    signature=BLSSignature(),
                ),
                is_from_block=True,
            )
```

Called by `on_block` (Algorithm 7) at the end of block processing. It extracts individual `PayloadAttestationMessage` objects from the aggregated `PayloadAttestation` containers in the block body and feeds each one through `on_payload_attestation_message` (Algorithm 9) with `is_from_block=True`. The `is_from_block` flag tells Algorithm 9 to skip signature verification and the current-slot check (which would reject these attestations because they are from the previous slot). This is the path by which PTC votes from slot N − 1 enter the store when block N is processed.

*Spec reference: `notify_ptc_messages` in `fork-choice.md`.*

#### PTC vote interaction: gossip vs. block inclusion

PTC votes can reach the store by two paths: directly from gossip via `on_payload_attestation_message` (Algorithm 9) with `is_from_block=False`, or indirectly via `notify_ptc_messages` (Helper 31) with `is_from_block=True`. The next lemma establishes that these two paths produce the same store state — the path of arrival does not matter.

**Lemma 27** (PTC vote idempotence across gossip and block inclusion). *Let m be a `PayloadAttestationMessage` from a PTC member for a beacon block with root r. If m reaches the store via gossip (Algorithm 9, `is_from_block=False`) and is later re-delivered via block inclusion (Helper 31), or vice versa, the final state of `store.payload_timeliness_vote[r]` and `store.payload_data_availability_vote[r]` is the same as if only one delivery occurred.*

*Proof.* By Algorithm 9 (`on_payload_attestation_message`), the writes are:

```
for idx in ptc_indices:
    store.payload_timeliness_vote[r][idx] = msg.data.payload_present
    store.payload_data_availability_vote[r][idx] = msg.data.blob_data_available
```

where `ptc_indices = [i for i, v_idx in enumerate(ptc) if v_idx == msg.validator_index]` enumerates **every** position the validator occupies in `get_ptc(state, slot)`. Because the PTC is constructed by `compute_balance_weighted_selection` (Helper 30) — which samples with replacement — a single validator may occupy several positions; the spec docstring for `compute_balance_weighted_selection` says explicitly "the returned list can contain duplicates", and Algorithm 9 writes to all duplicate positions in a single delivery.

Two key observations:

1. **The position set is deterministic in `(validator_index, slot, state)`.** `get_ptc` is a pure function of the state's `ptc_window`, and `ptc_window` is fixed across a slot's lifetime. The set `ptc_indices` is therefore the same regardless of when the message is processed.
2. **The values written depend only on `msg.data`.** `msg.data.payload_present` and `msg.data.blob_data_available` are fields of the `PayloadAttestationData` object, which is the same object signed by the PTC member regardless of which path the message took.

Two delivery orderings:

- **Gossip first, then block inclusion.** The gossip-received message writes the same pair of values to every position in `ptc_indices`. The later block-included message (via Helper 31) re-writes the same values to the same set of positions. The store's contents are unchanged after the second pass.
- **Block inclusion first, then gossip.** Symmetric: the block-included write occurs first, then the gossip-received re-write touches the same positions with the same values. Again, unchanged.

The two paths differ only in the validation performed before the writes (signature check in gossip, skipped for block-included since the proposer aggregated and signed the bundle). Both paths reach the same write site with the same target set and the same values. ∎

*Remark.* This lemma matters because it lets the proofs treat "PTC member i voted X for block r" as a single event with a deterministic effect on the store, regardless of whether the vote arrived via gossip, via block inclusion, or both. The proofs of Lemmas 7, 13, 14, and 17 implicitly rely on this property when reasoning about `payload_timeliness_vote[r]` independently of message-delivery paths.

#### Empty-slot safety

A common scenario in ePBS analysis is a missed slot (no block proposed) followed by recovery. The next lemma collects the structural facts about missed slots into a single statement that subsequent proofs can cite.

**Lemma 28** (Empty-slot state preservation). *Let block B be at slot N, processed by all honest nodes (so `store.blocks[B.root]` and `store.block_states[B.root]` exist). Suppose slot N + 1 is missed: no block is proposed and no block referencing slot N + 1 enters the store. Then at the start of slot N + 2:*

- *(i) The fork-choice store is unchanged from the start of slot N + 1, except: `proposer_boost_root` was reset to `Root()` by `on_tick_per_slot` at the slot N + 1 → N + 2 boundary, and `store.time` was advanced to the new slot's wall-clock value (both routine slot-boundary updates).*
- *(ii) `state.latest_block_hash` and `state.latest_execution_payload_bid` retain their values from B's processing — no block at slot N + 1 means no `process_execution_payload_bid` for slot N + 1, so the cached bid is still B's.*
- *(iii) `state.builder_pending_payments[SLOTS_PER_EPOCH + (N+1) % SLOTS_PER_EPOCH]` is empty (a fresh `BuilderPendingPayment()`) — no slot N + 1 block means no IOU was created.*
- *(iv) PTC members for slot N + 1 do not vote (Section 12.2, PTC handler lines 2–3: "If the validator has not seen any beacon block for the assigned slot, do not submit a payload attestation"). Therefore `store.payload_timeliness_vote` and `store.payload_data_availability_vote` are not updated for any slot-N+1 root (no such root exists).*

*Proof.* Each part follows directly from the absence of an `on_block` call for any slot N + 1 block:

*(i)* The store is modified by `on_block`, `on_execution_payload_envelope`, `on_payload_attestation_message`, and `on_tick_per_slot`. With no slot N + 1 block, the first three handlers are not triggered for any slot N + 1 root. `on_tick_per_slot` runs at the slot boundary, advances `store.time`, and resets `proposer_boost_root = Root()` (base spec, inherited by Gloas). All other store fields are unchanged.

*(ii)* `state.latest_block_hash` is updated only by `apply_parent_execution_payload` (Helper 12b, line 29) inside `process_block` (Helper 11). With no slot N + 1 block, no `process_block` runs for slot N + 1, and these fields retain their values from B's processing. Similarly `state.latest_execution_payload_bid` is updated only by `process_execution_payload_bid` (Helper 8, line 27).

*(iii)* `state.builder_pending_payments` is written only by `process_execution_payload_bid` (Helper 8, line 26), `settle_builder_payment` (Helper 12a), `process_attestation` (Helper 15, lines 28–32), `process_proposer_slashing` (Helper 22), and the epoch-boundary shift in `process_builder_pending_payments` (Helper 14, lines 6–8). None of these ran for the slot-N+1 entry during slot N + 1: no block means no Helper 8 call, no settle (no FULL declaration possible), no attestation (no block to attest to), no slashing for that slot. The entry remains as initialized.

*(iv)* By the PTC handler in Section 12.2, counting from `def ptc_vote(...)` as line 1, lines 2–3: a PTC member checks `has_beacon_block_for_slot(store, slot)` and early-returns when False. With no slot N + 1 block, this returns False, and the handler returns without broadcasting. The vote stores indexed by slot N + 1 roots do not exist (since no slot N + 1 root entered the store at all). ∎

*Remark.* This lemma is the foundation for Lemma 8 (attestation fallback for missed slots) and for the missed-slot fallback path in Lemma 11's Case 2 sub-case (a) ("builder withholds entirely" combined with subsequent missed slots, so that no `process_attestation` ever runs against the slot-N IOU). It also clarifies a subtle point in the attack analysis: at the start of slot N + 1, when an honest builder runs `get_head` to construct its bid, `proposer_boost_root` has been reset by `on_tick_per_slot` regardless of whether slot N + 1 will be missed. The reset happens at the slot boundary, before any block could arrive.

### 12.6 Free Option Analysis

*Pedagogical notation.* In the definitions below, we write `slot_start(N)` as a shorthand for `compute_time_at_slot(state, N)` — the wall-clock start time of slot N, equivalently `state.genesis_time + N * SECONDS_PER_SLOT`. `slot_start` is not a Gloas spec function; we use it for narrative clarity in the timing definitions.

The slot timeline (Definition 1) places the bid commitment at the proposer's beacon block broadcast (t = 0) and the payload reveal deadline at the PTC vote time (t = T_ptc). The interval between these two events is observable by the builder, who has already committed to a bid value but has not yet decided whether to reveal the execution payload. During this interval, the builder may observe new market information (e.g., centralized-exchange price movements) and adjust its decision. This subsection states the formal definitions and lemmas that describe this strategic situation and the spec-level mechanisms that bound it.

**Definition 14** (Bid commitment time). The *bid commitment time* for a slot N bid is the time `t_commit = slot_start(N)` at which the proposer's beacon block — containing `signed_execution_payload_bid` — is broadcast. From `t_commit` onward, an honest node that processes the block records a `BuilderPendingPayment` for slot N via `process_execution_payload_bid` (Helper 8, lines 17–26). Before `t_commit`, the bid exists only as a `SignedExecutionPayloadBid` on the `execution_payload_bid` gossip topic and creates no on-chain obligation.

*Spec reference: `process_execution_payload_bid` in `beacon-chain.md`.*

**Definition 15** (Reveal deadline). The *reveal deadline* for a slot N payload is `t_reveal = slot_start(N) + T_ptc` (with `T_ptc = 9s` under `PAYLOAD_ATTESTATION_DUE_BPS = 7500`). A reveal observed by an honest PTC member at or before `t_reveal` is voted as `payload_present = True`; a reveal observed strictly after `t_reveal` does not influence the same-slot PTC tally because PTC members have already broadcast. After `t_reveal`, even if `on_execution_payload_envelope` (Algorithm 8) succeeds and populates `store.payloads[r]`, the FULL/EMPTY tiebreaker for slot N (Definition 13) is decided based on the votes already cast.

*Spec reference: `is_payload_timely` (Helper 4); PTC member broadcast time in `validator.md`.*

**Definition 16** (Option window). The *option window* for a slot N bid is the interval `[t_commit, t_reveal)` of length `T_ptc = 9s`. Within this window, the builder can decide to reveal (broadcasting a `SignedExecutionPayloadEnvelope`) or to withhold (broadcasting nothing). The decision is unilateral: no consensus message during the window can revoke the bid commitment recorded at `t_commit`.

**Definition 17** (Withhold strategy). A builder *withholds* if, despite having a `BuilderPendingPayment` recorded for its bid at `t_commit`, it does not broadcast a valid `SignedExecutionPayloadEnvelope` before `t_reveal`. By Algorithm 8, no `store.payloads[r]` entry exists at any honest node by `t_reveal`, hence no FULL node is created in the fork-choice tree (Definition 6).

---

**Lemma 29** (Cost of withhold under attested block). *Let block B at slot N have `bid.value > 0` and meet the Path B quorum (Helper 18) — that is, the accumulated `payment.weight` for slot N at the epoch boundary is at least `get_builder_payment_quorum_threshold(state)`. If the builder withholds, then by the end of the epoch following N the builder's `balance` is decreased by `bid.value` (modulo the `min(amount, balance)` clamp in `apply_withdrawals` if out-of-band balance reductions intervene; see Lemma 12c), and the builder receives no on-chain compensation in return.*

*Proof.* By Definition 17, no `SignedExecutionPayloadEnvelope` for slot N is broadcast before `t_reveal`. By Algorithm 8, `store.payloads[B.root]` is never populated at any honest node, so `is_payload_verified(store, B.root) = False`. By Algorithm 7 (line 5 of `on_block`), no honest proposer can produce a valid block declaring `parentStatus = FULL` for B: the assertion `is_payload_verified(store, parent_root)` fails. Therefore `process_parent_execution_payload` (Helper 12c) never invokes `settle_builder_payment` for slot N, and Path A does not fire (Lemma 9 inapplicable).

The `BuilderPendingPayment` created by `process_execution_payload_bid` for slot N persists through the epoch. By the lemma's hypothesis, `payment.weight ≥ quorum`. By Lemma 10, `process_builder_pending_payments` (Helper 14) at the epoch boundary appends `payment.withdrawal` (with amount `bid.value`) to `state.builder_pending_withdrawals`. By Helper 21 (`apply_withdrawals`, modified), a subsequent block's `process_withdrawals` decreases the builder's `balance` by `min(bid.value, builder_balance)` (Helper 21 line 7). Under the standing solvency invariant (Lemma 12b), the builder's balance at withdrawal time satisfies `balance ≥ bid.value`, so the debit equals `bid.value` exactly — modulo out-of-band balance reductions noted in Lemma 12c's caveat.

The builder receives no on-chain payment in return: no payload was revealed, so no execution-layer fees are credited to the builder via `apply_parent_execution_payload`; no transaction inclusion occurs; and no MEV is realized from the slot. The builder's net economic outcome from slot N is `−bid.value` plus any off-chain revenue the builder may have realized by withholding (e.g., closing a position on a centralized exchange). The protocol does not observe or compensate any off-chain revenue. ∎

*Remark.* Lemma 29 captures the formal "free option exercise cost": the protocol guarantees that if the network attested to the beacon block, the builder pays for the option, regardless of whether the option ends up valuable to exercise. The strike price of the option is therefore at least `bid.value`. The "free" in *free option* is a misnomer: the option is not gratis, but its premium is the bid value rather than a separate option fee.

---

**Lemma 30** (Option window upper bound — PTC tally is frozen). *The option window has length `T_ptc` (= 9 seconds with current spec parameters). The same-slot PTC tally for B (i.e., the second conjunct of `is_payload_timely`, `sum(payload_timeliness_vote[B.root] is True) > PAYLOAD_TIMELY_THRESHOLD`) is determined entirely by votes broadcast at or before `slot_start(N) + T_ptc`. A late payload (arriving at `t > slot_start(N) + T_ptc`) does not modify the tally.*

*Proof.* By the honest PTC member behavior (Section 12.2, PTC handler), at time `slot_start(N) + T_ptc` each honest PTC member assigned to slot N broadcasts a `PayloadAttestationMessage` with `payload_present = is_payload_verified(store, B.root)` evaluated at that time. By the standing network-model synchrony assumption (every honest node receives any honestly broadcast message within delay $\Delta < T_{\mathrm{att}}$), all honest PTC votes for slot N reach all honest nodes by `slot_start(N+1)`. The `payload_timeliness_vote[B.root]` array is written only by `on_payload_attestation_message` (Algorithm 9) applied to these votes. A late payload arriving at `t > slot_start(N) + T_ptc` is processed by `on_execution_payload_envelope` (Algorithm 8) and may populate `store.payloads[B.root]`, but it does **not** trigger any write to `payload_timeliness_vote[B.root]`. Therefore the second conjunct of `is_payload_timely` reads the same value regardless of when the payload arrives. ∎

*Remark on `is_payload_timely` flipping.* `is_payload_timely` (Helper 4) is the conjunction of two conjuncts: (a) `is_payload_verified(store, B.root)`, and (b) the PTC tally above threshold. Lemma 30 establishes that the tally (b) is frozen at $T_{\mathrm{ptc}}$. The verification gate (a), however, CAN flip from `False` to `True` when a late payload arrives. So `is_payload_timely` can flip from `False` to `True` precisely in the case where the tally was already $>$ threshold at $T_{\mathrm{ptc}}$ but a local observer hadn't yet validated the payload. In the typical "builder withholds, PTC votes False" scenario this case does not arise (the tally is False), and `is_payload_timely` stays False even after a late payload.

*Remark on the fork-choice outcome.* `should_extend_payload` (Algorithm 6) has two inputs a late payload can affect: the hard guard `is_payload_verified(store, B.root)` (line 2, can flip True via late payload) and `is_payload_timely` (just analysed). Even when the PTC tally was False at $T_{\mathrm{ptc}}$, a flipped hard guard can enable the proposer-based fallbacks (lines 7–9): if no slot-N+1 block has been proposed yet (fallback (a)) or if the slot-N+1 block declares FULL for B (fallback (c)), the tiebreaker can flip in B's favour. The PTC-primary-path option window therefore closes strictly at `T_ptc`, but the broader proposer-based recovery window remains open until the slot-N+1 proposer commits. Both are subject to the builder's strategic withholding decision — the formal "free option" exercise cost (Lemma 29) applies regardless.

---

**Spec-level mitigations.** The spec contains three mechanisms that, together, bound the free option without eliminating it:

1. **Binding bid commitment.** Helper 8 records the IOU at block-processing time. The builder cannot rescind the bid after `t_commit`.
2. **Path B settlement at epoch boundary.** Lemma 10 ensures the builder pays whenever the beacon block is widely attested, even if the builder withholds. This converts withhold into a positive cost (Lemma 29).
3. **Bounded option window.** Lemma 30 caps the duration during which the builder can observe market information without consequence at `T_ptc = 9s`. Tightening the window (e.g., reducing `PAYLOAD_ATTESTATION_DUE_BPS`) would shorten the option but risks PTC members voting on an incomplete view of execution payload arrival; the spec balances these concerns at 75% of the slot.

**What the spec does not encode.** A fourth mitigation discussed in the literature — penalizing the builder beyond `bid.value` for withholding — is *not* in the spec. The maximum protocol-level penalty for a withhold is `bid.value`. Any additional disincentive must come from off-chain reputation or out-of-band slashing of the builder's centralized-exchange counterparty positions. Empirical analysis (Mazorra et al., 2025) suggests that under typical conditions the option is rarely exercised (~0.82% of blocks), with a peak of ~6% during high-volatility periods; this empirical baseline serves as the operational benchmark for whether further mitigations are needed.

*Remark.* The free option is a structural consequence of (a) requiring an explicit reveal step (which is what enables on-chain bid commitment without trusting a relay) and (b) placing the reveal after the attestation deadline (which is what enables attesters to vote without waiting for the execution payload, preserving the unconditional payment property). Eliminating the option entirely would require removing one of these design properties. The spec accepts the option as a trade-off and bounds its abuse via the three mechanisms above.
