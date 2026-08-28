# Private Data Extensions — design rationale

Status: proposal, decisions settled 2026-08-28
Normative text: `SPEC.md` §6.1
Related: `wallet-frontend#183` (WSCA migration / private data split)

This document records *why* `S.extensions` is shaped the way it is. The
normative rules are in `SPEC.md`; everything here is the reasoning and the
evidence behind them, including two corrections to claims made elsewhere.

---

## 1. Why this exists

Three kinds of native-SDK state need somewhere to live in the private data
blob:

- WSCD key metadata — how to address a key created on a roaming
  authenticator.
- OID4VCI renewal state — the `refresh_token` and the DPoP key it is bound
  to.
- Blind BBS holder state — the blinding factor, without which a BBS
  credential can never be presented again.

Each was heading for the same treatment: its own top-level field under `S`,
its own specification section, its own `wallet-frontend` reducer, and its
own note explaining that it is not yet normative and may be lost. That
pattern costs a cross-repo change per addition and had already accumulated a
standing data-loss warning. `wallet-frontend#183` §12.3 names resolving it
as the one remaining Phase 0 item.

This replaces the pattern rather than extending it a third time.

---

## 2. What was verified

Both claims below were checked by running `wallet-frontend`'s own schema
modules under vitest against commit `6cb68a35`, not by reading them. The
probe was removed afterwards.

### 2.1 Unknown `S` fields survive a fold — the existing documents are wrong

`wallet-frontend#183` §12.2 and the uncommitted draft of a
`privatedata-spec` §6.1 both state that `wallet-frontend`'s typed reducers
*silently drop* `S.wscdCredentials` on the next write.

On the normal path they do not. `foldState` starts from `container.S` — the
loaded base state, never a fresh one — and `walletStateReducer` spreads
`{...state}`, so unrecognised keys are carried through:

```
S.wscdCredentials = { fido2: "OPAQUE-STATE" }
foldState(container)          // with a known alter_settings event
// → {"fido2":"OPAQUE-STATE"}    survives
```

The `initialWalletStateContainer()` call that would discard them runs only
when migrating a V1 keystore, which predates the container entirely.

### 2.2 Unknown event types crash the merge

An event type `wallet-frontend` does not know does **not** pass through
harmlessly. `mergeDivergentHistoriesWithStrategies` buckets events with
`eventsByType[event.type][0].push(event)` against a literal map of the nine
known types. For anything else that lookup is `undefined`:

```
mergeEventHistories(historyWithUnknownEvent, otherHistory)
// → THREW: Cannot read properties of undefined (reading '1')
```

The subsequent `for (const type in mergeStrategies)` iterates only known
types, so even without the throw the events would be dropped from the merged
history. This fires on a `412` conflict — two devices, one account.

### 2.3 The real loss path is merge, not fold

`mergeEventHistories` sets `S: baseState` — the common ancestor's state —
and replays events on top. So **`S` is a fold cache and `events` are the
source of truth.** Anything written into `S` without a corresponding event
is not reconstructible after a merge.

That single sentence is the whole design constraint. It also explains why
the current fields look correct in testing and lose data in the field: the
fold path preserves them, and only a two-device conflict exposes the
problem.

### 2.4 Behaviour today

| Carrier | Load → fold → save | 412 conflict merge | Verdict |
|---|---|---|---|
| Unknown field on `S` | preserved | discarded | Silent loss, only on conflict |
| Unknown event type in `events[]` | preserved | throws | Hard failure, blocks sync |
| Known event type | preserved | merged | The only durable carrier |

Neither extension route works end to end. The one that does — a known event
type with a merge strategy — is what `S.extensions` generalises.

### 2.5 A note on §6.1 and §6.2

At the time of writing, `origin/main` of this repository contained no
`wscdCredentials` text at all. The sections that `wallet-frontend#183` and
several native-SDK source comments cite as "privatedata-spec §6.1 / §6.2"
existed only as uncommitted local edits and were never published.

`SPEC.md` §6.2 now documents both legacy fields as deprecated, so that a
reader encountering them in a real container knows what they are, without
implying they were ever the recommended shape.

---

## 3. The design

One event type, one state field, one reducer, one merge strategy —
namespaces underneath.

```json
"extensions": {
  "org.siros.wscd": { "kid-a1b2": "<opaque>", "kid-c3d4": "<opaque>" },
  "org.siros.bbs":  { "1849302113": "<opaque>" }
}
```

```json
{
  "type": "set_extension",
  "namespace": "org.siros.bbs",
  "key": "1849302113",
  "value": "<opaque string>"
}
```

The reducer is total and generic: assign, or delete on a `null` value.
`wallet-frontend` never needs to know what a namespace means. The merge
strategy is last-write-wins on `(namespace, key)` ordered by
`(timestampSeconds, eventId)` — the same shape as the existing
`new_presentation` dedupe, so it slots into the current machinery rather
than sitting beside it.

### 3.0 Two merge modes, after review

The first draft made last-write-wins the *only* option. Review
([PR #1](https://github.com/sirosfoundation/privatedata-spec/pull/1))
pointed out that this permanently prevents resolution finer than whole-entry
replacement — two devices each adding a distinct item to a collection cannot
both survive — and that flattening extensions into a key-value store sits
oddly in a design whose whole argument is that `events` carry more than a
snapshot. That criticism is correct.

`SPEC.md` §6.1.3 now registers a **merge mode** per namespace:

- **`lww`** — the generic strategy. Any client can merge *and fold* it
  without understanding the values, because correctness depends only on the
  key and the ordering.
- **`events`** — the namespace defines its own event types and reduction.
  Clients without support MUST retain those events, MUST ignore them when
  folding, and MUST NOT fold them.

The split is not a compromise between the two designs; it is a recognition
that they answer different questions. For entity-snapshot state — the state
of one credential, one key, one batch — last-write-wins is *correct*, not a
weaker approximation. For accumulating state it is simply wrong.

The cost of `events` mode is stated plainly in the spec: those events
accumulate in `events` for any client that never gains support and cannot be
relieved by it. §6.1.4's growth rule bounds `S`; nothing bounds `events` for
that mode. This is the trade the more expressive model asks for, and it is
why `lww` remains the default.

### 3.1 The decision that matters: keys name entities

`S.wscdCredentials.fido2` is one opaque blob for a whole plugin, and
last-write-wins on *that* is data loss: two devices each enrolling a
different authenticator produce two blobs, and one silently wins.

Keyed per `kid`, the two devices write disjoint keys and merge cleanly with
no special strategy. The same holds for BBS state keyed per credential and
refresh tokens keyed per `batchId`.

**Choosing keys so that last-write-wins is *correct* is what lets a single
generic strategy serve every future namespace.** This is why `SPEC.md`
§6.1.2 makes it a MUST rather than a recommendation.

---

## 4. Settled decisions

### 4.1 The registry belongs to this specification

A table in `SPEC.md` §6.1.6. Registration is a spec change: enough friction
that names get considered, little enough that it never blocks a release.

### 4.2 Bound the shape, because you cannot bound the size

A per-namespace byte budget was the original proposal and was wrong. It is
often impossible to say in advance how large a namespace becomes — it grows
with enrolled authenticators, issued credentials, renewable batches. Any
number picked up front is either uselessly generous or a ceiling someone
hits in production.

So §6.1.3 constrains *growth* instead: extension state MUST be proportional
to entities the user can enumerate and delete, and MUST NOT grow with event
count — in particular, no append-only history inside an entry value.
Last-write-wins already gives this, since only the newest value per key is
retained; the rule exists to stop it being defeated by packing a log into
the value.

That is checkable, unlike a byte budget. What remains is *accounting*:
per-namespace sizes surfaced at runtime, so a container approaching the
transport limit identifies which namespace is responsible.

### 4.2a Folding `lww` is order-independent

Review raised a determinism problem: under ignore-and-retain, a client
supporting only version 1 folds v1 events past the horizon while leaving v2
events unfolded, so the v1 events resolve first — and the folded outcome
depends on which client folded.

For `lww` namespaces this does not arise. The result for a key is the value
of the greatest `(timestampSeconds, eventId)` among its events, so folding a
prefix and later applying the remainder reaches the same state as folding
everything at once. Partial support across a fleet does not make the folded
state depend on who folded it.

It does arise for `events` mode, where the reduction belongs to the
namespace — which is why §6.1.3 requires such a namespace to specify its own
ordering, and why §6.1.7 makes each version an independent namespace so the
interaction cannot occur across versions.

Review also exposed a genuine gap: the first draft ordered on
`timestampSeconds` alone, which is not deterministic on equal timestamps.
The `eventId` tiebreak is now REQUIRED.

### 4.2b What a version actually is

"A version is a new namespace" removes the cross-version fold interaction,
but taken literally it leaves nothing distinguishing `org.siros.bbs/v2` from
an unrelated namespace — while §6.1.7 originally kept migration rules that
only mean something if the two *are* related. That was inconsistent.

Working it through against real namespaces resolves it. A v2 of
`org.siros.bbs` would exist because what a BBS credential carries changed —
a pseudonym slot, say. At that point v2 state cannot be synthesised for a v1
credential: the blinding factor and committed messages come from an issuance
handshake that cannot be replayed. The credential would be re-issued. The
same holds for `org.siros.wscd`: new metadata fields cannot be invented for
a key already on an authenticator; it would be re-enrolled.

So migration is not under-specified, it is usually **impossible**, and the
real model is coexistence and drain:

- an entity's state lives under whichever version was current when the
  entity was created;
- new entities go to the newest supported version;
- old versions drain as their entities are deleted, which §6.1.4 already
  requires;
- nothing migrates, and no client deletes another version's state.

A version therefore differs from an unrelated namespace only by a lifecycle
expectation — same class of entity, predecessor is legacy-but-live — and not
by any format mechanism. That is also why no client support index is needed:
it answers "when can superseded state be retired", and nothing retires
superseded state.

### 4.3 A stale peer is a question for the user, not a merge

Tombstones must outlive the fold horizon, or a merge with a long-absent
device resurrects deleted entries. But retention alone does not solve the
general case: once a peer has been away longer than the horizon, its merge
base has been folded into `S` and is gone, so no correct merge exists for
*any* state — extensions included.

The resolution (§6.1.5) is a maximum time between folds. When a returning
wallet exceeds it, the histories are not silently reconciled; the user is
asked which wallet is authoritative.

This lands on a real code path. The horizon is already configurable at
`FOLD_EVENT_HISTORY_AFTER_SECONDS`, defaulting to 30 days. And
`mergeEventHistories` already detects the condition: with no common ancestor
it concatenates both histories, after which `validateEventHistoryContinuity`
throws `"Invalid event history chain"`. Today that surfaces as an opaque
failure with no way forward. Turning it into an explicit choice is a strict
improvement on what is there, independent of extensions entirely.

### 4.4 Tolerance ships on its own

The `wallet-frontend` tolerance work is not gated on any of this. It fixes a
live crash on two-device accounts and is worth landing whatever becomes of
the rest.

---

## 4A. Prior art: typed collections (`wwWallet/wallet-frontend#751`)

`SPEC.md` Appendix A carries the full comparison. The short version, and
what it changed here:

Emil's ARKG/Split-BBS work extends the same wallet state with **typed
collections** — `arkgSeeds[]`, `splitBbsKeypairs[]` — plus four typed
events, a reducer per collection, and a merge strategy per event type. The
approach was abandoned along with ARKG, but as a schema-and-merge design it
is the closest prior art, and it independently reached three of the same
conclusions:

- merge identity is a **per-entity key** (`credentialId`), never one
  aggregate per subsystem — §6.1.2 from the other direction;
- **deletion is an event**, participating in the merge like creation —
  §6.1.4's tombstones;
- **sort by timestamp, then deduplicate by key** — literally §6.1.4's
  last-write-wins, already working in the reference implementation.

Two convergent derivations of the same rules is the best evidence available
that they are right, so §6.1.4 is now specified to match that existing code
rather than stating a second formulation of it.

What §6.1 does differently — and this is the one demonstrable correctness
advantage, verified in the code rather than argued:

#751 merges each event type independently, and `deduplicateBy` keeps the
*first* entry of an ascending-sorted list, so the earliest event per key
wins within its type. Create `X` at t1, delete `X` at t2, re-create `X` at
t3 on a diverged branch, and the `new_*` bucket deduplicates to t1 while
discarding t3. The merged history creates `X` and then deletes it; the
re-creation is lost.

Last-write-wins on `(namespace, key)` with `null` as deletion turns the same
sequence into `value@t1, null@t2, value@t3` → the t3 value. #751 could be
fixed (deduplicate to the latest, or move deletion into the value); the
point is that its shape admits the error and ours cannot express it.

An earlier draft of this document claimed a broader ordering advantage.
That was wrong: `mergeDivergentHistoriesWithStrategies` does globally
`sort(compareBy(timestampSeconds))` before returning, so new-versus-delete
ordering across types is sound. The re-creation case above is the real and
narrower difference.

What it does worse: no type safety inside a value, and weak
discoverability. That is a real cost, which is why Appendix A §A.4 draws the
line by *who needs to understand the data* — typed collections where the web
wallet participates (renders it, names it, decides from it), extensions
where it only carries it. A namespace SHOULD graduate to a typed collection
if the web wallet ever needs to read inside it.

Worth stealing outright: `MaybeNamed<T>`, which layers an optional
user-facing label on a cryptographic record without the record's owner
modelling that concern.

---

## 5. Work, by repository

| Repo | Change | Owner |
|---|---|---|
| `privatedata-spec` | Normative §6.1; namespace registry; growth rule; legacy fields deprecated in §6.2; Appendix A; conformance vectors; conformance-runner fixes | this PR |
| `wallet-frontend` | `set_extension` event, generic reducer, LWW strategy, unknown-type tolerance, stale-peer resolution | separate session |
| `siros-sdk-kotlin` | `WalletExtensionStore`; migrate WSCD + refresh-token state; add BBS holder state | native SDK work |
| `siros-sdk-swift` | Same surface, mirrored | native SDK work |

### 5.1 Conformance vectors added here

The existing `metadata-preservation-001` fixture asserts only that *known*
fields survive decrypt and re-encrypt. Nothing covered unknown data, which
is why both defects in §2 went unnoticed. Six vectors close that:

| Vector | Asserts |
|---|---|
| `extensions-unknown-field-001` | An unrecognised top-level `S` field survives decrypt → re-encrypt |
| `extensions-unknown-event-001` | An unrecognised event type survives a merge, without raising |
| `extensions-concurrent-keys-001` | Two devices writing different keys in one namespace both survive — the case today's design loses |
| `extensions-tombstone-001` | A tombstone deletes, and is not resurrected by a peer at the fold horizon |
| `extensions-stale-peer-001` | A peer beyond the fold horizon is an unresolvable divergence, not a merge |
| `extensions-size-accounting-001` | Per-namespace size accounting is reported for a large container |
| `extensions-lww-fold-order-001` | Folding a prefix then the remainder converges with folding everything — what lets a client fold a namespace it does not understand |
| `extensions-lww-tiebreak-001` | Equal timestamps resolve by `eventId`, identically on every client |
| `extensions-unknown-namespace-retained-001` | An unrecognised namespace is ignored, preserved, and not an error |
| `extensions-version-coexistence-001` | Two versions of one namespace coexist; neither client deletes nor migrates the other's state |

### 5.2 Conformance runner

The runner could never have exercised the vectors added here, or the two
that already existed. Four defects, all pre-existing, all fixed in this PR:

1. `((COUNTER++))` returns the *old* value, so the first increment of a
   zero counter exits non-zero and `set -e` kills the script. This is why
   exactly one vector ever ran and no summary was ever printed.
2. The vector loop was a pipeline, so it ran in a subshell and every
   counter increment was discarded before `print_summary` read it. It now
   reads from a dedicated file descriptor, which also stops anything in the
   loop body consuming the vector stream.
3. `WALLET_FRONTEND_PATH` and `SIROS_SDK_SWIFT_PATH` were dereferenced but
   never assigned. Under `set -u` that aborts the run, and the caller's
   `2>/dev/null` made the error invisible.
4. A missing client harness returned the same code as a wrong answer, so
   "not installed" counted as a failure and, with `STOP_ON_FAIL` defaulting
   to 1, aborted everything. Unavailability now has its own return code and
   is reported as a skip.

A run where every client was skipped also reported "All tests passed ✓". It
now says plainly that nothing was exercised — a corpus that silently
verifies nothing is worse than one that fails.

---

## 6. Sequencing

Ordered and gated.

1. **Tolerance and vectors.** `wallet-frontend` stops crashing on unknown
   event types and carries them through merges; the null-merge-base case
   becomes a user-facing choice. No format change, no SDK change.
   *Gate: none — ships independently of `#183` entirely.*

2. **The extension namespace.** `set_extension`, the generic reducer and the
   LWW strategy land in `wallet-frontend`; §6.1 becomes normative. Nothing
   written to it yet.
   *Gate: step 1, so a client that predates the event type does not choke on
   it.*

3. **SDKs move across.** Both SDKs write through their extension store. BBS
   holder state goes straight there — it has no legacy form. WSCD state
   becomes per-`kid` entries, which is the fix for the two-device enrolment
   loss, not just a relocation.
   *Gate: step 2. Read-both, write-new for one release, then drop the legacy
   fields.*

4. **Fold into V4.** `S.extensions` carries into `#183`'s V4 state JWE
   unchanged. It is already on the side of the split that keeps the event
   log.
   *Gate: `#183` Phase 3.*

### 6.1 Interim behaviour for the SDKs

Until step 2 lands, an SDK that emits `set_extension` events would crash
`wallet-frontend`'s merge (§2.2). SDKs therefore adopt the `S.extensions`
*shape* first and defer event emission: writing to the state field alone is
no worse than the status quo — preserved on fold, lost on merge, exactly as
the legacy fields behave today — while unifying three ad-hoc fields into one
and fixing the per-entity key problem immediately.
