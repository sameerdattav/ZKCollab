# PoA: ZK-Proven Aggregation Architecture (FL)

## Goal
Prove that the aggregator correctly computes the global model update:

$$
\Delta W_{\text{global}} = \sum_i \alpha_i \cdot \Delta W_i
$$

without revealing individual client updates \(\Delta W_i\), and optionally without revealing coefficients \(\alpha_i\).

## Scope
- Single-layer proof of aggregation for one federated round.
- Round-consistent inclusion of client updates tied to one base model commitment.
- ZK enforcement of weighted aggregation arithmetic and commitment consistency.
- Accepted-set linkage via round root, with a direct path to in-circuit membership verification.

## Roles
- Prover (Aggregator / FL server)
- Verifier (clients, smart contract, or downstream consumers)

### Prover Responsibilities
- Collect client updates and round metadata.
- Compute global aggregation.
- Generate zk proof of aggregation correctness.

### Verifier Responsibilities
- Verify proof against public inputs.
- Accept global update only if proof is valid.

## Round and Epoch Clarification
Round and epoch are related but not identical in this design.

- Round is the federated coordination unit.
- Epoch is a local training pass over a client's dataset.
- PoA generation cadence is one proof per finalized round, not one proof per epoch.

### Rules
1. One round must bind to exactly one base model commitment.
2. All updates included in a round PoA must reference that same base model commitment.
3. A client may run one or more local epochs before submitting its round update.
4. The protocol may allow one accepted submission per client per round (recommended default).
5. Stale submissions (trained on an older base model) cannot be included in that round's PoA.

### Practical examples
1. Example A: 1 round, 3 local epochs per client.
- Round 25 opens on model `M25`.
- Each client trains locally for 3 epochs.
- Clients submit one update each.
- Aggregator freezes accepted set and generates one PoA for Round 25.

2. Example B: 3 rounds, 1 local epoch each.
- Round 30 uses model `M30`; each client trains 1 epoch and submits.
- PoA is generated for Round 30, producing next model commitment `M31`.
- Process repeats for Rounds 31 and 32.

Both examples are valid. The difference is local compute budget per round, not proof semantics.

## Data Binding - Current Situation and Required Changes - FOR OJAS AND ABHISHEK.
Based on the  PR #9 data-binding changes, binding is currently dataset-root based and client-address based, not explicitly keyed by round or epoch.

Observed behavior in PR artifacts:
1. Registry API shape is root-centric:
- `commitRoot(bytes32 root)`
- `isCommitted(address client, bytes32 root)`
- `getRoots(address client)`
2. Data-binding witness fields are root/data/path centric (`merkle_root`, `raw_data`, `merkle_path`) with no `round_id` or `epoch_id` field.
3. Result: the same committed dataset root can be referenced across multiple rounds unless round separation is added by higher-level policy.

Implication for this PoA document:
- The round model written above is still valid as protocol design.
- However, current fetched PR data binding alone does not enforce round scoping cryptographically.

Required changes for strict round-consistent data binding:
1. Domain-separate the binding statement with round context.
- Bind against `H(round_id || base_model_commitment || dataset_root)` (or an equivalent canonical encoding).
2. Extend witness and proof statement inputs.
- Add `round_id` and `base_model_commitment` to data-binding witness/public inputs.
3. Extend contract registry interface.
- Move from root-only commit checks to round-scoped checks.
- Example interface direction:
	- `commitRoundRoot(round_id, base_model_commitment, dataset_root)`
	- `isCommittedForRound(client, round_id, base_model_commitment, dataset_root)`
4. Keep epoch as optional metadata, not the primary binding key.
- Epoch count can be policy metadata in PoT.
- Round identity should remain the settlement boundary used by PoA.

If you keep the current PR implementation unchanged, then the specification should treat data binding as submission-level dataset integrity, with round consistency enforced by off-circuit policy checks.

## Round Model (Sync vs Async) - UP FOR DISCUSSION
PoA uses asynchronous submission and synchronous settlement.

This means clients do not need to finish training at the same wall-clock second, but the proof is built over one deterministic frozen set for a round.

### Why this hybrid model
- Pure synchronous collection (everyone must arrive before proving) is simple but punishes stragglers and stalls rounds.
- Pure asynchronous aggregation (continuous updates, no freeze point) improves liveness but makes proof semantics ambiguous because the included set can change during proving.
- Hybrid mode keeps collection practical and proof semantics crisp.

### Protocol behavior
1. Round opens with fixed metadata: `round_id`, `base_model_commitment`, policy parameters, and cutoff or quorum rule.
2. Clients submit updates independently during the open window.
3. Aggregator validates policy and base-model match for each submission.
4. At cutoff (or when quorum is reached), aggregator freezes the accepted set.
5. Aggregator computes `\Delta W_global` from only the frozen set and generates one PoA.
6. Verifiers check proof against the same round metadata and accepted-set root.
7. Late or stale submissions are rolled into the next round (or rejected by policy).

### Concrete timeline example
Assume Round 12 has:
- `base_model_commitment = M12`
- window 12:00 to 12:10
- accepted-set root published as part of the proof package

Event flow:
1. Client A submits at 12:02 with update tied to `M12`.
2. Client B submits at 12:08 with update tied to `M12`.
3. Client C submits at 12:11 (late).
4. At 12:10, accepted set is frozen as `S12 = {A, B}`.
5. PoA proves:

$$
\Delta W_{global}^{(12)} = \alpha_A \Delta W_A + \alpha_B \Delta W_B
$$

6. Client C is not part of Round 12 proof; C is queued for Round 13 after policy checks.

### Stale update example
If Client D submits during Round 12 but trained on `M11` instead of `M12`, the update is stale for Round 12.
- It must be rejected or re-based by policy before inclusion.
- This prevents mixing updates from different model states in one proof statement.

### Verification implication
The verifier does not check wall-clock simultaneity. The verifier checks that:
- all included commitments belong to the frozen round set,
- aggregation arithmetic is correct for that set,
- the published global commitment matches computed result.

That is the key distinction: asynchronous participation, synchronous proof boundary.

## Data Model

### Public Inputs
- `round_id`
- `C_global = Commit(\Delta W_global)`
- `C_updates_root` (root over included client update commitments)
- `C_coeffs` (commitment to coefficients, or coefficients exposed if policy allows)
- `C_accepted_root` (root of accepted client set for the round)

### Private Witness (Prover only)
- `\Delta W_i` for each included client
- `\alpha_i` for each included client
- Optional inclusion proofs into accepted set

## Proof Statement
The prover proves all of the following:

1. Commitment consistency for each client update:

$$
\text{Commit}(\Delta W_i) = C_i \quad \forall i
$$

2. Correct weighted aggregation:

$$
\Delta W_{\text{global}} = \sum_i \alpha_i \cdot \Delta W_i
$$

3. Global commitment correctness:

$$
\text{Commit}(\Delta W_{\text{global}}) = C_{\text{global}}
$$

4. Accepted set membership:

$$
C_i \in C_{\text{accepted\_root}} \quad \forall i
$$

## Circuit Design

### 1) Commitment Re-computation
- Recompute `C_i = Commit(\Delta W_i)` in-circuit.
- Enforce equality with the committed client value.
- Prevents forged or post-hoc modified inputs.

### 2) Weighted Aggregation Constraints
For each tensor element `j`:

$$
(\Delta W_{\text{global}})_j = \sum_i \alpha_i \cdot (\Delta W_i)_j
$$

Repo mapping:
- Scalar multiplication primitive: `src/basic_block/mul.rs`
- Summation chain primitive: `src/basic_block/add.rs`

### 3) Global Commitment Check
- Recompute `C_global' = Commit(\Delta W_computed)`.
- Enforce `C_global' == C_global`.
- Binds arithmetic result to public output.

### 4) Accepted Set Membership (Two-Phase)
- Phase A (current): accepted clients filtered off-circuit; circuit proves arithmetic + commitment consistency.
- Phase B (future): enforce membership proofs in-circuit (for example Merkle path verification to `C_accepted_root`).

## Execution Flow

### Step 1: Client
- Compute local update `\Delta W_i`.
- Submit update or commitment plus required round metadata.

### Step 2: Aggregator (Prover)
- Freeze accepted client set for round.
- Compute global update with protocol coefficients.
- Generate PoA proof.
- Publish proof package: `round_id`, `C_global`, roots/metadata, proof bytes.

### Step 3: Verifier
- Verify proof against public inputs.
- Accept new global model only if proof verifies.

## Security Guarantees
- Correctness: aggregator cannot claim incorrect aggregation without failing proof.
- Integrity: committed updates are binding and cannot be swapped undetected.
- Privacy: client updates remain private; only commitments and proof are public.
- Verifiability: third parties can verify succinctly.

## Failure Cases
- Modified client update: fails commitment consistency.
- Modified coefficient: fails weighted aggregation constraint.
- Fake global update: fails global commitment check.
- Invalid client inclusion: fails membership check (Phase B in-circuit model).

## Implementation Mapping to Current Repo
- Proving flow and transcript style: `src/training/mod.rs`
- Arithmetic building blocks: `src/basic_block/mul.rs`, `src/basic_block/add.rs`
- Constraint extension point: `src/training/constraints.rs`
- Existing roadmap anchors: `docs/Plan.md`, `docs/Architecture.md`, `docs/Arjun_architecture.md`

## Acceptance Criteria
- Given a valid frozen set, proof verifies and binds to published `C_global`.
- Tampered update or coefficient causes proof failure.
- Round mismatch or stale base-model reference is rejected by policy and/or constraints.
- Deterministic reproducibility of proof inputs for the same frozen set.
