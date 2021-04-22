# WebAssembly signatures

<!-- @import "[TOC]" {cmd="toc" depthFrom=2 depthTo=6 orderedList=false} -->

<!-- code_chunk_output -->

- [WebAssembly signatures](#webassembly-signatures)
  - [Project scope](#project-scope)
  - [Requirements and justifications](#requirements-and-justifications)
  - [Discussed options](#discussed-options)
  - [Scratch notes](#scratch-notes)
  - [Appendix 1](#appendix-1)
  - [Appendix 2](#appendix-2)

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

  Split sections into parts (consecutive sections, delimited by a marker) that can be signed and verified independently, and sign the concatenation of their hashes.

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

**A single signature, still allowing verification of a subset and incremental updates.**

Requires two custom section types, or a single type with a bit to differentiate both:

- A marker to delimit parts (consecutive sections)
- The signatures themselves

**Delimiting parts:**

A module can be split into parts, by inserting a small marker between them:

| sections                                       |
| ---------------------------------------------- |
| `p1` = Input part 1 _(one or more sections)_   |
| _Marker_                                       |
| `p2` = Input part 2 _(one or more sections)_   |
| ...                                            |
| _Marker_                                       |
| `pn` = Input part `n` _(one or more sections)_ |

If partial verification is not required, no markers are necessary.

The only content of a marker is a 16 byte random string.

**Format of the signature section:**

|                                               |                       |              |
| --------------------------------------------- | --------------------- | ------------ |
| `m = H(p1) ‖ H(p1 ‖ p2) ‖ … ‖ H(p1 ‖ … ‖ pn)` | _(optional)_ `key id` | `Sign(k, m)` |

Signature verification for `{p1...pℓ}`, for any `ℓ ≤ n` :

1. `v={}`
2. Compute a rolling hash from the beginning, appending its output to `v` everytime a marker is crossed
3. Immediately return an error if `v` is not a prefix of `m`
4. Check that the signature is valid for `m`.

An existing signature section should be skipped when computing the rolling hash.

**Adding a part `pn+1`, signed with a different key:**

1. A marker is added before the additional sections
2. A new signature entry is appended.

Updated signature section of an already signed module, on top of which an additional part, signed with another key, is appended:

|                                               |                        |                |
| --------------------------------------------- | ---------------------- | -------------- |
| `m = H(p1) ‖ H(p1 ‖ p2) ‖ … ‖ H(p1 ‖ … ‖ pn)` | _(optional)_ `key id`  | `Sign(k, m)`   |
| `m’ = H(p1 ‖ … ‖ pn ‖ pn+1)`                  | _(optional)_ `key id’` | `Sign(k’, m’)` |

*Note: In the simplified notation above, the markers have been omitted from the hash computation. But these sections should actually be included like other sections.*

Reusing a key doesn’t require an additional row, only an update of `m` and the signature.

**Properties:**

- Verification of the data between the beginning of a module and any marker (or the end of the module) can be made using a single signature.
- Verifiers don't learn any information about removed sections due to markers containing random bits.


**Example schema for the signature section:**

```json
{
    "version": "...",
    "entries": [
        {
            "hash_function": "...",
            "m": "...",
            "signatures": [
                {
                    "key_id?": "...",
                    "signature": "..."
                }
            ]
        }
    ]
}
```

- A section set can be signed with multiple keys
- Multiple sets can be signed incrementally.
