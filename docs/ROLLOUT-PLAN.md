# Rollout: four clients, one container

**Non-normative.** Sequencing for BBS, then optionally Automerge, across the
clients that share this container.

Date: 2026-08-28
Related: `SPEC.md` §6.1 · `docs/EXTENSIONS-DESIGN.md` ·
`docs/SPEC-ALTERNATIVE-AUTOMERGE.md` · `wallet-frontend#183` · `SUNET/vc#616`

---

## 1. The rule the plan is built on

> **At every moment, a client that has not yet learned a data kind must be
> able to carry it faithfully.**

The rule is about clients that do not *understand* a data kind, not clients
running an old build. The native SDKs have a negligible deployed base and can
change freely, so version skew between them is not the hazard. What remains
is a client asked to carry state it has never modelled — `wallet-frontend`
holding `S.extensions`, or any future client meeting a namespace registered
after it shipped.

Break it there and the failure is not degraded behaviour; it is a destroyed
credential, surfacing on whichever client happens to be last.

Every client implements BBS. What varies is *when* each arrives, so the plan
is ordered by what must be true before the next one does, not by which client
matters most.

## 2. Where each client stands

Verified against the repositories, 2026-08-28.

| Client | Carries unknown state | Container binding | Presents BBS | Commits at issuance | Persists holder state | Request fields wired |
|---|---|---|---|---|---|---|
| `siros-sdk-kotlin` | yes | UniFFI | yes (#117) | yes (#123) | **gap** | **gap** |
| `siros-sdk-swift` | yes | UniFFI | yes (#113) | yes (#114) | **gap** | **gap** |
| `wallet-frontend` | fold only | **none** | **gap** | **gap** | **gap** | **gap** |
| `vc` (issuer/verifier) | n/a | cgo | verifies | #616 open | n/a | n/a |

**The SDKs already satisfy the invariant.** `JweKeystore` parses the container
plaintext into a `JsonObject` and holds it in `preservedWalletState` for
round-trip, so anything it does not understand survives verbatim. Swift
mirrors it.

**`wallet-frontend` satisfies it halfway, and the missing half is the
dangerous one.** Unknown `S` fields survive a fold — `foldState` starts from
`container.S` and the reducer spreads `{...state}`. They do *not* survive a
merge, which sets `S` to the common-ancestor base state and replays events.
And an unrecognised event type does not pass through at all:
`mergeDivergentHistoriesWithStrategies` buckets against a literal map of nine
known types, so it dereferences `undefined` and throws on a `412` conflict.

**The browser is the one consumer of four that never got the container.**
`zk-cred-bbs`'s `src/js_api.rs` exposes the raw BBS algebra and none of the
`jwp*` functions, so a web BBS client would reimplement the
claim-to-message mapping in TypeScript — the drift the container exists to
prevent.

**`go-wallet-backend` needs nothing.** `UpdatePrivateData` takes the raw
body, tries `{"$b64u": …}` and falls back to raw bytes; the ETag is over
bytes. No JSON validation, no content-type check, no size limit in the
handler.

## 3. What is already in the container

BBS is not the first extension. Two are live today, written by both SDKs.

| Field today | Keyed by | Target namespace | Migration |
|---|---|---|---|
| `S.credentialRefreshTokens` | `batchId` | `org.siros.oid4vci.refresh` | relocation — keying already per-entity |
| `S.wscdCredentials` | plugin id | `org.siros.wscd` | re-key — blocked, see §3.1 |
| — none — | credential id | `org.siros.bbs` | new, no legacy form |

### 3.1 The defect underneath `wscdCredentials`

One entry per plugin is the keying that loses an authenticator when two
devices enrol concurrently. Re-keying per `kid` does not fix it, because
`kid` is not unique — and this is not one plugin's problem.

| Plugin | Identifier | Collides across devices | Exported state |
|---|---|---|---|
| `preview_sign` (fido2) | `fido-{next_id}` | **yes** | `{keys, next_id, lifecycle}` |
| `softkey` | `sw-{next_id}` | **yes**, narrower window | `{keys, lifecycle}` |
| `r2ps` | assigned by the remote service | no | — |

```rust
let kid = format!("fido-{}", state.next_id);   // preview_sign
let kid = format!("sw-{}", state.next_id);     // softkey
state.next_id += 1;
```

Two devices starting from the same counter both mint the same identifier for
*different keys*. That is not one authenticator lost — it is an id collision:
two distinct keys sharing a `kid`, one silently unaddressable while appearing
present.

Three details decide the size of the fix:

- **The prefixes save the namespace.** `fido-`, `sw-` and r2ps's
  server-assigned identifiers cannot collide with each other, so a flat
  per-`kid` entry key is safe between plugins and need not become compound.
- **fido2 is strictly worse than softkey.** It persists `next_id` *in the
  exported state* — a shared mutable counter inside a synchronised container,
  where merging two values means nothing. Softkey does not sync it; it
  recomputes `next_id = max(existing) + 1` on import. That repair narrows the
  window without closing it.
- **r2ps is the existence proof.** Its identifier comes from the remote
  service, so there is no local allocator and no collision.

So the change is uniform: drop both counters, derive the identifier from the
key material or randomise it, and both exports collapse to
`{keys, lifecycle}`, which splits per `kid` cleanly. `lifecycle` is then the
one genuinely plugin-scoped item left, and needs an answer of its own — if it
is per-device it belongs in local storage rather than a synchronised
container.

### 3.2 Migrating the existing extensions is just a change

An earlier draft gave each existing extension a read-both, write-both window.
That is unnecessary, for two reasons worth separating because only the second
depends on the deployed base:

- **Only the SDKs interpret these fields.** `wallet-frontend` has never
  modelled `wscdCredentials` or `credentialRefreshTokens` — it carries them.
  Moving them under `S.extensions` is invisible to it.
- **The SDK deployed base is small enough to change freely.**

The identifier fix is likewise forward-only: existing `fido-0` and `sw-3`
entries keep their identifiers and stay addressable; only newly allocated
keys take the new form. No migration, no re-enrolment.

## 4. Sequence

**0 — `wallet-frontend`: make the invariant true.** Unknown event types
bucketed into a passthrough group instead of indexing a fixed map, with a
union-and-dedupe default strategy so they survive a merge. Unknown `S` fields
preserved across merge, not only fold. The null-merge-base case surfaced as a
user choice rather than dead-ending in `"Invalid event history chain"`.
*Gate: none.* Cheap, and load-bearing three times over — extensions now,
shared accounts next, any future format migration after that.

**1 — `vc`: land the issuer.** `SUNET/vc#616`, six checks green, waiting on a
maintainer. The only external dependency in the plan. *Gate: none.*

**2 — each SDK, in any order: join the two ends.** `credentialRequestFields`
is produced and dropped on the floor; `BbsHolderState` is produced by
`accept()` and read by `BbsProofSystem` but never persisted. Implement
`BbsHolderStateStore` over an extension store writing `org.siros.bbs`, and
wire the request fields into the OID4VCI credential request. The store is
shared with 2b, not built for BBS alone — BBS is simply the one namespace
with no legacy form, and so the cheapest first user.
*Gate: step 1 for an end-to-end run.*

**2b — both SDKs and `siros-wscd-manager`: migrate the existing extensions.**
`credentialRefreshTokens` is a straight relocation. `wscdCredentials` cannot
start until `kid` allocation stops using a shared counter in both the fido2
and softkey plugins. That fix is forward-only and worth making regardless of
whether the extension ever moves.
*Gate: the `kid` fix for the WSCD half; nothing for the refresh half.*

**3 — acceptance: issue on one client, present on another.** Same account,
second client on an older build. Any single client's end-to-end run proves
the feature; only this proves the mechanism. Add the case of a client that
has never seen the namespace — `wallet-frontend` carrying `org.siros.bbs` it
does not model, then handing it back intact.
*Gate: step 2 on two clients.*

**4 — `wallet-frontend`: bring the browser in.** Add the six `jwp*` functions
to `js_api.rs` — same code, fourth binding — then holder-state handling and
presentation. BBS then graduates from state the web wallet carries to state
it models.
*Gate: step 0. Independent of 2–3.*

**5 — Automerge.** See §5.

## 5. Automerge, if adopted

`docs/SPEC-ALTERNATIVE-AUTOMERGE.md` specifies the destination. This is how
it would be reached.

**The constraint that shapes everything:** an Automerge document is not
derivable. Two clients converting the same container independently produce
different documents that *duplicate* rather than reconcile, with no repair.
Conversion must happen once and be distributed.

**5a. Shared crate.** `automerge-rs` behind UniFFI and wasm — the shape
already proven by `zk-cred-bbs`, `zk-cred-longfellow`, `zk-cred-vega` and
`siros-wscd-manager`.

**5b. Projection.** `document → V3-shaped view`, and a one-time
`V3 snapshot → document`.

**5c. Shadow mode.** Maintain the document alongside, read from existing
state, diff the projection on every load and report divergence. No
user-visible change. This is where document growth and artifact growth get
measured — the numbers that decide whether to continue.

**5d. Convert.** Elected once via the backend ETag as compare-and-swap.
Document becomes truth; the legacy representation becomes a projection of it,
maintained write-through.

**5e. Retire the projection** once no legacy writes are observed.

The small deployed base helps: the conversion's blast radius is proportional
to how many containers exist when it runs, so converting early is cheaper
than converting late.

### 5.1 Two decisions this forces

**Defer the V3 → V4 split.** `wallet-frontend#183` plans one. Since it is
ours, make the Automerge conversion the single migration — one dangerous
one-time conversion per account rather than two. #183's key separation
survives as a layout choice: document in one JWE, keys in another.

**Agree the wasm budget deliberately.** `siros-wscd-manager` 94 KB +
`zk-cred-bbs` 97 KB + Automerge ~147 KB ≈ **340 KB brotli** in one web
wallet, all measured. Retiring the schema machinery returns 3.5 KB.

## 6. What would stop this

- **Document growth that does not compact acceptably** over a realistic
  multi-year wallet. Measured in 5c, before 5d.
- **Shadow-mode divergence** that does not resolve.
- **Single-use secrets.** A CRDT merges two branches that each spent a
  `refresh_token` into something plausible instead of flagging it. Those need
  explicit leases either way; Automerge makes the failure quieter, not
  louder.

Size is deliberately not on this list. It is a real cost and a known one, and
it is not the reason to stop.
