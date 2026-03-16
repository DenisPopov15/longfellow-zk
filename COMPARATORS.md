## Numeric Comparators in Longfellow ZK

This note summarizes what is required to express predicates like
“`balance > threshold` without revealing `balance`” inside the Longfellow
ZK system, and how that relates to the existing code.

### 1. What exists today

- **Logic / comparison primitives**
  - `lib/circuits/logic/logic.h` provides:
    - Bitvectors over the circuit field: `Logic::bitvec<N>` (`v8`, `v32`, `v256`, etc.).
    - Comparison helpers:
      - `lt`, `leq` on arrays of bits (`BitW`) and
      - `vlt`, `vleq` on bitvectors.
  - These implement comparisons over bit-encoded integers inside the circuit, returning a `BitW` (boolean wire) that can be fed into `assert1` / `assert0`.

- **JWT / MDOC circuits**
  - The JWT circuit (`lib/circuits/tests/jwt/jwt.h`) and MDOC / anoncred circuits primarily:
    - Verify signatures (ECDSA over P-256).
    - Verify hashes (SHA-256).
    - Check **string equality** or “attribute present with this exact value” by:
      - Parsing/decoding payload bytes (base64/CBOR),
      - Shifting / routing to the right position,
      - Comparing bytes or short strings.
  - Current JWT tests (`jwt_test.cc`) open attributes like `"given_name":"Erika"` and prove equality; they do **not** implement `>` on numeric fields in-circuit.

### 2. What is needed for `balance > threshold` (without revealing balance)

To support a predicate of the form:

> There exists a signed credential with field `balance` such that `balance > threshold`,
> while keeping `balance` itself hidden,

you need a new circuit and witness format with the following pieces:

- **(a) Circuit-side numeric representation of `balance`**
  - Decide how `balance` is encoded in the signed payload:
    - As a decimal string `"10000"`, or
    - As a binary/CBOR integer.
  - In the circuit:
    - Route or parse the bytes for the `balance` field from the payload.
    - Convert those bytes into a bitvector `balance_bits` (e.g. little-endian binary) using logic gates.
    - Represent the public threshold as:
      - Either a constant bitvector `threshold_bits`, or
      - A field element that is then expanded into bits.

- **(b) A comparison gadget for `>`**
  - Use the existing comparison functions on bitvectors:
    - For vectors: `Logic::vlt`, `Logic::vleq`.
  - Implement:

    ```c++
    // Conceptually: assert(balance > threshold)
    auto is_gt = L.vlt(threshold_bits, balance_bits);  // threshold < balance
    L.assert1(is_gt);
    ```

  - The key point: `balance_bits` lives in the **private witness** only; the circuit never outputs it as a public value, only the fact that the inequality holds.

- **(c) A new circuit definition**
  - Create a circuit similar in spirit to `JWT<...>` or `Small<...>` that:
    - Verifies the ES256 signature of the JWT / VC (reusing `VerifyCircuit` and SHA-256 gadgets).
    - Locates and parses the `balance` field in the payload.
    - Builds `balance_bits` and enforces `balance > threshold` via the comparison gadget.
    - Exposes only:
      - Public inputs (issuer public key, hash of the kb2 message, possibly the threshold itself),
      - And a “predicate holds” condition (encoded by `assert1(is_gt)`), *not* the numeric balance.
  - Compile this with `QuadCircuit` / `mkcircuit` into a `Circuit<Field>` and serialize it like existing circuits.

- **(d) A matching witness generator**
  - Implement a `BalanceWitness` helper (similar to `JWTWitness`) that:
    - Accepts: `(jwt_or_vc_string, issuer_pkx, issuer_pky, threshold)`.
    - Parses the JWT / VC (JSON or CBOR) outside the circuit.
    - Extracts the numeric `balance` field and converts it into the bit representation expected by the circuit.
    - Fills the ECDSA and SHA witnesses the same way `JWTWitness::compute_witness` does.
  - This code learns the actual `balance`, but only passes its encoded bits as **private** witness entries to the prover.

- **(e) Integration with the ZK engine and protocols**
  - Reuse the generic ZK stack:
    - `ZkProver` / `ZkCommon` / `sumcheck` / `ligero` work unchanged; they are circuit-agnostic.
  - For protocol-level use (e.g. OpenID4VP / ISO 18013-5 style):
    - Assign the new circuit a `circuit_hash` and define a `ZkSpec` entry (similar to entries in `lib/circuits/mdoc/zk_spec.cc`).
    - Ensure verifiers know that this `circuit_hash` means: “credential is valid and `balance > threshold`” (not an equality predicate).

### 3. Why this doesn’t exist “for free” today

- The **building blocks are present**:
  - Bit-level logic (`Logic`),
  - Comparators (`lt` / `leq` / `vlt` / `vleq`),
  - ECDSA and SHA circuits,
  - Generic ZK engine (sumcheck + Ligero).
- What’s **missing** is a **domain-specific circuit** that:
  - Knows how to parse a particular credential format (JWT / MDOC / VC),
  - Extracts a numeric field like `balance`,
  - Applies a numeric comparison,
  - And keeps the underlying value private.

Designing and wiring that circuit (plus its witness builder and any ZK spec metadata) is the main work required to support `balance > threshold` or similar hidden-value comparator predicates in this codebase.

