# Changelog

All notable changes to multiformats-nv are recorded here. The format is
[Keep a Changelog](https://keepachangelog.com/en/1.1.0/), and this
package follows [Semantic Versioning](https://semver.org/spec/v2.0.0.html)
with the pre-1.0 rule that a breaking change bumps the MINOR number.

## [0.0.1]

**The interface, published before anyone implements it.** Every public
type and function carries its full signature, its effect row and its
doc comment; every body is `todo()`; the release is recorded
`implemented = false`.

### Added

- `mfhash` — the load-bearing interface, and it is an ABSENCE. A
  multihash here is a function code, a digest length and a digest the
  CALLER computed; no hash function is in this package and none is
  coming. The consequence is what the decision is for: a program that
  parses, compares, re-encodes and routes content identifiers links no
  hash function at all, where a package that hashed would have had to
  depend on SHA-2, SHA-3 and BLAKE2 to cover the codes in ordinary use
  and put all three in every consumer that never verifies anything.
  `matches` is the whole of verification — a comparison — and
  `expected_length` is what lets the shape be checked without computing
  anything, so a `sha2-256` multihash with a twenty-byte digest is
  refused at construction rather than at the comparison that never
  succeeds. The README names crypto-nv, sha3-nv and blake2-nv against
  the codes each of them answers.
- `mfcodec` — an unknown code is CARRIED, not refused. The multicodec
  table grows, and an identifier whose codec this build has no name for
  is still valid: its bytes are well formed, it compares equal to
  itself, it re-encodes into another base, and a program routing it has
  no need to know what it points at. A package that refused one would
  have to be re-released every time the table grew. So `MfCodec` is the
  number, `name_of` may answer `""`, and `MfUnknownCodec` is
  deliberately not a fault.
- `mfcid` — one type with a version field, and the asymmetry in the
  conversions. Every version 0 identifier has a version 1 form; a
  version 1 identifier has a version 0 form only when its codec is
  dag-pb and its hash sha2-256, and `to_v0` refuses by name rather than
  producing an identifier for different data. Equality is over the
  BYTES: the same identifier in two bases is one identifier, and a
  version 0 identifier and its version 1 form are two.
- `mfbase` — twenty-one rows of the table, with case as part of the
  prefix rather than a flag, because a reader that lowercased a string
  before decoding would turn a `B` string into a `b` string whose
  payload no longer decodes. `is_big_integer_base` is the distinction
  that decides what a leading zero byte does.
- `mfvarint` — the nine-byte cap and the shortest-spelling rule as
  refusals, not tolerances. Two spellings of one number are two
  addresses for one object, which is the property the whole family
  exists to prevent.
- `mferror` — fourteen reasons with the byte offset each was found at,
  shared by all four formats because they nest.

### Known

- `novo test` is red, and that is the release's expected state: every
  assertion in the API suite reaches `not implemented:
  multiformats-nv.<module>.<fn>`.
- **The four fixture files are named but not generated.** The suite
  carries cases taken from them and from the specifications' worked
  examples; the generated run lands with the implementation.
- **multiaddr is not here**, and is a package of its own: its table, its
  text form and its tunnelling rules are as large as everything in this
  one put together.
- **No `tests/embedded_probe.nv`.** The varint and multihash halves
  would build for a device — they are byte arithmetic over a buffer —
  but the multibase alphabets and the codec name table are owned
  strings, and a claim covering half a package would have to be
  qualified in every sentence that stated it. Splitting the device half
  out is a decision for the implementation.
