# Design Decisions Needed

Running to-do list of open design decisions for the Composable Adjudication ERCs. The ERC drafts stay clean and self-contained; anything still undecided lives here, with enough description to pick it up cold. When a decision is made: apply it to the drafts, record it in `AGENTS.md`, and delete it from this file.

## Core ERC

### 1. EAS schema contents
The three schemas (**adjudication definition**, **evidence**, **resolution**) need concrete field lists registered in the EAS schema registry. **Priority raised (2026-07-30):** definitions are now mandatory on-chain and Adjudicators enforce activation authorization by decoding them, so at minimum the party-list/authorization field encoding must be normative. Until it is, enforcement is per-implementation decoding, and a definition authored for one Adjudicator cannot be handed to another, which defeats swappability at the semantic layer while the function signatures match perfectly.

**Which schemas need normative content.** The test is which of them a *contract* decodes on-chain:

| Schema | On-chain reader | Normative layout needed |
|---|---|---|
| Definition | The Adjudicator, to enforce activation authorization (decision 15) | Yes |
| Resolution | Composites, to adopt a child's outcome or compare children's outcomes | Yes |
| Evidence | None; the normative case link is the `EvidenceLinked` event, and adjudication systems read content off-chain | No; RECOMMENDED suffices |

A generic X-of-Y panel cannot decide whether children *agree*, and an escalation router cannot adopt a child's resolution, unless outcomes decode uniformly across heterogeneous leaves. This is §5 below, but the enabling change belongs in the core schema, not in ERC #2.

**Candidate schemas (proposal for circulation, not settled).** All registered with `revocable = false` and no resolver, per decision 16; attestations carry `expirationTime` zero. EAS schema strings are a flat comma-separated list with no comment syntax, so the annotated blocks below are documentation only: the exact one-line string under each block is what gets registered, and its field order and spacing are load-bearing because the schema UID is a hash over that string.

**Adjudication definition**

```
{
  // ─── Enforced on-chain by the Adjudicator ───
  address[] parties,           // may activate, each unilaterally; empty = none named
  bool      openActivation,    // definition opts into activation by anyone

  // ─── Not read by the Adjudicator, but still part of the canonical schema ───
  bytes32   resolutionSchema,  // EAS schema UID the resolution must use
  string    question,          // what is being decided
  string[]  outcomes,          // the outcome space, index-addressable
  bytes32   baselineHash,      // content hash of the parties' agreement and its context
  bytes     processParams      // implementation-defined, opaque
}
```
`address[] parties,bool openActivation,bytes32 resolutionSchema,string question,string[] outcomes,bytes32 baselineHash,bytes processParams`

The head is exactly decision 15's precedence rule and nothing more: parties named → only they activate; else `openActivation` → anyone; else registrant only. Activation arrives as a transaction from an address, so `address[]` is the only enforceable form: a definition whose parties are off-chain identities leaves the array empty and correctly falls through to registrant-only. `outcomes` on-chain lets composites and UIs interpret a resolution without an off-chain fetch.

**Resolution**

```
{
  // ─── Read on-chain by the X-of-Y Adjudicator composites ───
  bytes32 outcome,        // comparable across leaves; its meaning is set by the definition

  // ─── Not read on-chain ───
  bytes32 rationaleHash,  // content hash of the justification, if any
  bytes   systemData      // the underlying system's native result, opaque
}
```
`bytes32 outcome,bytes32 rationaleHash,bytes systemData`

The ERC constrains only that two outcomes are comparable, never what one means (decision 9 intact). The tail deliberately carries **no** definition reference: `resolution()` is itself the provenance.

**Evidence.** RECOMMENDED in full; no field is decoded on-chain, so nothing here is normative.

```
{
  bytes32 contentHash,  // authoritative
  string  contentURI,   // retrieval hint, non-authoritative
  string  kind          // type tag for renderers
}
```
`bytes32 contentHash,string contentURI,string kind`

No definition field; the draft's `refUID` traceability recommendation covers that.

**How the layout is made binding: resolved by research (2026-07-30), see `knowledgebase.md`.** Of the two candidates, the second is rejected on security grounds.

1. **Canonical schema UID, whole-tuple decode (recommended).** The ERC publishes byte-exact schema strings; anyone registers them permissionlessly. An Adjudicator gates on `attestation.schema == CANONICAL_DEFINITION_SCHEMA_UID`, a plain `bytes32` comparison with no registry lookup, and then decodes the full tuple. **Verified:** the schema UID is `keccak256(abi.encodePacked(schemaString, resolver, revocable))` with no chain id, sender, nonce, or registry address in the preimage, so the same string with `resolver = address(0)` and `revocable = false` yields the *same UID on every chain* and the ERC can name it as a chain-independent constant. (A resolver would break this unless deployed at an identical address per chain.)
2. **Layout-prefix compatibility: REJECTED.** Mandating only that the first two fields are `parties` and `openActivation`, letting extenders append, is unsafe over attestation data that anyone may author. **Verified empirically:** with a leading dynamic `address[]`, a payload too short to contain the expected fields does *not* reliably revert, because the decoder silently aliases the array's length word as the missing static field; and because ABI head slots hold attacker-controlled offsets, a crafted payload can make a two-field prefix decode return *different* `parties`/`openActivation` values than a full decode of the same bytes. A contract enforcing authorization on the prefix and a UI displaying the full decode would then disagree about who may activate, precisely the failure mode on-chain definitions exist to prevent. Solidity's tolerance of trailing data is also documented as current behavior, not a guarantee.

**Consequence: the head/tail split collapses.** With one canonical schema decoded whole, every field is normative in practice: the annotations above mark which fields the *Adjudicator* reads, not which are optional. Extension goes through the opaque `bytes processParams` field, never through a varied schema. If a use case genuinely needs a different definition schema, it is a different schema with a different UID and no claim to portability.

**Open sub-questions.** Whether `outcome` reserves a zero value for "undecided / refused to arbitrate" (useful to composites, but it is a semantic constraint and decision 9 pushes against it); whether the baseline gets a `string baselineURI` retrieval hint alongside the authoritative hash (decision 16 warns against mutable references, so a hint would have to be explicitly non-authoritative, or the hash can simply be a CID); whether `question`/`outcomes` belong on-chain at all or behind the baseline hash, trading UI convenience against decision 16's compactness SHOULD; and whether to attach a schema **resolver** that validates `data` at attest time (EAS does not validate `data` against the schema string, so a malformed definition is registrable, but a resolver costs the chain-independent UID above, and a malformed definition arguably just binds nobody).

**Drafting requirement.** Schema strings are hashed verbatim: whitespace is significant, field names are optional, and neither the EAS SDK nor `easctl` normalizes anything. The ERC must publish the canonical strings byte-exact, and should state that registration is not idempotent (a second `register` of the same triple reverts `AlreadyExists`, which is expected once someone has registered it on a given chain).

**Expected objection.** Mandating schema layouts is the over-specification the draft's Rationale blames for ERC-1497's fate. The distinguishing argument, which belongs in the Rationale if this lands: 1497 standardized off-chain JSON conventions with no enforcement point, so implementations drifted at zero cost; a head that a contract decodes has an enforcement point, since a non-conforming definition reverts.

**Not an open question:** who may attest a definition. An attacker can attest a definition naming themselves sole party, but it binds nobody, by the same orphan-registration reasoning that makes a resolution-attester binding unnecessary.

### 2. Multi-chain attestation references and hosting chain
Mostly settled (2026-07-30): **definition and resolution attestations MUST live on the Adjudicator's own chain.** Definitions for on-chain readability and enforcement; resolutions as a necessary consequence of the Adjudicator's verification duty (it can only verify attestation properties on its own chain). Remaining: hosting and referencing conventions for evidence (read by adjudication systems off-chain, so flexibility may be acceptable), and the general referencing convention where cross-chain pointers are unavoidable. **Sharpened by research (2026-07-30):** an attestation UID's preimage contains no chain id and no EAS contract address, so a bare 32-byte UID does not merely fail to *name* a chain, it is not globally unique at all, and the same UID can exist on two chains pointing at unrelated attestations. Any cross-chain convention must therefore carry a chain identifier alongside the UID, not assume the UID disambiguates itself.

### 3. ERC-165 interface IDs
Mechanical: compute and freeze the interface IDs once the function signatures stop moving.

### 4. Token fee channel
`payable` channels only the native asset, yet several adjudication systems price fees in ERC-20 tokens (stable across the potentially long registration→activation gap, where a fee fixed in native value drifts). An adapter fronting a token-priced system has no standard way to receive its fee: `data` is barred from carrying semantic content, and ad-hoc approve-then-pull conventions would diverge per adapter, recreating the fragmentation this ERC exists to remove. Sharpest in composites, where an escalation router must fund children whose fees may be denominated in different assets. Decide: standardize a token payment path (e.g., a normatively blessed approve-and-pull pattern in which the Adjudicator announces its fee token) or keep the channel native-only and push token handling into adapters.

**Sub-point raised 2026-07-31 (Kleros side), for circulation: cost shifting, and why it may be the same decision as `cost()`.** Loser-pays is the classic discipline mechanism and it appears nowhere in the standard: fee semantics are implementation-defined, so nothing shifts costs onto the losing side, and the ERC #2 claim that later rungs discipline earlier ones currently rests on it. The Kleros pattern is that both parties pay, the winner is refunded, and the loser's deposit compensates the winner in part or in whole depending on whether that deposit exceeds the adjudication fee. UMA is two-sided as well (proposer and disputer bonds, the loser's bond paying the winner), so two-sided bonding is not a Kleros idiosyncrasy; but the shapes diverge enough (who bonds, how much, whether the loser's deposit pays the winner or the protocol, what happens when it falls short of the fee) that standardizing across them is the false precision decision 11 already rejects. **Standardizing loser-pays is therefore not what is proposed here.** Three findings and one question:

1. **The adversarial-timing objection dissolves.** Both parties cannot be made to pay in the activation transaction, since one is adverse by construction. They do not have to: non-payment becomes the losing move, as in Kleros escrow, where activation opens a window and a counterparty who does not match the fee loses by default. This needs no change to the core. The window lives inside `Active`, decision 17 is satisfied because every transition still occurs, and decision 22 already discourages the atomic activate-and-resolve that would foreclose it. Two-sided funding is expressible today, per Adjudicator; what is missing is portability across them, not permission.

2. **Bonds at registration are the wrong instrument.** They would undo decision 4's rationale (registration puts the commitment on the books at near-zero cost while fees wait for a contest actually to exist), lock capital against contests that mostly never happen, and go stale as the fee drifts across a long dormancy, requiring a top-up path in any case. Commit-at-registration and collect-at-activation through a permit or allowance does not rescue the idea: an allowance is revocable, so the adverse party revokes before activation. Locking is what makes a bond a bond.

3. **The Adjudicable is a more natural home for cost shifting than the Adjudicator.** The value at stake already sits there (escrow, lending position, marketplace balance), and the Adjudicable already reads `resolution()` and controls that stake, so it can reallocate the fee out of collateral it holds. This also answers what pure reimbursement cannot: "the loser pays afterwards" is an unsecured claim against someone who may hold nothing, while "the loser pays out of locked collateral" is not. Two-sided bonds exist largely because most adjudicators have no claim on the parties' assets, whereas an Adjudicable does. If this holds, cost shifting stays outside the core by design, and the ERC #2 escalation-economics text should say so rather than leave the discipline claim unsupported.

**Question for co-authors, and the reason this sits under §4.** The one piece that does look like a core gap is `cost()`, currently deferred to a future extension on the grounds that callers learn costs from the definition or off-chain. A composite cannot: no generic router can fund a child without asking the downstream Adjudicator what activation requires, and every funding pattern in the ERC #2 outline needs exactly that. `cost()` and the token fee channel above are plausibly one decision rather than two, because a cost view returning a bare `uint256` is already wrong once fees can be token-denominated; it would have to name an asset alongside an amount. Settling them in sequence risks fixing the same signature twice, and interface IDs cannot freeze until both land. Should `cost()` be pulled out of future extensions and decided together with the fee channel?

## Composition & Factories ERC

### 5. X-of-Y agreement semantics
What "X children agree" means over EAS-defined resolutions. Likely equality of a designated outcome field in the resolution schema, but the aggregation rule and its reference (instantiation config vs. definition) need specification. Design direction already settled (2026-07-30): window-based exclusion of unresolved children (pool shrinks, threshold never lowers) and a no-quorum outcome in the definition's outcome space when the threshold becomes unreachable; non-conforming child resolutions and operator independence are implementation-defined; detailed semantics come with the implementation. The enabling piece lives in §1: without a normative `outcome` head in the resolution schema, no generic composite can compare outcomes across heterogeneous leaves, so §1 must settle first.

### 6. Escalation funding patterns (parked as future work)
Pay-as-you-go vs. pre-funded budget. Already moved to the outline's Future Work section; listed here only so the parked status is visible in one place.

---

Resolved and removed (recorded in `AGENTS.md`): `registerAdjudication` parameter shape (settled as-is); register-and-activate single-transaction convenience (removed from the spec as unneeded); zero `definitionUID` (allowed: Adjudicator-supplied definition; non-templates revert); escalation window source (per-composite design choice: instantiation configuration or on-chain definition, both enforceable).
