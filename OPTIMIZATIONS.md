Future-Work Optimizations
=========================

`design notes` `non-normative` `not-yet-a-NIP`

This file is a **design scratchpad**, not a NIP. It captures byte-budget
and discovery optimizations that could compose on top of NIPs N0–N5
once the underlying questions are settled with the relevant upstream
maintainers (JeremyRand for the Namecoin record schema, the
[`nostr-protocol/nips`][upstream-nips] editors for any Nostr-side
shape, Shock for [CLINK][clink]). Nothing here is implemented and
nothing here is proposed for merge as a NIP. If/when any individual
section matures, it gets promoted to its own `N*.md` file (or a PR
to upstream NIPs / namecoin.org) and removed from this document.

[upstream-nips]: https://github.com/nostr-protocol/nips
[clink]: https://github.com/shocknet/CLINK

## Motivation

The current Namecoin track (N0–N5) was designed to fit comfortably
inside the 520-byte `name_update` value limit using **plaintext** JSON
records under the `nostr` item, plus the existing hashed-TLSA
primitive ([namecoin.org PR #747][nmco-pr-747], merged Feb 2026) for
the `tls` field. That works well today, but the byte budget tightens
quickly as soon as a name wants to carry:

- a hashed TLSA leaf (~32 B post-#747),
- an SSHFP leaf (a separate 32 B hash if added the same way),
- a Nostr identity (the `nostr.names` map, typically 40–80 B for a
  single localpart, more for `_` + named users),
- one or more relays (URL strings, 30–80 B each in `nostr.relays`),
- a CLINK `clink_offer` and `clink_debit` (bech32 strings, hundreds of
  bytes each — see NIP-N5 §"520-byte budget"),
- future leaves we have not yet imagined.

For small names (single localpart, one or two relays, one TLSA hash)
the plaintext encoding is fine. For richer names — exactly the names
where Namecoin's anchoring is most valuable — the budget pressure is
real, and the [`dd/<name>` factoring pattern][n5-budget] only buys
one or two extra slots before the next name has to be registered.

[nmco-pr-747]: https://github.com/namecoin/namecoin.org/pull/747
[n5-budget]: ./N5.md

There are two orthogonal axes of optimization:

1. **Logical compression** — replace N independent on-chain hashes with
   one Merkle root over the same N leaves, serving the expansion
   off-chain (typically as a Nostr addressable event).
2. **Byte-slot relocation** — move the 32 B commitment from the
   Namecoin name-value (which lives in the non-witness portion of the
   scriptPubKey today) into Taproot witness data, where each byte
   costs 4× less weight, or — better — into the Taproot output key
   itself, where the commitment is free.

The two axes multiply. Layer 1 alone takes 96+ B of independent hashes
down to 32 B; Layer 2 alone takes 32 B of non-witness data down to
~8 B of witness data, or 0 B marginal if it's tweaked into the output
key. Composed end-to-end they are roughly an order of magnitude
cheaper per record set, with effectively unbounded room for
never-revealed variants under [P2MR][p2mr] (not yet active on Bitcoin
at time of writing).

[p2mr]: https://en.bitcoin.it/wiki/P2MR

## Status of Bitcoin / Namecoin consensus features

| Feature | Bitcoin mainnet | Namecoin mainnet | Used in this doc |
|---|---|---|---|
| Taproot (BIP-341/342) | Active | Active (merge-mined) | Layer 2a (today) |
| Tagged hashes (BIP-340) | Active | Active | Layer 1 (today) |
| Hashed TLSA (32 B) | n/a | Live (post-PR #747) | Layer 1 baseline |
| P2MR / MAST | Not yet active | Not yet active | Layer 2b (future) |
| Cross-input signature aggregation | Not yet active | Not yet active | Out of scope |

Namecoin follows Bitcoin's soft-fork cadence (merge-mined). Anything
that requires a Bitcoin soft-fork is gated by Bitcoin's activation,
not by Namecoin's.

----

Layer 1 — Merkle compression of PKI / identity leaves
-----------------------------------------------------

### 1.1 Construction

Replace independent on-chain hashes with a single 32 B commitment:

```
root = MerkleRoot(
  leaf("nmc/tlsa",  tlsa_data),
  leaf("nmc/sshfp", sshfp_data),
  leaf("nmc/nostr", nostr_data),
  leaf("nmc/import", import_data),
  ...
)
```

Each leaf MUST be computed as a **BIP-340-style tagged hash**:

```
leaf(tag, data) = SHA256(SHA256(tag) || SHA256(tag) || data)
```

The tag string is non-negotiable and locked at spec time per leaf type
(changing a tag later silently invalidates every prior commitment).
Domain separation is mandatory: without it, an attacker who controls
one leaf type's preimage space can grind values that are also valid as
other leaf types.

### 1.2 Expansion event

Each Namecoin name that uses a Merkle commitment SHOULD publish a Nostr
addressable event carrying the leaves:

- `kind`: parameterised replaceable, in the `30000–39999` range
  (specific number TBD, but follows the convention of
  [NIP-01 §"Kinds"][nip-01-kinds] and existing namespaces such as
  NIP-23 long-form (`30023`) and NIP-72 community definition (`34550`)).
- `tags`:
  - `["d", "<name>.bit"]` — addressable identifier
    (per [NIP-01][nip-01] `d`-tag semantics), so the event is unique
    per name and can be replaced by a newer one from the same author.
  - `["i", "nmc:<namespace>/<name>"]` — external content ID
    (per [NIP-73][nip-73]) so the event can be discovered by any
    relay that indexes the `#i` tag. Example: `nmc:d/example` or
    `nmc:id/alice`. See §3.2 for the proposed `nmc:` prefix.
  - `["t", "<name>.bit"]` — optional hashtag (per [NIP-24][nip-24]
    §"Tags") for ambient discoverability in regular hashtag feeds.
  - `["nmc-merkle-root", "<32-byte hex>"]` — the root the leaves
    Merkle-hash to, for client-side cross-checks before the on-chain
    fetch completes.
- `content`: JSON object with the leaves as plaintext, plus optional
  per-leaf salts:

  ```json
  {
    "tlsa":  { "data": "...", "salt": "..." },
    "sshfp": { "data": "...", "salt": "..." },
    "nostr": { "names": { ... }, "relays": { ... },
               "clink_offer": "noffer1...", "clink_debit": "ndebit1..." },
    "import": { "from": "dd/example" }
  }
  ```

Salts are optional and per-leaf. They matter only for leaves whose
preimage space is small enough to brute-force; for the leaves
currently defined in N0–N5 (public TLSA records, public SSHFP records,
public Nostr pubkeys, public CLINK pointers) the preimage is meant to
be discoverable and salts can be omitted.

[nip-01]: https://github.com/nostr-protocol/nips/blob/master/01.md
[nip-01-kinds]: https://github.com/nostr-protocol/nips/blob/master/01.md#kinds
[nip-24]: https://github.com/nostr-protocol/nips/blob/master/24.md
[nip-73]: https://github.com/nostr-protocol/nips/blob/master/73.md

### 1.3 Verification flow

1. Client reads the Namecoin name and finds a 32 B Merkle root in the
   record (under a field name TBD — leading candidates `merkle` or
   `mr`, alongside but not replacing the existing `tls` field for
   backwards compatibility).
2. Client queries any subset of its Nostr relays for the addressable
   event:

   ```json
   ["REQ", "sub1", {
     "kinds": [<merkle-expansion-kind>],
     "#d": ["<name>.bit"]
   }]
   ```

3. Client recomputes the tagged-hash Merkle root over the returned
   leaves and compares to the on-chain value.
4. On match, the client uses the leaf data for the relevant protocol
   layer (TLS handshake validation, SSH host-key validation, Nostr
   identity verification, CLINK payment dispatch, etc.).

If multiple relays return divergent events, the one whose recomputed
root matches the on-chain commitment wins. This is the same shape as
NIP-N1 NIP-05 verification: the chain is the anchor, the relay is the
delivery channel, and a malicious relay cannot forge a passing event
because it would have to find a SHA256 second-preimage.

### 1.4 Wildcard / inheritance interaction

Today the Namecoin name schema lets a parent name's `tls`, `nostr`,
etc. fields inherit into child names via the `map` item (ifa-0001
§"map", §"_"). With a single Merkle root over the whole record set,
inheriting the parent's root implicitly inherits *every* leaf —
including SSH host keys and Nostr identity — which is almost never
what a child wants.

Two reasonable resolutions, both deferred to spec time:

- **Per-leaf inheritance.** The root commits not to a flat leaf set
  but to a `(leaf_type, leaf_data)` map, and `map`-walked children
  inherit per-leaf-type rather than the whole root. Implementation
  cost: child resolvers must materialise the parent's leaves and
  re-Merkle-ise their own override set.
- **Mandate `d/`+`dd/` factoring.** Children that need a different
  SSH key or Nostr identity register a sibling name (per
  [delegated-alteration][nmco-delegated]) and pin their own Merkle
  root there. Implementation cost: more on-chain names per deployment,
  but each child name carries its own 32 B root and the parent stays
  pristine. This is the pattern N5 already uses for budget reasons
  (`dd/<base>` → `d/cl-<base>` for CLINK records that don't fit).

[nmco-delegated]: https://www.namecoin.org/docs/name-owners/delegated-alteration/

### 1.5 Special-case: the on-chain commitment IS the Nostr pubkey

A Nostr pubkey is exactly 32 bytes (x-only secp256k1, per
[NIP-01][nip-01] §"Events and signatures" and [BIP-340][bip340]).
The simplest possible version of Layer 1 is therefore:

> The 32 B on-chain field is a Nostr pubkey directly. The expansion
> event is signed by that pubkey. Everything else (TLSA, SSHFP, CLINK
> pointers, relay lists, ...) is published as content inside that
> signed event.

This is **astonishingly clean** — Namecoin becomes a pure
"this name owns this Nostr pubkey" anchor, and *all* schema
evolution happens on the Nostr side under the keyholder's signature.
No tagged-hash domain separation needed (because there's only one
leaf), no Merkle tree, no per-leaf salt question.

The cost: the on-chain commitment is now opinionated about identity.
A name that wants to commit to TLSA + SSHFP without claiming a Nostr
identity loses the slot, and a name that wants to rotate its Nostr key
without re-registering on-chain has to use the `import` indirection.

This special case is *not* mutually exclusive with the full Merkle
scheme — they could be specified as two profiles ("Profile A:
Nostr-anchored", "Profile B: PKI-anchored") with a one-byte type
prefix on the commitment, or as two different field names.

[bip340]: https://github.com/bitcoin/bips/blob/master/bip-0340.mediawiki

----

Layer 2 — Byte-slot relocation via Taproot / P2MR
-------------------------------------------------

### 2.1 Cost model

Bitcoin (and Namecoin) charge transaction weight, not raw bytes:

- Non-witness bytes: 4 weight units per byte.
- Witness bytes: 1 weight unit per byte.
- 1 vbyte = 4 weight units.

A 32 B commitment in the scriptPubKey costs 32 vbytes. The same 32 B
in witness data costs 8 vbytes. Free-riding inside an already-paid-for
field (such as a Taproot output's pubkey) costs 0 marginal vbytes.

### 2.2 Tier 1a — Tapscript leaf (Taproot, available today)

The PKI/identity Merkle root can be carried inside a never-executing
Tapscript leaf, using the same `OP_FALSE OP_IF <data> OP_ENDIF`
envelope that Ordinals uses for inscriptions ([BIP-341][bip341]):

```
OP_FALSE OP_IF
  <32 B PKI root>
OP_ENDIF
OP_TRUE   ; or any always-true script
```

The Taproot output commits to a Merkle root over all script leaves
(BIP-341 §"Constructing and spending Taproot outputs"). At spend
time (or, for the Namecoin name-update use case, at name-update
time — see §2.5 caveat) the leaf is revealed in the witness with a
control block providing the Merkle path.

Cost: ~8–12 vbytes witness per reveal, depending on tree depth
(`control_block` is 33 + 32·depth bytes, leaf script is a handful of
bytes plus the 32 B payload).

### 2.3 Tier 1b — Tweak the commitment into the output key (Taproot, free)

This is the real prize, and it's available **today**, no fork needed.

Taproot tweaks the internal key with the script Merkle root:

```
Q = P + H_TapTweak(P || merkle_root) · G
```

The 32 B output pubkey `Q` already commits to `merkle_root`. The same
construction works for any 32 B commitment: replace `merkle_root` with
the PKI Merkle root, and the Taproot output key now commits to the
PKI data for **zero marginal on-chain bytes**.

Cost: nothing. The 32 B output pubkey existed anyway. The PKI root
lives entirely in off-chain data (Nostr expansion event per §1.2),
and verifiers recompute `Q` from the revealed root and the internal
key to confirm the commitment.

Trade-off: key-path spending without revealing the commitment becomes
impossible (any spend that bypasses the commitment is unverifiable),
which is exactly what we want — the whole point of an on-chain
commitment is that it must be revealed.

[bip341]: https://github.com/bitcoin/bips/blob/master/bip-0341.mediawiki

### 2.4 Tier 2 — P2MR (BIP-345 family, future)

[P2MR / Pay-to-Merkle-Root][p2mr] generalises Taproot's script-path:
multiple script branches committed under one Merkle root, reveal only
the branch being spent. **Not currently active on Bitcoin.**

Marginal benefit over Tier 1b for our use case: lets a single UTXO
pre-commit to many PKI variants (e.g. current production cert,
future rollover cert, emergency revocation cert) at zero on-chain
cost until one of them needs to be revealed. Useful for rotation
schemes that today would require multiple `name_update` transactions.

For the single-PKI-leaf case, Tier 1b already gets the byte cost to
zero, so P2MR is a *future bonus* and not on the critical path.

### 2.5 Namecoin-specific caveats

These are the real blockers, and they are not byte-budget questions
— they are consensus-rule and resolver-indexing questions.

**Where does the Namecoin name-value live?** Today:

```
OP_NAME_UPDATE <name> <value> OP_2DROP OP_DROP <normal P2*>
```

The `<value>` is part of the scriptPubKey, not the witness. Moving
the value's bytes (or just the canonical commitment) to a witness
location requires either:

- (a) A consensus-rule change: Namecoin's name-update validation
  reads the value from witness data. Soft-fork-shaped, needs
  JeremyRand's design input.
- (b) A resolver convention: the scriptPubKey still carries a small
  field (a tag + the 32 B commitment in the existing value, or the
  Taproot-tweaked output key with a sentinel value), and the
  canonical content lives in a Nostr expansion event verifiable
  against the on-chain commitment. **Implementable today, no
  consensus change.**

Option (b) is the natural first step. Option (a) is a later
optimization once the resolver-convention path is proven.

**ElectrumX-NMC indexing.** Today ElectrumX-NMC resolves a name by
reading the latest UTXO's scriptPubKey from the UTXO set. If the
canonical content lives in witness data, indexers must additionally
fetch and parse the spending transaction's witness. Cheap per-lookup
(one tx fetch per name), but a different indexing shape and incompatible
with witness-pruned nodes. **Worth confirming with the existing
[`testls.bit` / `relay.testls.bit`][testls] resolver path before any
spec PR proposes witness-borne values.**

[testls]: https://relay.testls.bit/

**520-byte cap.** The current consensus rule caps the
scriptPubKey-side value at 520 B (see
[ifa-0001][ifa-0001] §"Size limits" and the [FAQ entry][nmco-faq]).
If value bytes ever move to witness, does the cap apply to the
witness data, the scriptPubKey, or both? This is a design decision
with DoS-surface implications. Not free.

[ifa-0001]: https://github.com/namecoin/proposals/blob/master/ifa-0001.md
[nmco-faq]: https://www.namecoin.org/docs/faq/#what-applications-is-namecoin-well-suited-to

----

Layer 3 — Distribution and discovery of Merkle expansions across relays
-----------------------------------------------------------------------

The on-chain commitment doesn't help anyone if clients can't *find*
the expansion event. This layer is about leveraging existing Nostr
discovery primitives so that a `.bit`-tagged event spreads naturally
across the relay network without bespoke infrastructure.

### 3.1 Single-letter indexed tags (NIP-01 §"Tags")

[NIP-01][nip-01] §"Tags" mandates that **single-letter (a-zA-Z) tag
names are indexed by relays**. Any of the following filter shapes
will work on every spec-compliant relay today:

```json
{"#d": ["example.bit"]}        // exact addressable identifier
{"#t": ["example.bit"]}        // hashtag (NIP-24)
{"#i": ["nmc:d/example"]}      // external content id (NIP-73)
```

No new NIP work is required to *use* these — the tag-index slots
already exist. The only design question is **which tag(s) the
expansion event SHOULD carry**, and the answer is "all three", for
different audiences:

- `d` for canonical lookup by clients that know the .bit name.
- `t` for ambient discovery in regular hashtag feeds — e.g. a client
  showing the `#example.bit` feed sees every event that references
  the name, including the expansion event, replies to the
  expansion event, attestations, and so on. This is what M wanted
  when asking "can I invent a hashtag that embeds a `.bit` domain?"
  — the answer is *yes, and you don't even need to invent anything*.
- `i` for cross-system indexers (NIP-73) that catalog references
  to external systems.

### 3.2 Proposed NIP-73 `nmc:` external-content-id prefix

[NIP-73][nip-73] defines the `i` tag for external content identifiers
with a `<system>:<id>` shape. The existing registry includes
`isbn:`, `doi:`, `geohash:`, `podcast:guid:`, etc. Adding `nmc:` is
a single-line addition to the NIP-73 table:

| system | format | example |
|---|---|---|
| `nmc` | `nmc:<namespace>/<name>` | `nmc:d/example`, `nmc:id/alice` |

This is **independent of every other section in this document** —
it can land as its own one-line PR to `nostr-protocol/nips`
without any of the Merkle scheme being settled. It immediately
makes any Nostr event referencing a Namecoin name discoverable by
every relay that indexes NIP-73.

Recommended sequencing: open this as a standalone PR first, because
(a) it's tiny, (b) it unblocks the discovery story for the entire
Namecoin track, and (c) it doesn't pre-commit any of the more
contentious design choices.

### 3.3 NIP-50 search (optional)

[NIP-50][nip-50] defines a `search` filter on REQ messages for
relays that support full-text search. Expansion events carrying
`.bit` names in their `content` or `tags` are searchable through
this channel where it's supported, but `#d`/`#t`/`#i` index filters
should be the primary discovery path (search support varies; tag
indexing is mandatory).

[nip-50]: https://github.com/nostr-protocol/nips/blob/master/50.md

### 3.4 NIP-65 outbox model interaction

[NIP-65][nip-65] (Relay List Metadata) describes the "outbox model"
where clients learn which relays an author writes to from their
`kind:10002` event. An expansion event for `example.bit` SHOULD be
published to:

- the author's NIP-65 write relays (so followers find it through
  the normal outbox flow);
- the relay set listed in the name's `nostr.relays` item (so clients
  resolving `example.bit` natively can find it without any prior
  knowledge of the author);
- a few well-known general-purpose public relays (so the event is
  recoverable in worst-case bootstrap scenarios).

For `.bit`-anchored relays (NIP-N2), publishing the expansion event
to the same `.bit` relay closes the loop: the relay's TLS
fingerprint is itself pinned in the expansion, so a client that has
the on-chain root can verify the TLS handshake of the very relay
that's serving the expansion.

[nip-65]: https://github.com/nostr-protocol/nips/blob/master/65.md

### 3.5 NIP-77 Negentropy syncing

[NIP-77][nip-77] (Negentropy Syncing) lets relays efficiently
reconcile their event sets. Once expansion events are in
circulation, relays that support Negentropy will naturally
gossip them around without any explicit publication push. This
matters because Namecoin names change rarely (typical
`name_update` cadence is months to years), so the expansion-event
set is small, mostly-static, and ideal for Negentropy-style sync.

[nip-77]: https://github.com/nostr-protocol/nips/blob/master/77.md

### 3.6 NIP-66 relay discovery

[NIP-66][nip-66] (Relay Discovery) defines a `kind:30166` event
that announces relay liveness. A `.bit`-anchored relay that wants
ambient discoverability SHOULD publish a `kind:30166` carrying
its `.bit` URL (NIP-N2 form) and tag it `["t", "<name>.bit"]`
so it shows up in `#<name>.bit` feeds as a relay record, not
just as note content.

[nip-66]: https://github.com/nostr-protocol/nips/blob/master/66.md

----

How this interacts with existing NIPs in this repo
--------------------------------------------------

- **N0 (record container).** The Merkle root would be a new
  top-level field in the Namecoin record alongside (not replacing)
  the existing `nostr`, `tls`, `import`, `map` items. Specific
  field name TBD. The plaintext `nostr` item stays the canonical
  form for Tier 0 deployments; the Merkle scheme is an
  alternative encoding for byte-pressed names.
- **N1 (NIP-05 verification).** Verification flow extends with a
  fallback: if the Namecoin name carries a Merkle commitment
  rather than a plaintext `nostr.names`, the client first fetches
  the expansion event (§1.2) and then proceeds with NIP-N1
  verification against the materialised names map.
- **N2 (relay discovery).** Same as N1: relay URLs from the
  expansion event's `nostr.relays` are treated identically to
  relay URLs from a plaintext record.
- **N3 (TLSA pinning).** Currently consumes the 32 B hashed-TLSA
  primitive from PR #747. Under the Merkle scheme, the TLSA hash
  becomes a leaf; the verification flow is unchanged once the
  client has the materialised leaf.
- **N4 (service attestations).** Attestations could optionally
  reference the Merkle root in their `proof` block, so the
  attestation is bound to a specific snapshot of the name's PKI
  state. Out of scope until N4 itself stabilises.
- **N5 (CLINK).** The biggest beneficiary. CLINK's `noffer1...` /
  `ndebit1...` pointers are bech32 strings that can easily exceed
  the remaining 520 B budget for a name that already carries TLSA,
  relays, and a names map. Moving them into the expansion event
  (as `nostr.clink_offer` / `nostr.clink_debit` leaves under the
  Merkle root) eliminates the need for the `dd/`/`d/cl-` factoring
  pattern documented in N5 §"520-byte budget". A wallet's discovery
  flow becomes: read 32 B from chain → fetch expansion from any
  relay → verify root → use the `clink_offer` / `clink_debit`
  pointers.

----

Sequencing — what would actually happen, in order
-------------------------------------------------

Nothing in this file should be opened as a PR yet. The right order
of operations, once the existing namecoin.org PRs (#755–#760) have
landed or received clear feedback, is:

1. **Issue (not PR) on `namecoin/namecoin.org`** titled something
   like "Merkle commitment for multi-record name values". Layer 1
   only. Asks JeremyRand for design input on tagged-hash strings,
   tree construction, wildcard/inheritance, and where the
   commitment field name should live in the record schema. No code,
   no doc patch — just framing.
2. **Standalone PR to `nostr-protocol/nips`** adding `nmc:` to the
   NIP-73 external-content-id registry. Tiny, independent, useful
   regardless of whether the Merkle scheme lands. Unblocks
   §3.2 discovery for the entire Namecoin track. (Could happen
   in parallel with step 1.)
3. **Reference deployment.** Publish a Merkle-format expansion
   event for `testls.bit` (or a sibling name like `merkle.testls.bit`)
   while leaving the production `d/testls` plaintext record
   untouched. Prove the verification flow end-to-end before
   proposing any spec change.
4. **NIP draft in this repo** as `N6.md` (or wherever fits the
   numbering at that time), once steps 1–3 have produced enough
   design consensus to write a normative spec. At that point this
   file gets pruned of the sections that become normative and kept
   only for the genuinely-future bits (Layer 2b / P2MR, witness-
   borne values).
5. **CLINK follow-up** (`shocknet/CLINK#6`): once the Merkle
   expansion event is specified, the CLINK issue answer becomes
   "use the Namecoin discovery flow defined in NIP-N6, which
   carries clink_offer/clink_debit in the expansion event" —
   no HTTPS, no CA, ~0 marginal on-chain bytes (under §2.3).
6. **(Deferred) Witness-borne value spec.** Only after
   §2.5's resolver-indexing questions are resolved with
   JeremyRand. Almost certainly a Namecoin consensus discussion,
   not a Nostr discussion. May never need to happen — Tier 1b
   (Taproot tweak) already gets the marginal byte cost to zero.

----

Open design questions to settle before any of this becomes a NIP
---------------------------------------------------------------

These are flagged for future-me. Do not pick answers in this file —
that's the spec's job, once the upstream conversations are open.

1. **Tagged-hash tag strings.** Lock them down at spec time;
   changing later silently invalidates every prior commitment.
   Current candidates: `nmc/tlsa`, `nmc/sshfp`, `nmc/nostr`,
   `nmc/import`. Naming convention is BIP-340-shaped.
2. **Merkle tree construction.** Options: RFC 6962 (CT log style,
   prepends `0x00` / `0x01` for leaf / internal), BIP-340 tagged
   construction, sorted-by-tag with power-of-2 padding. The choice
   affects whether two implementations with the same leaf set
   always agree on the root.
3. **Leaf-set extensibility.** Adding a new leaf type later (e.g.
   a future post-quantum cert hash) MUST not invalidate existing
   roots. "New leaf type = new tag = unaffected sibling hashes"
   works if and only if the tree construction tolerates missing
   leaf slots. Possibly a Sparse Merkle Tree is needed instead.
4. **Wildcard inheritance.** Per-leaf inheritance vs. mandate
   `d/`+`dd/` factoring (see §1.4). Affects every name that uses
   `map`.
5. **Salt policy.** Per-leaf salts vs. global salt vs. no salts.
   Probably none of the currently-defined leaves need salts, but
   the field shape should be specified now so it can be added
   later without a v2 root format.
6. **Field name on the Namecoin side.** `merkle`, `mr`, `pki`,
   `root`, or sitting under the existing `tls` item with a
   discriminator. Affects every existing resolver.
7. **Backwards compatibility.** Must coexist with plaintext
   `nostr`/`tls`/etc. records during the transition. Probably the
   resolver looks for the Merkle root first and falls back to
   plaintext if absent, but that's a spec decision.
8. **Resolver pruning behaviour.** ElectrumX-NMC behaviour when
   the expansion event isn't found on any relay the client knows
   about. Hard fail? Soft fail with a typed error? Cache last-seen
   expansion?
9. **Tweaked-key encoding on the Namecoin side.** If the commitment
   rides on the Taproot output key (Tier 1b), how does the
   `name_update` value reference it without duplicating the 32 B
   on-chain? Possibly a single-byte sentinel meaning "the
   commitment is the output's Taproot internal key".

----

Lessons captured while drafting this
------------------------------------

- "32 B on chain" undersells Taproot. The actual floor is 0 B
  marginal via output-key tweaking. Worth saying out loud every
  time someone is optimizing on-chain byte budgets.
- Logical compression (Layer 1) and byte-slot relocation (Layer 2)
  operate on different axes and compose multiplicatively. Don't
  conflate them in spec PRs — one PR per axis.
- Nostr hashtag-style discovery (`#t`, `#d`, `#i`) works today on
  every NIP-01-compliant relay. No NIP changes are required to
  use it; only to standardise *which* tag values are used for a
  given application. The `nmc:` NIP-73 prefix is the smallest
  useful spec PR in the entire stack.
- The blocker for witness-borne Namecoin values is the resolver
  indexing model, not the consensus rule. Address ElectrumX-NMC
  before pitching any consensus change.
- Hashed TLSA (PR #747) is the existence proof that JeremyRand
  accepts the "32 B on chain + reveal at handshake" pattern.
  The Merkle scheme is a natural extension of that pattern, not
  a new direction.
