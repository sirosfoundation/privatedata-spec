# SIROS Wallet Private Data Blob Specification (Normative)

Version: 2.1  
Date: 2026-08-28  
Status: Normative (reference implementation aligned)

## 1. Scope and Source of Truth

This document defines the normative format and processing rules for the wallet private data blob ("privateData"), which contains encrypted wallet state including credential key material and credentials.

The canonical source of truth is the web wallet implementation in `wallet-frontend`.

If any SDK behavior differs from this document, the `wallet-frontend` behavior is normative.

## 2. Normative Language

The terms MUST, MUST NOT, SHOULD, SHOULD NOT, and MAY are to be interpreted as in RFC 2119.

## 3. Blob Transport and Encoding

### 3.1 Stored backend field

`privateData` is stored as a serialized UTF-8 JSON container blob.

### 3.2 HTTP transport

- `POST /user/session/private-data` MUST receive the serialized container bytes.
- `GET /user/session/private-data` returns JSON with `privateData` encoded as tagged binary:

```json
{
  "privateData": { "$b64u": "<base64url(container-json-bytes)>" }
}
```

### 3.3 Tagged binary representation

Binary fields in JSON MUST be encoded as:

```json
{ "$b64u": "<base64url-no-padding>" }
```

This applies both to the top-level response payload wrapper and binary fields inside the container.

## 4. Container Model

The current normative output format is the asymmetric encapsulation container:

```json
{
  "mainKey": { ... },
  "prfKeys": [ ... ],
  "jwe": "<compact-jwe>"
}
```

### 4.1 Top-level fields

- `mainKey` MUST be present in newly written containers.
- `prfKeys` MUST be present (possibly empty).
- `jwe` MUST be present and MUST be a compact JWE string.

### 4.2 `mainKey`

`mainKey` defines how a content-encryption main AES key is decapsulated by PRF entries.

`mainKey.publicKey.importKey` MUST contain:

- `format`: `"raw"`
- `keyData`: tagged binary, uncompressed P-256 point (65 bytes)
- `algorithm.name`: `"ECDH"`
- `algorithm.namedCurve`: `"P-256"`

`mainKey.unwrapKey` MUST contain:

- `format`: `"raw"`
- `unwrapAlgo`: `"AES-KW"`
- `unwrappedKeyAlgo.name`: `"AES-GCM"`
- `unwrappedKeyAlgo.length`: `256`

### 4.3 `prfKeys[]`

Each PRF key entry corresponds to one WebAuthn credential.

Required fields:

- `credentialId` (tagged binary)
- `prfSalt` (32 bytes tagged binary)
- `hkdfSalt` (32 bytes tagged binary)
- `hkdfInfo` (tagged binary, normative value below)
- `keypair` (encapsulation keypair)
- `unwrapKey` (wrapped main key)

Optional fields:

- `transports`
- `algorithm` (if present, MUST be AES-GCM 256)

`keypair.publicKey.importKey` and `keypair.privateKey.unwrapKey` MUST encode ECDH P-256 keypair material as in the reference.

`unwrapKey.wrappedKey` MUST be AES-KW wrapped main key bytes.

## 5. Cryptographic Processing

## 5.1 PRF-derived key

Given WebAuthn PRF output (`prfOutput`), derive PRF wrapping key:

- HKDF hash: SHA-256
- IKM: PRF output
- salt: `hkdfSalt`
- info: `hkdfInfo`
- output length: 32 bytes

Normative default info string is UTF-8 `"eDiplomas PRF"`.

## 5.2 Main key decapsulation flow

1. Decrypt `keypair.privateKey.unwrapKey.wrappedKey` with PRF key using AES-GCM (`unwrapAlgo.iv`).
2. Import resulting ECDH private key (JWK).
3. ECDH with `mainKey.publicKey`.
4. Use derived secret as AES-KW key material (256-bit key).
5. Unwrap `prfKeys[i].unwrapKey.wrappedKey` to obtain main key.

## 5.3 Payload encryption

`jwe` MUST use:

- `alg`: `A256GCMKW`
- `enc`: `A256GCM`

The decrypted plaintext MUST be UTF-8 JSON WalletStateContainer (Section 6).

## 6. JWE Plaintext: WalletStateContainer

The plaintext JSON MUST follow the event-sourced wallet schema used by `wallet-frontend`:

```json
{
  "lastEventHash": "...",
  "events": [ ... ],
  "S": {
    "schemaVersion": 3,
    "credentials": [ ... ],
    "keypairs": [ ... ],
    "presentations": [ ... ],
    "settings": { "openidRefreshTokenMaxAgeInSeconds": "..." },
    "credentialIssuanceSessions": [ ... ]
  }
}
```

Normative requirements:

- `S.schemaVersion` MUST be `3` for newly written state.
- `events` and `lastEventHash` MUST be preserved as authored unless explicitly folded/merged by event logic.
- `S.keypairs[].keypair.privateKey` MUST contain JWK private key material for ES256 signing keys.
- `S.credentials[]` entries MUST preserve wallet metadata fields (`format`, `kid`, `instanceId`, `batchId`, `credentialIssuerIdentifier`, `credentialConfigurationId`).
- `S.settings.openidRefreshTokenMaxAgeInSeconds` MUST be preserved.
- `presentations` and `credentialIssuanceSessions` MUST be preserved.

### 6.1 `S.extensions` (normative)

Clients MAY carry state that this specification does not define, under a
single namespaced field:

```json
"extensions": {
  "org.siros.wscd": { "kid-a1b2": "<opaque string>", "kid-c3d4": "<opaque string>" },
  "org.siros.bbs":  { "1849302113": "<opaque string>" }
}
```

`S.extensions` is an object keyed by **namespace**; each namespace is an
object keyed by **entry key**; each value is an opaque string whose meaning
belongs entirely to the namespace owner. Readers MUST NOT interpret a value
in a namespace they do not own, and MUST preserve namespaces they do not
recognise.

#### 6.1.1 Why a single field

Before this section, each new kind of client state was added as its own
top-level field under `S` (see §6.2). That required a specification change,
a `wallet-frontend` reducer, and a merge strategy per addition, and every
such field carried the same unresolved data-loss note. One namespaced field
with one generic reducer removes the per-addition cost: a new namespace is a
registry entry, not a schema change.

#### 6.1.2 Entry keys MUST name an entity

An entry key MUST identify a single entity that the wallet already tracks —
a key identifier, a credential identifier, a batch identifier. An entry key
MUST NOT name a subsystem, a plugin, or a category.

This is a correctness requirement, not a style preference. Merge resolution
is last-write-wins per `(namespace, key)` (§6.1.4). A namespace that stores
one aggregate value per subsystem therefore loses data whenever two devices
write concurrently: two devices each enrolling a different authenticator
produce two whole-subsystem blobs, and one silently wins. Keyed per entity,
the same two writes touch disjoint keys and merge without conflict.

#### 6.1.3 Size MUST be bounded by entities, not by history

An implementation cannot in general predict how large a namespace becomes —
it grows with enrolled authenticators, issued credentials, or renewable
batches. This specification therefore constrains growth rather than size:

- A namespace's total size MUST be proportional to entities the user can
  enumerate and delete.
- A namespace MUST NOT grow with the number of events. In particular, an
  entry value MUST NOT accumulate an append-only history; only current state
  belongs in an entry.
- Deleting the entity a key names MUST delete the entry (§6.1.4).

Implementations SHOULD report per-namespace sizes, so that a container
approaching the transport limit identifies which namespace is responsible.

#### 6.1.4 Events, reduction and merge

`S` is a fold cache; `events` are the source of truth (§6, §8). State
written only into `S` is not reconstructible after a history merge, which
replays events onto a common-ancestor base state. Extension state therefore
MUST be written as events:

```json
{
  "type": "set_extension",
  "schemaVersion": 3,
  "eventId": 1234,
  "parentHash": "...",
  "timestampSeconds": 1756400000,
  "namespace": "org.siros.bbs",
  "key": "1849302113",
  "value": "<opaque string>"
}
```

- Reduction: set `S.extensions[namespace][key]` to `value`.
- A `value` of `null` is a **tombstone**: reduction MUST delete the key.
- Merge: last-write-wins per `(namespace, key)`, ordered by
  `timestampSeconds`.

Tombstones MUST survive event-history folding for at least as long as the
maximum permitted time between folds (§6.1.5); otherwise a merge with a
long-absent peer resurrects a deleted entry.

#### 6.1.5 Peers beyond the fold horizon

Implementations fold event history older than a configured horizon into `S`.
A peer that has not synchronised within that horizon no longer shares a
reachable common ancestor, and no correct merge exists for any part of the
state.

Implementations MUST NOT silently reconcile such histories. When no common
ancestor is found, the implementation MUST surface the divergence and let
the user choose which wallet is authoritative. Concatenating the two
histories produces an invalid event chain and MUST NOT be treated as a
merge result.

#### 6.1.6 Namespace registry

Namespaces are reverse-DNS strings and are registered here. Registration is
a change to this document.

| Namespace | Owner | Entry key | Value |
|---|---|---|---|
| `org.siros.wscd` | WSCD plugins (native SDKs) | `kid` | Key metadata needed to address a key created on a roaming authenticator — never private key material |
| `org.siros.bbs` | Blind BBS credentials (native SDKs) | credential id | Holder state a BBS credential cannot be presented without: blinding factor, committed messages, bound key binding public keys |
| `org.siros.oid4vci.refresh` | OID4VCI renewal (native SDKs) | `batchId` | `refresh_token` and the DPoP key it is bound to, per renewable batch |

### 6.2 Legacy top-level extension fields (deprecated)

Native SDKs released before §6.1 write two top-level fields under `S`:
`wscdCredentials`, keyed by WSCD plugin ID, and `credentialRefreshTokens`,
keyed by stringified `batchId`. Neither was ever described by a published
version of this document, though other documents and source comments refer
to them.

Readers MAY accept both for migration. Writers MUST NOT emit them; the
equivalent namespaces in §6.1.6 replace them. `wscdCredentials` in
particular is not merely relocated: keyed per plugin rather than per `kid`,
it violates §6.1.2 and loses an authenticator whenever two devices enrol
concurrently.

## 7. Legacy Compatibility Rules

`wallet-frontend` accepts legacy containers where PRF/password entries directly contain wrapped main keys (`mainKey` symmetric wrapping style) and may upgrade them to asymmetric encapsulation style.

Normative behavior:

- Readers MAY accept legacy formats.
- Writers SHOULD emit asymmetric format (`mainKey` + encapsulation keypair per entry).

## 8. Concurrency and Sync

Backend optimistic concurrency is part of privateData lifecycle:

- Clients SHOULD use `X-Private-Data-If-Match` with backend ETag (`X-Private-Data-ETag`).
- On `412 Precondition Failed`, clients SHOULD reconcile by re-fetching remote privateData and merging according to wallet event-history rules before retrying.

## 9. Reference Implementation Pointers

- `wallet-frontend/src/services/keystore.ts`
- `wallet-frontend/src/services/WalletStateSchemaVersion1.ts`
- `wallet-frontend/src/services/WalletStateSchemaVersion3.ts`
- `wallet-frontend/src/util.ts`

---

## Appendix A. Comparison with typed schema collections (`wwWallet/wallet-frontend#751`)

`wwWallet/wallet-frontend#751` ("Add support for JPT with Split-BBS, and JWT
key generation via ARKG") extends the same wallet state with two new kinds
of client data, using a different mechanism. That work was not adopted — the
ARKG approach it serves was superseded by the WSCD-manager design — but as a
schema-and-merge design it is the closest prior art to §6.1, and it is
better than §6.1 in ways worth being explicit about.

### A.1 What it does

It introduces a new schema version carrying two **typed collections**:

```typescript
export type WalletStateV3 = SchemaV2.WalletState & {
  arkgSeeds: MaybeNamed<WebauthnSignArkgPublicSeed>[],
  splitBbsKeypairs: MaybeNamed<WebauthnSignSplitBbsKeypair>[],
}
```

with four typed events (`new_arkg_seed`, `delete_arkg_seed`,
`new_split_bbs_keypair`, `delete_split_bbs_keypair`), a reducer per
collection, and a merge strategy per event type. Each strategy is the same
shape:

```typescript
deduplicateBy(
  a.concat(b).filter(e => e.type === 'new_arkg_seed')
   .sort(compareBy(e => e.timestampSeconds)),
  e => toBase64Url(e.arkgSeed.credentialId),
)
```

### A.2 Where it agrees, and what §6.1 takes from it

Three of its choices arrived independently at the same conclusions as
§6.1, which is the strongest available evidence that those rules are right:

- **Merge identity is a per-entity key.** Deduplication is by
  `credentialId` — one entry per authenticator credential, never one
  aggregate per subsystem. This is §6.1.2, reached from the other
  direction.
- **Deletion is an event, not an absence.** `delete_arkg_seed` participates
  in the merge exactly as the creation event does. These are §6.1.4's
  tombstones under another name.
- **Ordering before deduplication.** `sort(compareBy(timestampSeconds))`
  then `deduplicateBy(key)` is precisely §6.1.4's last-write-wins, and it
  is already proven code in the reference implementation. §6.1.4's merge
  rule is specified to match it rather than inventing a second formulation.

One further idea is worth borrowing: `MaybeNamed<T>` layers an optional
user-facing `name` on top of a cryptographic record, so a wallet can label
an authenticator without the record's owner having to model that. A
namespace MAY adopt the same convention within its own entry values.

### A.3 Where the two differ

| | Typed collections (#751) | `S.extensions` (§6.1) |
|---|---|---|
| Type safety | Full — compile-time types, exhaustive reducers | None inside a value; the value is opaque |
| Cost of a new data kind | New event types, reducer, merge strategy, schema version, migration | One registry row |
| Who must change | The web wallet, for every addition | Nobody, once the mechanism exists |
| Cross-client carriage | Only clients that model the type can hold it | Any client can carry any namespace |
| Coordination | A new data kind needs a change in the web wallet | A new data kind needs a registry row |
| Merge correctness | A hand-written strategy per type | One generic strategy; correctness comes from §6.1.2 |
| Discoverability | Excellent — the shape is in the type system | Weak — meaning lives with the namespace owner |

§6.1 also makes one class of merge error unrepresentable. #751 merges each
event type independently — `new_*` events are deduplicated against each
other by entity id, `delete_*` events likewise — and `deduplicateBy` retains
the *first* entry of an ascending-sorted list, so the earliest event per key
wins within its type. Consider one entity across a divergence:

| | |
|---|---|
| `t1` | device A creates entity `X` |
| `t2` | device A deletes `X` |
| `t3` | device B, diverged, creates `X` again |

The `new_*` bucket deduplicates to the `t1` creation and **discards the
`t3` re-creation**; the `delete_*` bucket keeps `t2`. The merged history is
then globally sorted and reduced, so `X` is created and then deleted, and
the later re-creation is lost.

Expressing deletion as a `null` value on the same `(namespace, key)` reduces
creation and deletion to one ordered sequence per key: `value@t1`,
`null@t2`, `value@t3` resolves to the `t3` value.

This is a property of the model rather than an unfixable defect in #751 —
deduplicating to the latest entry, or moving deletion into the value, would
correct it. The point is that the typed-collection shape admits the error
and the reference implementation contains it, while last-write-wins on a
single key cannot express it.

### A.4 When to use which

They are complementary, not competing, and the distinction is who needs to
understand the data:

- **Use a typed collection** when the web wallet itself participates in the
  data — renders it, lets the user name or delete it, or makes decisions
  from its contents. #751's ARKG seeds are exactly this: they appear in
  Settings and carry user-assigned names. Type safety and UI integration
  are worth a schema version there.
There is also a practical asymmetry behind that line. This organisation's
web wallet is a fork of `wwWallet/wallet-frontend`. Under the typed-collection
model, every new native-SDK data kind requires a new event type, reducer,
merge strategy and schema version *in the web wallet* — for state the web
wallet cannot use and has no reason to model. That is either permanent fork
divergence or an upstream change for someone else's data. An extension
namespace lets the web wallet carry the state faithfully while modelling
nothing.

- **Use an extension namespace** when the data is opaque to the web wallet
  and owned by another client. WSCD key metadata, BBS holder state and
  OID4VCI refresh material are all in this category: the web wallet needs
  to carry them faithfully and nothing more. Requiring it to model those
  types is pure cost, and it is the cost that produced the
  "not yet normative" fields in §6.2.

A namespace SHOULD graduate to a typed collection if the web wallet ever
needs to interpret its contents. Extensions are the right home for state in
transit between clients, not a permanent substitute for modelling state the
wallet genuinely owns.
