# Provenance and AI disclosure

CarryLog was started from an empty local project during the November 3–24,
2026 Fe Builders Challenge build window. No pre-existing application source
is included. The entry's contribution is the Fe frontier, the positional
checkpoint verifier, and their integration as a journal with typed receipts.
The left-balanced tree is a known design; no new cryptographic primitive
or claim of research novelty is made.

| Source or dependency | Use | License / status |
| --- | --- | --- |
| [Fe v26.5.0](https://github.com/argotorg/fe/tree/v26.5.0), including bundled `core` and `std` | Compiler, EVM tests, ABI and Keccak helpers | [Apache-2.0](https://github.com/argotorg/fe/blob/v26.5.0/LICENSE-APACHE); no source vendored |
| [Fe Builders Challenge starter](https://github.com/fe-lang/builders-challenge-starter) and [Fe Guide](https://fe-lang.org/) | Setup and language/API reference; standard Apache license text taken from the starter | Starter Apache-2.0; no starter contract, tests or CI copied |
| [RFC 9162 §2.1](https://www.rfc-editor.org/rfc/rfc9162.html#section-2.1) | Reference for left-balanced shape and bottom-up proof order | Algorithm reference; no text or code incorporated |
| [Python](https://docs.python.org/3/license.html) and [OpenSSL](https://openssl-library.org/source/license/index.html) | Optional local artifact proof builder and independent fixture regeneration | PSF-2.0 and Apache-2.0 respectively; no packages or binaries vendored |

OpenAI Codex helped research the toolchain and rules, propose and implement
the design, write the Fe code, develop the independent test fixtures and local
artifact proof builder, run and review checks, and draft the documentation.
The automated checks were run locally. The local draft does not claim human authorship, review, or acceptance
that has not taken place. Before submission, the human participant should
review the source and behavior, confirm their own and any teammate's actual
contributions, and agree to be the responsible contact or name the person who
will be. Record any later human or AI changes accurately in the declaration.
