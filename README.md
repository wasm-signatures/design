# WebAssembly modules signatures

<!-- @import "[TOC]" {cmd="toc" depthFrom=2 depthTo=6 orderedList=false} -->

<!-- code_chunk_output -->

- [WebAssembly modules signatures](#webassembly-modules-signatures)
  - [Project scope](#project-scope)
  - [Requirements and justifications](#requirements-and-justifications)
  - [Discussed options](#discussed-options)
  - [Scratch notes](#scratch-notes)
  - [Appendix 1](#appendix-1)
  - [Appendix 2](#appendix-2)
  - [Appendix 3](#appendix-3)

<!-- /code_chunk_output -->

## Project scope

This work is specifically about embedded digital signatures in WebAssembly modules, not about package/OCI signatures.

The goal is to converge on requirements that can be used to build out an initial and extensible specification.

Existing projects:

- [WasmSign](https://github.com/jedisct1/wasmsign) - a tool to embed signatures in WebAssembly modules
- [wasm-sign](https://github.com/frehberg/wasm-sign) - another tool to sign WebAssembly modules
- [Istio/Envoy example](https://github.com/proxy-wasm/proxy-wasm-cpp-host/pull/147) load-time check of WasmSign signatures
- [Validation in Lucet](https://bytecodealliance.github.io/lucet/Integrity-and-authentication.html)
- [WAPM package manager](https://medium.com/wasmer/securing-wapm-packages-with-package-signing-3cf0d12f32f3) signature verification

Related:

- [Sigstore](https://sigstore.dev) transparency log

## Requirements and justifications

- [X] Signatures may implement multiple signature algorithms
  - Compliance requirements
  - Public key size/signature size/verification CPU cost tradeoffs (ex: post-quantum schemes vs EdDSA)
  - Code and key reuse (ex: ECDSA-SHA3 for blockchains already leveraging SHA3, EdDSA to use existing SSH keys)

- [X] It should be possible to verify a module file before execution

- [X] It should be possible to add signed custom sections to a potentially signed module so that one can verify the original module and the additional custom section independently

  See [Appendix 1](#appendix-1). A user may want to add additional signed information (debug data, precompiled headers) to a signed module and then re-distribute. Users may or may not trust and choose to consume the additional information.

- [X] Signatures should support streaming compilations

- [X] A module may contain multiple signatures possibly with key identifiers for each signature
  - Required for key rotation
  - Keys may be short-term
  - There can be a signature for sections 1, 2, 3 and another one for sections 1, 2, 3, 4 in order to make section 4 optional, yet still verifiable when needed
  - [Oak](http://projectoak.com) use case, where each signature represents a property
  - The ability to require multiple signers to trust a module (ex: CI system + module maintainer)
  - Key identifiers can be used by verifiers to map to public key identifiers they have, and/or quickly reject a signature if it does not match.

- [X] The format used to encode signatures and related should be extensible

- [X] Arbitrary sections may not be signed (partial signature)

- [X] Arbitrary sections may be ignored during verification (partial verification)

## Discussed options

- **(a) Sign complete bytecode**

  Append “signature” section at the end of the file.

- **(b) Sign all bytecode preceding the “signature” Section**

  This allows adding new Custom Sections after the signature was created.
  See [Appendix 1](#appendix-1).

- **(c) Sign all bytecode since the previous “signature” section**

  Sign bytecode since the previous signature section or start of the module if there wasn’t any (i.e. signing consecutive groups of sections).

  This allows adding new custom sections after the signature was created, and removing consecutive groups of sections along with their signatures.

- **(d) Sign hashes of consecutive sections**

  Split sections into parts (consecutive sections, delimited by a delimiter) that can be signed and verified independently, and sign the concatenation of their hashes.

  This allows complete and partial verification of a module using a single signature, as well as adding new custom sections with their signatures.

  See [Appendix 2](#appendix-2).

- **(e) Include manifest in the “signature” section**

  In the signature section, include a manifest which describes which sections are signed using a given signature. This allows signing arbitrary sets of sections.

## Scratch notes

- Should signatures have associated metadata/annotations?
- Should the signature section include a timestamp, a module version, or more generally, should we define a minimal set of optional/required metadata that will be signed along with the rest of the module?
- We may want the set of signature algorithms we support and the ones required by the [WASI-crypto](https://github.com/jedisct1/WASI-crypto/blob/main/docs/wasi-crypto.md#algorithms) proposal to overlap.
- Add new “description” and “usage” Custom Sections, to be defined in this working group, then presented to the WebAssembly core as a standalone proposal.

## Appendix 1

**Use case for adding new Custom Sections *after* the original signature was generated, and the need for signatures to be able to cover only part of the Wasm module.**

1. Wasm module author compiles Wasm module (bytecode)
2. Wasm module author signs Wasm module (`sign(hash(bytecode))`), and appends the signature in a “signature” (`A`) Custom Section.
3. Wasm module author distributes signed Wasm module:

| sections          | covered by "signature" (`A`) |
| ----------------- | ---------------------------- |
| bytecode          | yes                          |
| "signature" (`A`) |                              |

4. Customer uploads Wasm module to the Wasm optimization service, which verifies that the Wasm module is signed by the provided public key.
“Wasm optimization” service compiles uploaded Wasm module (`compile(bytecode)`), appends it in a `precompiled_runtimeX` Custom Section, then signs the complete Wasm module (`sign(hash(bytecode+signatureA+precompiled_runtimeX))`), and appends the signature in a “signature” (`B`) Custom Section:

| sections               | covered by "signature" (`A`) | covered by "signature" (`B`) |
| ---------------------- | ---------------------------- | ---------------------------- |
| bytecode               | yes                          | yes                          |
| "signature" (`A`)      |                              | yes                          |
| `precompiled_runtimeX` |                              | yes                          |
| "signature" (`B`)      |                              |                              |

## Appendix 2

This signature format allows full and partial signatures, as well as incremental updates.

It requires two custom section types, or a byte to differentiate three different cases:

- A signature section.
- A delimiter to separate parts (consecutive sections)

| sections                                  |
| ----------------------------------------- |
| signatures                                |
| part _(one or more consecutive sections)_ |
| delimiter                                 |
| part _(one or more consecutive sections)_ |
| delimiter                                 |
| ...                                       |
| part _(one or more consecutive sections)_ |
| delimiter                                 |

**Parts and delimiters:**

A module is split into one or more parts (one or more consecutive sections).
Each part is followd by a delimiter: a section containing a 16 byte random string.

| sections                                       |
| ---------------------------------------------- |
| `p1` = input part 1 _(one or more sections)_   |
| `d1` = delimiter 1                             |
| `p2` = input part 2 _(one or more sections)_   |
| `d2` = delimiter 2                             |
| ...                                            |
| `pn` = input part `n` _(one or more sections)_ |
| `dn` = delimiter `n`                           |

If a signature covers the entire module (i.e. there is only one part), the delimiter
is optional. However, its absence prevents additional sections to be added and signed later.

**Signature section:**

A signed module starts with a custom section containing:

- An identifier representing the version of the specification the module was signed with.
- An identifier representing the hash function whose output will be signed.
- Hashes of parts being signed, and their signatures, serialized using nested custom sections as documented in [Appendix 3](#appendix-3).

That custom section must be the first section of a signed module.

A hash is computed for all the parts to be signed:

`hn = H(pn‖dn)`

A signature is computed on the concatenation of these hashes:

`hashes = h1 ‖ h2 ‖ … ‖ hn`

`s = Sign(k, "wasmsig" ‖ spec_version ‖ hash_id ‖ hashes)`

The signature section of an entire module signed using a single key has the following structure:

|                                               |                       |                   |
| --------------------------------------------- | --------------------- | ----------------- |
| `hashes = H(p1‖d1) ‖ H(p2‖d2) ‖ … ‖ H(p1‖dn)` | _(optional)_ `key id` | `Sign(k, hashes)` |

That section must be the first section of a module.

One or more signatures can be associated with `hashes`, allowing multiple parties to sign the same data.

**Example schema for the hashes and signatures:**

```json
[
    {
        "hashes": "...",
        "signatures": [
            {
                "key_id?": "...",
                "signature": "..."
            }
        ]
    }
]
```

**Signature verification algorithm for an entire module:**

1. Verify the presence of the signature section, extract the specification version, the hash function to use and the signatures.
2. Check that at least one of the signatures is valid for `hashes`. If not, return an error and stop.
3. Split `hashes` (included in the signature) into `h1 … hn`
4. Read the module, computing the hash of every `(pi, di)` tuple with `i ∈ {1 … n}`, immediately returning an error if the output doesn't match `hi`
5. Return an error if the number of the number of hashes doesn't match the number of parts.
6. Verify that the signature is valid for `hashes`.

**Partial signatures:**

The above format is compatible with partial signatures, i.e. signatures ignoring one or more parts. In order to do so, a signer only includes the hashes of relevant parts.

By default, partial signatures must be ignored by WebAssembly runtimes. An explicit configuration is required to accept a partially signed module.

**Partial verification:**

The format is also compatible with partial verification, i.e. verification of an arbitrary subset of a module:

1. Verify the presence of the header, extract the specification version, the hash function to use and the signatures.
2. Check that at least one of the signatures is valid for `hashes`. If not, return an error and stop.
3. Split `hashes` (included in the signature) into `h1 … hn`
4. Read the module, computing the hash of every `(pi, di)` tuple to verify, immediately returning an error if the output doesn't match `hi`
5. Return an error if the number of the number of hashes doesn't match the number of parts to verify.
6. Verify that the signature is valid for `hashes`.

Notes:

- Subset verification doesn't require additional signatures, as verification is always made using the full set `hashes`.
- Verifiers don't learn any information about removed sections due to delimiters containing random bits.

**Multiple signatures:**

The format also supports:

- Multiple signatures for a given section set (`hashes`). Signatures can be added incrementally, without any overhead beyond the signature sizes.
- Arbitrary section subsets and signatures combinations.
- Signature verification even if sections have been reordered.

**Implementation complexity:**

We expect the most common scenario to be entire modules being signed and verified, using one or more signatures.

Supporting additional use cases introduces implementation complexity, that can be summarized as follows:

| Complexity | Signatures | Signed sections | Verified sections | Arbitrary combinations | Reordering |
| ---------- | ---------- | --------------- | ----------------- | ---------------------- | ---------- |
| 1          | 1          | all             | all               | no                     | no         |
| 2          | 1+         | all             | all               | no                     | no         |
| 3          | 1+         | any subset      | signed subset     | no                     | no         |
| 4          | 1+         | any subset      | any subset        | no                     | no         |
| 5          | 1+         | any subset      | any subset        | yes                    | no         |
| 6          | 1+         | any subset      | any subset        | yes                    | yes        |

The specification will define what implementations must, should and may implement based on real-world requirements.

All support levels share the same signature format, so "must" and "should" feature sets can be updated incrementally.

**Detached signatures:**

Signatures can also be detached, i.e. not stored in the module itself, but provided separately.

A detached signature is equivalent to the payload of a signature section.

Given an existing signed module with an embedded signature, the signature can be detached by:

- Copying the payload of the signature section
- Removing the signature section.

Reciprocally, a detached signature can be embedded by adding a signature section, whose payload is a copy of the detached signature.

Implementations should accept signatures as an optional parameter. If this parameter is not defined, the signature is assumed to be embedded, but the verification function remains the same.

## Appendix 3

**Algorithms and identifiers:**

Identifier for the current version of the specification: `0x01`

A conformant implementation must include support for the following hash functions:

| Function | Identifier |
| -------- | ---------- |
| SHA-256  | `0x01`     |

**Signature algorithms and key serialization:**

For interoperability purposes, a conformant implementation must include support for the following signature systems:

- Ed25519 (RFC8032)

Public and private keys must include the algorithm and parameters they were created for.

| Key type           | Serialized key size | Identifier |
| ------------------ | ------------------- | ---------- |
| Ed25519 public key | 1 + 32 bytes        | `0x01`     |
| Ed25519 key pair   | 1 + 64 bytes        | `0x81`     |

Representation of Ed25519 keys:

- Ed25519 public key:

`0x01 ‖ public key (32 bytes)`

- Ed25519 key pair:

`0x81 ‖ secret key (32 bytes) ‖ public key (32 bytes)`

Implementations may support additional signatures schemes and key encoding formats.

**Serialization of structured data:**

Structured data is serialized by nesting custom sections.

A `(key, value)` pair is stored in a custom section whose name matches `key` and the payload is set to `value`.
List elements are custom sections with an empty name.

The following JSON representation of an object representing a set of hashes and their signatures:

```json
[
    {
        "hashes": "...",
        "signatures": [
           {
               "key_id_1": "...",
               "signature_1": "..."
           },
           {
               "key_id_2": "...",
               "signature_2": "..."
           }
        ]
    }
]
```

is serialized as:

```text
custom_section("",
    custom_section("hashes", "...") ǁ
    custom_section("signatures",
        custom_section("",
            custom_section("key_id_1", "...") ǁ
            custom_section("signature_1", "...")
        )
        custom_section("",
            custom_section("key_id_2", "...") ǁ
            custom_section("signature_2", "...")
        )
    )
)
```

`custom_section(name, payload)` outputs a custom section named `name` with payload `payload`.

This serialization format can easily be implemented by reusing mechanisms already present in WebAssembly runtimes.