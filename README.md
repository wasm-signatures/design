# WebAssembly signatures

## Project scope

This work is specifically about embedded digital signatures in WebAssembly modules, not about package/OCI signatures.

The goal is to converge on requirements that can be used to build out an initial and extensible specification.

Relevant projects:

- [WasmSign](https://github.com/jedisct1/wasmsign) - a tool to embed signatures in WebAssembly modules
- [wasm-sign](https://github.com/frehberg/wasm-sign) - another tool to sign WebAssembly modules
- [Istio/Envoy example](https://github.com/proxy-wasm/proxy-wasm-cpp-host/pull/147) load-time check of WasmSign signatures
- [Validation in Lucet](https://bytecodealliance.github.io/lucet/Integrity-and-authentication.html)
- [WAPM package manager](https://medium.com/wasmer/securing-wapm-packages-with-package-signing-3cf0d12f32f3) signature verification

## Requirements and justifications

### Signatures may implement multiple signature algorithms
- Compliance requirements
- Public key size/signature size/verification CPU cost tradeoffs (ex: PQ schemes vs EdDSA)
- Code and key reuse (ex: ECDSA-SHA3 for blockchains already leveraging SHA3, EdDSA to use existing SSH keys)

### It should be possible to verify a module file before execution 

### It should be possible to add signed custom sections

...to a potentially signed module so that one can verify the original module and the additional custom section independently

- See Appendix 1. A user may want to add additional signed information (debug data, precompiled headers) to a signed module and then re-distribute. Users may or may not trust and choose to consume the additional information.

### Signatures should support streaming compilations

### A module may contain multiple signatures

...possibly with key identifiers for each signature  

- Required for key rotation
- Keys may be short-term
- There can be a signature for sections 1, 2, 3 and another one for sections 1, 2, 3, 4 in order to make section 4 optional, yet still verifiable when needed
- Oak use case, where each signature represents a property
- The ability to require multiple signers to trust a module (ex: CI system + module maintainer)
- Key identifiers can be used by verifiers to map to public key identifiers they have, and/or quickly reject a signature if it does not match.
  
### The format used to encode signatures and related should be extensible

## Scratch notes

- Should signatures have associated metadata/annotations?
- Should the signature section include a timestamp, a module version, or more generally, should we define a minimal set of optional/required metadata that will be signed along with the rest of the module?
- We may want the set of signature algorithms we support and the ones required by the WASI-crypto proposal to overlap.
- Add new “description” and “usage” Custom Sections, to be defined in this working group, then presented to the WebAssembly core as a standalone proposal.

## Discussed options

### (a) Sign complete bytecode

Append “signature” section at the end of the file.

### (b) Sign all bytecode preceding the “signature” Section

This allows adding new Custom Sections after the signature was created (see: Appendix 1). 

### (c) Sign all bytecode since the previous “signature” section

Sign bytecode since the previous signature section or start of the module if there wasn’t any (i.e. signing consecutive groups of sections).

This allows adding new Custom Sections after the signature was created, and removing consecutive groups of sections along with their signatures.

### (d) Sign hashes of consecutive sections

Split sections into parts (consecutive sections, delimited by a marker) that can be signed and verified independently, and sign the concatenation of their hashes.

This allows complete and partial verification of a module using a single signature, as well as adding new custom sections with their signatures.

See Appendix 2.

### (e) Include manifest in the “signature” section

In the signature section, include a manifest which describes which sections are signed using a given signature. This allows signing arbitrary sets of sections.

## Appendix 1

Use case for adding new Custom Sections *after* the original signature was generated, and the need for signatures to be able to cover only part of the Wasm module.

1. Wasm module author compiles Wasm module (bytecode)
2. Wasm module author signs Wasm module (`sign(hash(bytecode))`), and appends the signature in a “signature” (`A`) Custom Section.
3. Wasm module author distributes signed Wasm module:

| sections          | covered by "signature" (`A`) |
| ----------------- | ---------------------------- |
| bytecode          | yes                          |
| "signature" (`A`) |                              |

1. Customer uploads Wasm module to the Wasm optimization service, which verifies that the Wasm module is signed by the provided public key.
“Wasm optimization” service compiles uploaded Wasm module (`compile(bytecode)`), appends it in a `precompiled_runtimeX` Custom Section, then signs the complete Wasm module (`sign(hash(bytecode+signatureA+precompiled_runtimeX))`), and appends the signature in a “signature” (`B`) Custom Section:

| sections               | covered by "signature" (`A`) | covered by "signature" (`B`) |
| ---------------------- | ---------------------------- | ---------------------------- |
| bytecode               | yes                          | yes                          |
| "signature" (`A`)      |                              | yes                          |
| `precompiled_runtimeX` |                              | yes                          |
| "signature" (`B`)      |                              |                              |

