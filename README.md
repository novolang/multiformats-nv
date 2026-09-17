# multiformats-nv

The multiformats are four small self-describing formats used together by
IPFS, libp2p and the systems built on them:
[multibase](https://github.com/multiformats/multibase),
[multicodec](https://github.com/multiformats/multicodec),
[multihash](https://github.com/multiformats/multihash) and the
[content identifier](https://github.com/multiformats/cid), or CID. Each
of them makes a value say what it is, so that a program reading it does
not have to be told separately. This package reads and writes all four
in novo-lang.

**Status: NOT IMPLEMENTED — interface only.** Every function is declared
with its full signature, but every body is a `todo()` that panics when
called. The package is published so its design can be reviewed and
depended on before it is implemented. Version 0.1.0 will be the first
working release.

## What the four formats are

**A varint** is the number encoding all four are built on: seven bits of
the number per byte, least significant first, with the high bit set on
every byte but the last. The multiformats specification adds two rules
to it. At most nine bytes, so a varint holds sixty-three bits. And the
shortest spelling only — because a value that has two spellings is an
address that names one object twice.

**Multibase** puts the encoding in front of the string. A base-encoded
string does not say which base it is in, and `1111` is a valid base16,
base32, base58 and base64 string with four different meanings. So the
first character names the alphabet: `z` is base58btc, `b` is base32, `f`
is base16, `u` is base64url, `m` is base64. Case is part of the prefix:
`b` is lowercase base32 and `B` is uppercase.

**Multicodec** is a varint at the front of a buffer saying what the rest
of it is. `0x55` is raw bytes, `0x70` is dag-pb, `0x71` is dag-cbor,
`0x12` is a SHA-256 digest, `0xed` is an Ed25519 public key. The table
is a file in the multiformats repository with several hundred rows, and
it grows.

**A multihash** is three things: the multicodec of the hash function,
the digest's length in bytes, and the digest. A SHA-256 multihash is
`0x12 0x20` and then thirty-two bytes. It exists because a bare digest
does not say what made it, so a system that stored one could never move
to another hash function without reinterpreting everything it had
stored.

**A content identifier** at version 1 is a version varint, a multicodec
saying what the data is, and a multihash of that data. Written as text
it takes a multibase prefix. At version 0 it is none of that: it is a
bare SHA-256 multihash, the data is always dag-pb, and it is always
written in base58btc with no prefix — which is why every version 0
identifier begins `Qm`.

## No hash function is in this package

A multihash here is a code, a length and a digest **the caller
computed**. Nothing in this package hashes anything.

The consequence is the reason for it. A program that parses, compares,
re-encodes and routes content identifiers links no hash function at
all. A package that hashed would have had to depend on SHA-2, SHA-3 and
BLAKE2 to cover the codes in ordinary use, and every consumer would
carry all three to read an identifier it never verifies.

A caller that does verify computes the digest itself and calls
`mfhash.matches`.

| Code | Function | Which package computes it |
| --- | --- | --- |
| `0x12` | sha2-256 | [crypto-nv](https://novo-lang.org/packages/crypto-nv) |
| `0x13` | sha2-512 | crypto-nv |
| `0x16` | sha3-256 | [sha3-nv](https://novo-lang.org/packages/sha3-nv) |
| `0x14`–`0x17` | sha3-512, sha3-384, sha3-256, sha3-224 | sha3-nv |
| `0x18`, `0x19` | shake-128, shake-256 | sha3-nv |
| `0xb220` | blake2b-256 | [blake2-nv](https://novo-lang.org/packages/blake2-nv) |
| `0xb200 + n` | blake2b with an *n*-byte digest | blake2-nv |
| `0x00` | identity — the digest is the data | nothing; no hashing happens |

`mfhash.expected_length` knows how long each of those digests is, so a
multihash claiming a twenty-byte SHA-256 is refused here rather than at
the comparison.

## Install

```
novo pkg add multiformats-nv
```

## Example

```novo
use std.bytes
use mfbase
use mfcodec
use mfhash
use mfcid

fn main() [io]
    // A digest the caller computed with crypto-nv, sha3-nv or blake2-nv.
    // Nothing in this package hashes anything.
    let digest = bytes.zeros(32)

    match mfhash.multihash(mfcodec.sha2_256(), digest)
        Err(e) => println("not a digest of that function: ${e.message()}")
        Ok(hash) =>
            // A version 1 identifier for raw bytes.
            let cid = mfcid.cid_v1(mfcodec.raw(), hash)

            // The base is chosen at the moment of writing, not stored.
            match mfcid.to_string(cid, MfBase32)
                Err(e) => println(e.message())
                Ok(s)  => println(s)

            // What it is, for a person.
            println(mfcid.describe(cid))

    // A string somebody pasted. Its first character says how to read it.
    match mfbase.decode_tagged("f00")
        Err(e) => println(e.message())
        Ok(d)  => println("it was ${mfbase.name_of(d.base)}")
```

Build and test with `novo pkg build` and `novo test`. Today `novo test`
fails on purpose: every test reaches a
`not implemented: multiformats-nv.<module>.<fn>` panic. The tests are
the specification the implementation will have to satisfy.

## What the package contains

| Module | Contents |
| --- | --- |
| `mferror` | Every refusal in the four formats, with the byte offset it was found at. |
| `mfvarint` | The number encoding all four are built on, with its nine-byte cap and its shortest-spelling rule. |
| `mfbase` | The multibase table: twenty-one alphabets, their prefixes and their names, and encoding both ways. |
| `mfcodec` | Multicodec codes, the names this package knows for them, and the ones it does not. |
| `mfhash` | A multihash as a code, a length and a digest the caller supplied, and the comparison that verifies one. |
| `mfcid` | The content identifier in both versions, the conversion between them, and equality over the bytes. |

## How to choose an entry point

**`mfcid.parse` takes the string a person pasted.** It handles both
versions: a forty-six-character string beginning `Qm` is version 0, and
anything else is read as multibase.

**`mfcid.to_default_string` writes it back the way the IPFS tools do**
— base58btc at version 0, base32 at version 1. `mfcid.to_string` is for
a caller that wants a particular base.

**`mfhash.matches` is the whole of verification.** Hash the data with
whichever package answers the code, and hand the digest over.

**`mfcid.check` validates an identifier without the data**: the version,
version 0's fixed codec and hash, the codec's place in the table, and
the digest's length.

**`mfbase.decode_tagged` says which base a string was in**, which is
what a tool that re-encodes an identifier needs.

## The rules a user needs

1. **A varint is at most nine bytes**, so it holds sixty-three bits. The
   cap lets a parser bound its work before it starts.
2. **A varint must be its shortest spelling.** `0x81 0x00` and `0x01`
   are the same number, and a format whose values are addresses cannot
   have two spellings of one address.
3. **Case is part of a multibase prefix.** `b` is lowercase base32 and
   `B` is uppercase. Lowercasing a string before decoding it turns one
   into the other and destroys the payload.
4. **The identity multibase is not text.** Its prefix is a NUL byte, so
   `mfbase.encode` refuses it and `mfbase.encode_bytes` is the form
   that holds it.
5. **base58, base36 and base10 are big-integer conversions, not bit
   regroupings.** A leading zero byte has to be written as a separate
   leading digit, which is why a base58btc string decoding to bytes
   beginning `0x00` begins with `1`.
   `mfbase.is_big_integer_base` says which bases those are.
6. **An unknown multicodec is carried, not refused.** The table grows.
   An identifier whose codec this build has no name for is still valid,
   compares equal to itself and re-encodes; `mfcodec.name_of` answers
   `""` and `mfcodec.is_known` says so.
7. **The blake2b codes are a range.** `0xb200` plus the digest length in
   bytes, so blake2b-256 is `0xb220`.
   `mfcodec.blake2b_of_length` is that arithmetic.
8. **A multihash's digest length is checked against its function**, for
   the functions this package knows. A code it has no entry for accepts
   any length, and so does the identity code, whose digest is the data.
9. **Version 0 is dag-pb and sha2-256 and nothing else.** It has no
   version varint, no codec varint and no multibase prefix.
10. **Every version 0 identifier converts up; only some version 1
    identifiers convert down.** `mfcid.to_v0` answers
    `MfNotV0Convertible` naming which rule failed, because producing
    one anyway would produce an identifier for different data.
11. **A version 0 identifier can only be written in base58btc.** There
    is no prefix in which to record that it was written otherwise, so
    `mfcid.to_string` with any other base is `MfNoSuchForm`.
12. **Equality is over the bytes, not the text.** The same identifier in
    base32 and in base58btc is two strings and one identifier. A
    version 0 identifier and its version 1 form are **not** equal —
    they are different bytes — and `mfcid.equals_any_version` is the
    comparison for a store that holds both.
13. **Thirty-four bytes beginning `0x12 0x20` is a version 0
    identifier.** That special case is the specification's, and it is
    why version 1 could not use `0x12` as its version number.
14. **A CID's codec should be an `ipld` or `serialization` entry and the
    code inside its multihash a `multihash` one.**
    `mfcodec.tag_of` answers the table's own column, and `mfcid.check`
    applies it.

## What is not included

- **Every hash function.** See the section above. A digest arrives as a
  parameter.
- **multiaddr.** It is a fifth multiformat — a network address written
  as a sequence of multicodec-tagged components — and it is a package of
  its own, because its table, its text form and its tunnelling rules
  are as large as everything here put together.
- **The whole multicodec table.** This package names the subset a
  content identifier, a peer identifier and a cryptographic key use.
  Every other code is carried by its number, and
  `mfcodec.known_codecs` prints what is here.
- **base1 and the emoji alphabet**, two rows of the multibase table that
  nothing in practice writes.
- **IPLD.** Knowing that a `0x71` block is dag-cbor is this package's
  job; decoding it is [cbor-nv](https://novo-lang.org/packages/cbor-nv)'s.
- **A block store, and the network.** Fetching the data an identifier
  names costs `[net]`, and this package declares no effects.
- **The `NUL`-prefixed identity multibase inside a CID**, which some
  tools emit for an inline identifier. It parses, and this package
  writes base32 or base58btc.

## Related packages

- [crypto-nv](https://novo-lang.org/packages/crypto-nv),
  [sha3-nv](https://novo-lang.org/packages/sha3-nv) and
  [blake2-nv](https://novo-lang.org/packages/blake2-nv) compute the
  digests a multihash carries. See the table above for which answers
  which code.
- [cbor-nv](https://novo-lang.org/packages/cbor-nv) decodes a dag-cbor
  block once a CID has said that is what it is.
- [base64-nv](https://novo-lang.org/packages/base64-nv) and
  [bech32-nv](https://novo-lang.org/packages/bech32-nv) are the other
  two text encodings on this registry; neither is a multibase alphabet.
- [bigint-nv](https://novo-lang.org/packages/bigint-nv) is the
  arithmetic a big-integer base is, for a caller that wants to do the
  conversion itself.

## Test vectors

The normative sources are the four specifications in the multiformats
organisation: the multibase table, the multicodec `table.csv`, the
multihash specification and the CID specification.

Each of those repositories ships its own fixtures, and together they
are what every implementation is measured against: the multibase
repository holds the same bytes encoded in every base in the table, the
multihash repository a table of encoded digests, and the CID repository
a list of identifiers with their version, codec, hash and both text
forms. The cases in this package's suite are taken from them and from
the worked examples in the specifications, and the generated run over
the fixture files lands with the implementation.

```bash
novo test tests/multiformats_tests.nv    # the varint, the four formats
```

The suite asserts that a varint longer than nine bytes and a varint
written longer than it needs are both refused, that each alphabet is
the length its name says, that case is part of the prefix, that an
unknown codec is carried and named `""`, that a sha2-256 multihash with
a twenty-byte digest is refused, that a version 0 identifier is
thirty-four bytes and begins `Qm`, that a raw-codec identifier has no
version 0 form, and that a version 0 identifier and its version 1 form
are not equal.

The tests compile today and fail at run, each on the
`not implemented: multiformats-nv.<module>.<fn>` panic that is its
body. That is the expected state of an interface release. They turn
green one at a time as bodies land.

## Implementation status

| Item | Implemented |
| --- | --- |
| `mfbase.MfBase`, `mfcodec.MfCodec`, `mfhash.MfMultihash`, `mfcid.MfCid` and the other types | the types are declared |
| `mferror.offset_of`, `.code_of`, `MfFault.message` | no |
| `mfvarint.encoded_len`, `.encode`, `.check`, `.decode`, `.decode_all` | no |
| `mfbase.prefix_of`, `.base_of_prefix`, `.name_of`, `.base_of_name`, `.alphabet_of` | no |
| `mfbase.is_padded`, `.is_big_integer_base` | no |
| `mfbase.encode`, `.encode_bytes`, `.decode`, `.decode_tagged`, `.base_of` | no |
| `mfcodec.codec`, `.name_of`, `.is_known`, `.codec_of_name`, `.known_codecs`, `.tag_of` | no |
| `mfcodec.identity`, `.raw`, `.dag_pb`, `.dag_cbor`, `.dag_json`, `.sha2_256`, `.sha2_512`, `.sha3_256`, `.blake3`, `.blake2b_256`, `.blake2b_of_length`, `.ed25519_pub` | no |
| `mfcodec.to_bytes`, `.read` | no |
| `mfhash.multihash`, `.expected_length`, `.identity_multihash`, `.check` | no |
| `mfhash.to_bytes`, `.encoded_len`, `.from_bytes`, `.read`, `.matches`, `.describe` | no |
| `mfcid.cid_v1`, `.cid_v0`, `.to_bytes`, `.from_bytes`, `.read` | no |
| `mfcid.to_string`, `.to_default_string`, `.parse`, `.describe` | no |
| `mfcid.to_v1`, `.to_v0`, `.is_v0_convertible`, `.equals`, `.equals_any_version`, `.check` | no |

## Licence

Apache-2.0. See `LICENSE`.

<!-- docs/writing-a-readme.md is the style guide for this page. -->
