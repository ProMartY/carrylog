# CarryLog

CarryLog is a Fe journal for release artifact digests. A publisher commits one
digest at a time. Anyone with an authenticated checkpoint and a short proof can
check whether a digest was recorded at a particular position. This is useful
when a release is assembled in several CI jobs and a reviewer needs to check
one artifact without downloading the whole history.

## Run it locally

Use [Fe v26.5.0](https://github.com/argotorg/fe/releases/tag/v26.5.0),
which reports `fe 26.5.0 (3dab95f)`. See the
[installation guide](https://fe-lang.org/getting-started/installation/) for
your platform. On Windows x64, download the release's
`fe_windows_amd64.exe`, verify it, and run it as `fe.exe`. Its SHA-256
is `d86ca3c6fdadeb8335f7033e3a68fdb7d2aab4eafd22c1e23bec047d2a8b2674`
([release asset list](https://github.com/argotorg/fe/releases/expanded_assets/v26.5.0)).
`.fe-version` records the pin for reviewers; check the executable's version.

From this directory, with `fe` on PATH (or use its full path):

```text
fe --version
fe fmt --check
fe check
fe test
fe build
fe test --filter demo_release_receipts --show-logs
```

`fe build` emits `out/CarryLog.bin`, `out/CarryLog.runtime.bin` and
`out/CarryLog.abi.json`. Tests use Fe's built-in EVM runner: no RPC,
wallet, funds, extra ingots, or blockchain deployment are needed.

## Demo: verify a release artifact

The tiny files in [examples/](examples/) stand in for the Windows, Linux and
macOS artifacts of one release. On PowerShell, run
`Get-FileHash examples/*.txt -Algorithm SHA256` to reproduce the three
inputs. The files contain a single LF at the end of each line.

The demo creates the contract in the local EVM, appends the three digests, and
checks the Linux digest at index 1. Its original proof still verifies against
the recorded size-2 checkpoint. Verifying against the live size-3 checkpoint
requires a new sibling, and changing one bit of the digest makes that proof
fail. The demo prints three `Appended` logs: the second topic is the index,
and the data words are the digest and new root.

| After append | Size | Checkpoint root |
| --- | ---: | --- |
| Linux | 2 | `0xf589415ad7a809240aa89ba5b9e3983fd9a71ed9a100386082a5257f8faa1d6a` |
| macOS | 3 | `0xe1fe71dd6bd218c86a153dae7367791a4392ef90e022877c7da26dc115fdb942` |

## Build a proof from local files

Use Python 3.10+ and an OpenSSL executable with `-keccak-256` support.
Pass the exact file bytes in append order. The zero-based index selects
the file for which you want a proof. This example selects the Linux file.

```text
python fixtures/generate.py --openssl /path/to/openssl --proof-index 1 examples/windows.txt examples/linux.txt examples/macos.txt
```

The command emits a JSON object with `format`, `size`, `root`, `index`,
`digest`, and `path` on standard output. With the three example files,
its size and root match the macOS row above. The builder hashes each file
with SHA-256, so it applies to journal entries published with that digest
scheme. Use the full ordered prefix ending at the checkpoint you want.
Repeated file contents are allowed at different positions.

To verify an artifact, hash your local copy with SHA-256 and use that
value as `digest`. Pass the proof's `index` and `path` to
`verifyCurrent` for the current contract state. For an earlier state,
pass those fields and the separately authenticated size and root to the
Fe function `verify_checkpoint`. The JSON's own root is useful for
comparing the local history with the trusted checkpoint. The file builder
walks the full supplied history and uses an OpenSSL process per distinct
hash input, which makes it best suited to small local batches.

## Design

The [contract](src/lib.fe) stores a count and 17 peak slots, enough for
65,536 entries. Each set bit in the count represents one occupied slot at
that height. Append merges the incoming right subtree with occupied left
subtrees, like carrying in binary addition. Folding occupied peaks from
lowest to highest gives the same left-balanced tree as recursively splitting
at the largest power of two strictly below the subtree size. Each operation
visits at most a logarithmic number of levels; state does not grow with each
digest. The full ordered digest history is available from the event stream.

- `append(expected_size, digest)` returns the new root and emits
  `Appended(index, digest, root)`. Only the immutable publisher configured
  at construction can append. A stale expected size reverts, so a retried
  request cannot silently occupy a later position. At capacity, append reverts.
- `checkpoint()` returns the current `(size, root)`.
- `verifyCurrent(index, digest, siblings)` uses the contract's current
  checkpoint. The exported [Fe verifier](src/proof.fe) can verify a historical
  checkpoint supplied by its caller. It derives left/right directions and the
  exact proof length from size and index, then folds at most 16 siblings.

**Fe highlight.** The `Journal` message type defines the ABI for both callers
and handlers, with selectors checked at compile time. Handler effects make
the trust boundary visible in source: `Append` reads caller context, mutates
the frontier and emits a typed event; `Checkpoint` and `VerifyCurrent`
only read the frontier. The historical verifier has no contract storage
effects and can be used by another Fe ingot. Integration tests use the same
typed messages as the contract.

**Proof format (CarryLog v1).** A checkpoint consists of its size and root.
Let `W(x)` be a 32-byte unsigned big-endian word and `K` be Ethereum
Keccak-256. The empty root is `K("")`; a leaf is
`K(W(0) || W(digest))`; a branch is `K(W(1) || W(left) || W(right))`.
Sibling hashes are ordered from leaf to root. Digest words are opaque; the
demo uses SHA-256 for the file digest, then Keccak-256 for the tree. The tree
shape follows [RFC 9162 §2.1](https://www.rfc-editor.org/rfc/rfc9162.html#section-2.1);
its hashes and proof encoding are CarryLog-specific.

## Evidence and limits

The suite checks independent prefix roots, all seven positions of a
non-power-of-two tree, malformed proofs, stale and unauthorized writes, and
65,536 actual appends in the local EVM followed by a rejected overflow
attempt. The [fixture generator](fixtures/generate.py) uses an independent
recursive tree oracle. It is optional for users; to check the committed JSON
and generated Fe tests, use Python 3.10+ and an OpenSSL binary with
`-keccak-256` support:

```text
python fixtures/generate.py --openssl /path/to/openssl --check
```

Authenticate both size and root for the intended journal (and, if using a
chain, the chain, address and block) before trusting a proof. Each proof
covers digest inclusion at a single checkpoint. The local builder needs
the ordered artifact history, and deployments would need their own event
indexing and checkpoint retention. The publisher address is fixed, and the
publisher controls the accuracy and availability of published digests.
Validation so far covers the local EVM and independent fixtures.

License: [Apache-2.0](LICENSE). See [provenance and AI disclosure](PROVENANCE.md).
