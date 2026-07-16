# 🌪️ HoloRAID

## Authenticated holographic-inspired threshold erasure coding

HoloRAID encrypts data, disperses the ciphertext across `n` signed CRT shards, and reconstructs the plaintext from any `k` authentic shards.

> **Security boundary:** HoloRAID v3 provides plaintext confidentiality through ChaCha20-Poly1305, threshold key release through Shamir secret sharing, and malicious shard-rewrite detection through Ed25519 signatures. Keep the signer fingerprint outside the shard directory.

**License:** MIT · **Python:** 3.8+ · **Dependency:** `cryptography` · **Format:** signed binary shards · **Default:** 6-of-10 recovery

## What changed in v3

The original CRT prototype demonstrated exact any-`k` erasure recovery, but deterministic plaintext residues and self-checksums were not sufficient security. HoloRAID v3 adds:

- encrypt-before-disperse with a fresh 256-bit key and nonce;
- Shamir `k`-of-`n` sharing of the data-encryption key;
- Ed25519 signatures on the manifest and every shard;
- an out-of-band SHA-256 signer fingerprint;
- dataset-bound metadata to stop shard mixing and replay;
- AEAD verification and ciphertext hashing during recovery;
- bounded authenticated subset recovery;
- a one-file CLI and executable adversarial test suite.

SafeGear remains an invertible modular representation transform. **It is not encryption.**

## Quick start

```python
from holoraid import HoloRAID

raid = HoloRAID(n=10, k=6)

data = b"The universe does not collapse; it flows through coprime gears."
shards = raid.encode(data)

# Store this fingerprint through an independent trusted channel.
trust_anchor = shards.trust_anchor

# Lose four of ten records. Any six authentic shards still suffice.
shards[0] = None
shards[3] = None
shards[5] = None
shards[9] = None

recovered = raid.decode(
    shards,
    expected_trust_anchor=trust_anchor,
)
assert recovered == data
```

The in-memory `trust_anchor` is convenient for a local demonstration. For persisted data, do not trust a fingerprint read only from the same directory as the shards.

## Security model

For plaintext `P`, HoloRAID generates a random 256-bit key `K`, nonce `N`, and dataset identifier `D`:

```text
C = ChaCha20-Poly1305.Encrypt(K, N, P, associated_data=D||n||k||length)
```

The ciphertext is divided into bounded integers `x_j`. For each prime modulus `m_i`, HoloRAID stores a reversible SafeGear-wound projection:

```text
r_i,j = x_j mod m_i
w_i,j = gear_i * r_i,j mod m_i
```

The data key is independently split using a degree-`k-1` Shamir polynomial. Shard `i` contains one Shamir share and one sequence of CRT projections. Every security-relevant field is signed with Ed25519.

Recovery requires both:

```text
k authentic ciphertext projections
k authentic key shares
```

A rewritten shard fails signature verification and becomes a detectable erasure. A replaced manifest signed under another key fails the separately retained signer fingerprint.

## Key properties

| Property | HoloRAID v3 guarantee |
|---|---|
| Any-`k` recovery | Any `k` authentic shards reconstruct the encrypted payload exactly |
| Plaintext confidentiality | AEAD encryption occurs before CRT projection |
| Threshold key access | Fewer than `k` Shamir shares do not reconstruct the data key |
| Rewrite detection | Ed25519 authenticates every shard and manifest |
| Cross-dataset isolation | Random dataset ID is signed and AEAD-bound |
| Exact integrity | Ciphertext hash and Poly1305 tag must verify |
| Damage tolerance | Up to `n-k` missing or rejected shards |
| SafeGear | Reversible modular bijection; not a cryptographic secrecy layer |
| Coding classification | MDS-like optimal erasure threshold in a heterogeneous residue alphabet |

### What HoloRAID does not claim

- A residue contains information about encrypted ciphertext coordinates; it is not literally information-free.
- HoloRAID is not a proof of the physical holographic principle, AdS/CFT, black-hole entropy, or quantum gravity.
- CRT/RRNS coding, Shamir sharing, AEAD, and signatures are established prior art.
- The present implementation is not independently audited or production-certified.
- Ed25519 authentication is not post-quantum.

The original contribution is the specific integrated HoloRAID architecture and file format: authenticated threshold key release joined to signed heterogeneous CRT projections and SafeGear representation transforms.

## Mathematical foundations

### Theorem 1 — Chinese Remainder reconstruction

For pairwise-coprime moduli `m_1, …, m_k`, the congruences

```text
x ≡ r_i (mod m_i)
```

have a unique solution modulo `M = product(m_i)`.

### Theorem 2 — Any-`k` threshold property

Let `M_k` be the product of the `k` smallest moduli. HoloRAID chooses the plaintext-ciphertext chunk capacity so every encoded integer satisfies `x < M_k`. Therefore any subset of `k` moduli has a product at least `M_k` and uniquely reconstructs `x`.

### Theorem 3 — SafeGear bijection

For `gcd(b, m)=1`, the map

```text
f(x) = b*x mod m
```

is a bijection on `Z_m`, with inverse

```text
f^-1(y) = b^-1*y mod m.
```

### Theorem 4 — Threshold key reconstruction

A degree-`k-1` Shamir polynomial reconstructs its constant term from any `k` distinct evaluations. HoloRAID enforces a nonzero highest coefficient so the effective degree cannot accidentally fall below `k-1`.

### Theorem 5 — Authenticated erasure conversion

Assuming Ed25519 unforgeability, a storage attacker without the signing key cannot modify a signed shard while retaining a valid signature. Invalid records are rejected before reconstruction and are treated as erasures.

### Coding-distance statement

HoloRAID reconstructs after any `n-k` erasures and generally cannot reconstruct from `k-1` records. This gives the optimal erasure threshold `d = n-k+1` in its heterogeneous residue-coordinate model. To avoid conflating it with a conventional linear code over one finite-field alphabet, the project describes this as **MDS-like** rather than claiming a standard Reed-Solomon-style MDS classification.

## Installation

```bash
git clone https://github.com/shaunpaull/HoloRAID.git
cd HoloRAID
python -m pip install -e .
```

Or install the dependency and run the single module directly:

```bash
python -m pip install cryptography
python holoraid.py self-test
```

## Command line

Encode:

```bash
holoraid encode input.bin vault \
  --signing-key holoraid_signer.key \
  --n 10 --k 6
```

The command prints a SHA-256 trust fingerprint. Copy it to an independent trusted channel.

Decode:

```bash
holoraid decode vault recovered.bin \
  --trust-anchor <64-character-fingerprint>
```

Self-test:

```bash
holoraid self-test --bytes 1048576 --trials 100
```

## Executed reference results

A 1 MiB run of the included suite produced:

```text
Public HoloRAID API           : authenticated round-trip=True
Configuration                 : n=10, k=6
Recoverable erasures          : 4/10
CRT chunk capacity            : 45 bytes
Payload-only storage factor   : 1.778x
Measured serialized factor    : 1.780x
Exact authenticated recovery  : True
All k-subsets                 : 210/210 exact
3 rewritten + 1 erased        : recovered=True
Rewritten shard signatures    : rejected=3
Below-threshold access        : blocked=True
Same plaintext randomized     : True
Plaintext absent from shards  : True
Signer substitution attack    : blocked=True
Random attack campaign        : 100/100 exact recoveries
Injected shard rewrites       : 94 (signature-rejected)
```

Throughput varies by machine and Python version. The reference implementation prioritizes inspectability over optimized performance.

## Damage tolerance and overhead

Maximum erasure tolerance is:

```text
(n-k)/n
```

| Configuration | Erasure tolerance | Ideal `n/k` ratio |
|---|---:|---:|
| `n=10, k=6` | 40% | 1.67× |
| `n=10, k=5` | 50% | 2.00× |
| `n=10, k=3` | 70% | 3.33× |
| `n=10, k=2` | 80% | 5.00× |
| `n=10, k=1` | 90% | 10.00× |

High damage tolerance is not free. As `k` decreases, each shard must carry more recoverable capacity and storage overhead approaches replication.

## API reference

### `HoloRAID(n=10, k=6, prime_start=2**61-1)`

Properties:

- `max_failures`
- `redundancy_factor`
- `damage_tolerance`
- `chunk_size`

Methods:

- `encode(data) -> EncodedShards`
- `decode(encoded, expected_trust_anchor=...) -> bytes`
- `encode_bundle(data) -> Bundle`
- `recover(manifest, shards, expected_trust_anchor=...)`
- `encode_file(input_path, output_dir)`
- `decode_file(shard_dir, output_path, expected_trust_anchor=...)`
- `info()`
- `measure_holographic_property(data)`

### `EncodedShards`

A mutable list-like object supporting:

```python
encoded[2] = None
for shard in encoded:
    ...
```

It also carries the signed manifest and the same-process trust fingerprint.

## Holographic interpretation

HoloRAID uses the following computational analogy:

| Holographic language | HoloRAID object |
|---|---|
| Bulk | encrypted payload |
| Boundary coordinates | signed modular shards |
| Bulk-to-boundary map | CRT projection |
| Boundary recovery | threshold CRT reconstruction |
| Representation change | SafeGear modular bijection |

This analogy is useful for reasoning about distributed representation and recoverable invariants. It is not an isomorphism to quantum entanglement, holographic quantum codes, or gravitational dynamics.

## Prior art and positioning

HoloRAID builds on established work in:

- the Chinese Remainder Theorem and redundant residue number systems;
- threshold secret sharing;
- computational secret sharing / encrypt-then-disperse systems;
- authenticated encryption;
- digital signatures and erasure coding.

A defensible project description is:

> HoloRAID is an authenticated, confidential, holographic-inspired threshold storage architecture combining AEAD-encrypted payload dispersal, Shamir-shared keys, signed CRT/RRNS projections, and reversible SafeGear representation transforms.

## Threat model

Protected against, within the configured threshold:

- missing shards;
- bit rot and malformed shard records;
- malicious shard-content rewrites without the signing key;
- shard replay or mixing across datasets;
- complete bundle replacement signed under an untrusted key;
- fewer-than-`k` attempts to recover the data key.

Not protected against:

- compromise of the Ed25519 private signing key;
- compromise of `k` authentic shards;
- rollback to an older correctly signed bundle unless external versioning is used;
- metadata disclosure such as file size, `n`, `k`, and shard count;
- endpoint compromise before encryption or after recovery;
- denial-of-service through file deletion or resource exhaustion;
- future quantum attacks against Ed25519.

See `SECURITY.md` for deployment guidance.

## Future directions

- streaming segmented AEAD for very large files;
- authenticated Merkle indexing and random-access repair;
- hardware-backed signer keys and rotation;
- fuzzing and property-based malformed-input testing;
- Rust/C acceleration;
- hybrid Ed25519 plus post-quantum signatures;
- formal file-format specification and external audit;
- comparison against Reed-Solomon, IDA, and RRNS baselines.

## Contributing

Contributions are welcome, especially in testing, cryptographic review, optimized reconstruction, interoperable ports, and honest comparative benchmarking.

## License

MIT. See `LICENSE`.

## Citation

```bibtex
@software{holoraid2026,
  author = {Gerrard, Shaun Paul},
  title = {HoloRAID: Authenticated Holographic-Inspired Threshold Erasure Coding},
  year = {2026},
  url = {https://github.com/shaunpaull/HoloRAID},
  note = {Research reference implementation}
}
```

🌪️💜 **The universe does not collapse; it flows through coprime gears.** 💜🌪️
