# NIPs — Namecoin Track

A small set of Nostr Implementation Possibilities that anchor parts of the Nostr
identity / relay stack in the Namecoin blockchain, so they survive DNS-level
takedowns, public-CA compromise, and registrar coercion.

These are **proposals**. They are written in the same style as
[`nostr-protocol/nips`](https://github.com/nostr-protocol/nips) and are
deliberately separable: a client can implement any single one and still
interoperate with classic NIP-05, NIP-65, etc.

All four NIPs are wire-compatible with the Namecoin project's
[`namecoin/proposals`](https://github.com/namecoin/proposals)
(in particular [ifa-0001](https://github.com/namecoin/proposals/blob/master/ifa-0001.md)
"Domain Names",
[ifa-0002](https://github.com/namecoin/proposals/blob/master/ifa-0002.md)
"Domain Names — Non-DNS Types", and
[ifa-0003](https://github.com/namecoin/proposals/blob/master/ifa-0003.md)
"Domain Names — Dehydrated TLS Certificates"). They re-use existing fields
(`map`, `import`, `tor`, `tls`, `_tor.txt`) wherever possible and add
**zero new top-level keys** to the Namecoin record schema. Everything Nostr-specific
lives under the `nostr` item.

## Overview

| NIP | Title | Status |
|---|---|---|
| [B0](B0.md) | Namecoin record container for Nostr (`nostr` item) | Draft |
| [B1](B1.md) | NIP-05 verification via Namecoin (`.bit` / `d/` / `id/`) | Draft |
| [B2](B2.md) | Nostr relay discovery, subdomains, and Tor routing via Namecoin | Draft |
| [B3](B3.md) | TLSA pinning for `.bit` Nostr relay WebSockets (RFC 6698) | Draft |

The `B` prefix avoids stepping on the upstream NIP numbering (which is allocated by
PR-merge order in `nostr-protocol/nips`). If any of these are upstreamed, the
upstream number takes precedence and the `B*` file becomes a redirect.

## Reference implementations

The four NIPs are each implemented in production code today. References:

- **Amethyst** (Android / KMP iOS / Desktop, Kotlin):
  - NIP-05 over Namecoin: [PR #1734](https://github.com/vitorpamplona/amethyst/pull/1734)
    (merged), [PR #1771](https://github.com/vitorpamplona/amethyst/pull/1771)
    (race-condition / cancellation fixes, merged),
    [PR #1786](https://github.com/vitorpamplona/amethyst/pull/1786) (custom
    ElectrumX servers, merged),
    [PR #1785](https://github.com/vitorpamplona/amethyst/pull/1785) (import
    follow list, merged),
    [PR #1806](https://github.com/vitorpamplona/amethyst/pull/1806) /
    [#1815](https://github.com/vitorpamplona/amethyst/pull/1815)
    (settings + explorer, merged),
    [PR #1937](https://github.com/vitorpamplona/amethyst/pull/1937) (Samsung
    One UI 7 ElectrumX TLS pinning, merged).
  - Bare and `@`-prefixed `.bit` identifiers in note content:
    [PR #1915](https://github.com/vitorpamplona/amethyst/pull/1915) (closed,
    feature complete pending review).
  - iOS port: [PR #2199](https://github.com/vitorpamplona/amethyst/pull/2199).
  - Desktop port: [PR #1914](https://github.com/vitorpamplona/amethyst/pull/1914).
  - `.bit` relay resolution + TLSA + Tor:
    [PR #2595](https://github.com/vitorpamplona/amethyst/pull/2595).
  - `import` item resolution (ifa-0001 §"import"):
    [PR #2612](https://github.com/vitorpamplona/amethyst/pull/2612)
    (full stack), [PR #2707](https://github.com/vitorpamplona/amethyst/pull/2707)
    (NIP-05-only carve-out so it can land independently).
- **Nostur** (iOS, Swift): [PR #60](https://github.com/nostur-com/nostur-ios-public/pull/60).
- **nostr-tools** (TypeScript / Node, isomorphic): [PR #533](https://github.com/nbd-wtf/nostr-tools/pull/533).
- **strfry** (relay write-policy plugin, C++/JS): https://github.com/mstrofnone/strfry-namecoin-policy.
- **strfry NIP-05 sidecar**: https://github.com/mstrofnone/strfry-nip05-namecoin.
- **nak-nmc** (CLI front-end for [`nak`](https://github.com/fiatjaf/nak)): https://github.com/mstrofnone/nak-nmc.
- **nostrlib-nip05-namecoin** (Go, fiatjaf/nostr-lib subpackage): https://github.com/mstrofnone/nostrlib-nip05-namecoin.
- **nmcLightningService** (Node, end-to-end on-chain publishing pipeline): https://github.com/mstrofnone/nmcLightningService.

A live reference deployment (relay + identity + TLSA + Tor) sits at
`relay.testls.bit` / `testls.bit` and is exercised by the test suites of every
implementation listed above.

## Conventions

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD",
"SHOULD NOT", "RECOMMENDED", "MAY", and "OPTIONAL" in these documents are to be
interpreted as described in [RFC 2119](https://www.rfc-editor.org/rfc/rfc2119).

These documents are released into the public domain (CC0-1.0).
