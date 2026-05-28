---
eip: XXXX
title: WYRIWE — What You Read Is What You Execute
description: An input-provenance commitment scheme and attestation profile for verifiable AI agent inference
author: Tiago Merlini (@TMerlini)
discussions-to: https://ethereum-magicians.org/t/wyriwe-what-you-read-is-what-you-execute-input-provenance-for-verifiable-ai-inference/28655
status: Draft
type: Standards Track
category: ERC
created: 2026-05-28
requires: 712
---

## Abstract

This ERC defines a triple-hash commitment scheme and EIP-712 attestation profile for proving that the input a model received is the input the user intended. It introduces three linked fields — `raw_input_hash`, `sanitization_pipeline_hash`, and `input_hash` — that together form a verifiable chain of custody for AI inference inputs. A verifier can confirm input integrity using only the committed hashes and the public sanitization specification, without trusting the agent, gateway, or execution environment. This standard occupies layer 3 of the AI inference trust stack, complementing ERC-8004 (agent identity) and ERC-8263 / OCP (execution attestation).

---

## Motivation

On-chain AI agent systems built on standards such as ERC-8004, ERC-8263, and ERC-8274 can attest to which model ran and what output was produced, but no standard defines how to commit to the *input* before inference. This creates a trust gap: an agent may sanitize, rewrite, or substitute the user's input between request submission and model execution, leaving no on-chain evidence of the transformation.

Without a committed input record:
- A settlement contract cannot verify that the delivered output corresponds to the funded input.
- A proof verifier (e.g. an `IProofVerifier` implementation) cannot confirm the `inputHash` it receives matches what was originally requested.
- A dispute resolution mechanism has no ground truth for what the model was actually asked to do.

WYRIWE (What You Read Is What You Execute) closes this gap by defining a minimal, hash-based commitment that any compliant gateway MUST produce at execution time and that any verifier can check independently.

---

## Specification

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED", "MAY", and "OPTIONAL" in this document are to be interpreted as described in RFC 2119.

### 1. Triple-Hash Construction

A WYRIWE-compliant execution MUST produce the following three values:

```
raw_input_hash             = keccak256(raw_user_input)
sanitization_pipeline_hash = keccak256(sanitization_spec_cid || raw_input_hash)
input_hash                 = keccak256(sanitized_input)
```

Where:

- `raw_user_input` is the exact bytes of the user's input as received, before any transformation.
- `sanitization_spec_cid` is the IPFS CID (as UTF-8 bytes) of the sanitization pipeline specification applied to the input.
- `sanitized_input` is the exact bytes fed to the model after applying the sanitization pipeline.
- `||` denotes byte concatenation.

**Verification invariant:** Given `raw_input_hash`, `sanitization_pipeline_hash`, and the public sanitization specification at `sanitization_spec_cid`, any verifier MUST be able to confirm that `input_hash` is the correct output of that pipeline applied to that raw input. No party needs to be trusted to assert this.

### 2. IDENTITY_SENTINEL — No-Sanitization Case

When no sanitization is applied (identity transform), the following MUST hold:

```
sanitization_pipeline_hash = keccak256(IDENTITY_SENTINEL_CID || raw_input_hash)
input_hash                 = raw_input_hash
```

`IDENTITY_SENTINEL_CID` is a stable IPFS reference to the identity-transform specification:

```
ipfs://QmTst97dG8i9tFrutdetqMbVhSHqJGJaxMmPzWCcVVTWDU
```

The `input_hash == raw_input_hash` equality in the no-sanitization case is a provable on-chain claim, not an assumption. Implementations MUST NOT omit `sanitization_pipeline_hash` even when no sanitization is applied.

### 3. WyriweAttestation Struct

A WYRIWE attestation is an EIP-712 typed structured data record with the following fields:

```solidity
struct WyriweAttestation {
    bytes32 agentId;                    // ERC-8004 agent identity anchor
    address registry;                   // ERC-8004 registry address
    bytes32 modelHash;                  // Hash of model weights or manifest
    bytes32 rawInputHash;               // keccak256(raw_user_input)
    bytes32 sanitizationPipelineHash;   // keccak256(sanitization_spec_cid || raw_input_hash)
    bytes32 inputHash;                  // keccak256(sanitized_input)
    bytes32 outputHash;                 // keccak256(model_output)
    uint256 timestamp;                  // Unix timestamp of execution
}
```

All fields are REQUIRED. A conforming attestation MUST populate every field. `agentId` and `registry` MAY be zero-valued if the execution environment does not implement ERC-8004, but MUST NOT be omitted from the struct.

### 4. EIP-712 Domain

The EIP-712 domain separator for WYRIWE attestations is:

```solidity
EIP712Domain({
    name:    "ERC8004AttestationGateway",
    version: "1",
    chainId: 1
})
```

The `l4_signature` is an `eth_sign` signature over the EIP-712 digest of the `WyriweAttestation` struct, produced by the gateway attestor.

### 5. Verification Procedure

A verifier MUST execute the following steps to accept a WYRIWE attestation as valid:

1. Recompute the EIP-712 digest from the attestation struct fields and verify `l4_signature` against the known attestor address.
2. Verify `rawInputHash == keccak256(raw_user_input)` if the raw input is available.
3. Fetch the sanitization specification at `sanitization_spec_cid` and apply it to `raw_user_input`; verify the result hashes to `inputHash`.
4. If `sanitization_spec_cid == IDENTITY_SENTINEL_CID`, verify `inputHash == rawInputHash`.
5. Accept the attestation only if all applicable steps pass.

Steps 2–4 are REQUIRED when the corresponding inputs are available. Step 1 is always REQUIRED.

### 6. Gateway Query Interface

A conforming gateway MUST expose the following HTTP endpoint:

```
GET /agent/verify/:inputHash
```

Where `:inputHash` is the hex-encoded (no `0x` prefix) `inputHash` value. The response MUST be a JSON object containing the `WyriweAttestation` fields and the `l4_signature`. The response MUST use HTTP 200 on success and HTTP 404 when no attestation exists for the given `inputHash`.

---

## Rationale

### Why three hashes?

Two hashes (raw and final) are insufficient: they prove the input was transformed but do not commit to *which* transformation was applied. The `sanitization_pipeline_hash` commits to both the specification and the raw input, making the transform auditable and reproducible by any third party.

### Why IPFS CIDs for the sanitization spec?

Content-addressed references ensure the specification retrieved at verification time is identical to the one applied at execution time. A mutable URL reference would allow the spec to be swapped after the fact, defeating the commitment.

### Why EIP-712?

EIP-712 typed structured data signatures are natively verifiable on-chain by Ethereum contracts, wallet UIs, and existing tooling. The structured hash is deterministic and auditable without any off-chain oracle.

### Why include `agentId` and `registry`?

Linking attestations to an ERC-8004 agent identity makes the attestation attributable — not just to a signing key, but to an on-chain registered agent. This is load-bearing for settlement systems (e.g. ERC-8274, ERC-8183) that need to associate an output with a specific funded agent.

### Why `IDENTITY_SENTINEL_CID` instead of a null value?

Using a null or zero value for `sanitization_pipeline_hash` in the no-sanitization case would make it ambiguous whether the field was intentionally omitted or a transform was applied but not committed. The sentinel makes the no-sanitization case explicit, auditable, and verifiable on equal footing with sanitized cases.

---

## Backwards Compatibility

This ERC introduces a new standard with no dependencies on or conflicts with existing ERCs beyond the voluntary integration points described in the Specification. It does not modify any existing interface.

---

## Test Cases

### Case 1: Identity transform (no sanitization)

```
raw_user_input             = "transfer 1 ETH to 0xABCD..."
raw_input_hash             = keccak256("transfer 1 ETH to 0xABCD...")
sanitization_spec_cid      = "ipfs://QmTst97dG8i9tFrutdetqMbVhSHqJGJaxMmPzWCcVVTWDU"
sanitization_pipeline_hash = keccak256(sanitization_spec_cid_bytes || raw_input_hash)
sanitized_input            = raw_user_input
input_hash                 = raw_input_hash
```

Expected: `input_hash == raw_input_hash` — verifier confirms identity case.

### Case 2: Sanitization applied

```
raw_user_input             = "transfer 1 ETH to 0xABCD... <script>alert(1)</script>"
raw_input_hash             = keccak256(raw_user_input)
sanitization_spec_cid      = "ipfs://Qm<strip-html-spec-cid>"
sanitized_input            = "transfer 1 ETH to 0xABCD..."
sanitization_pipeline_hash = keccak256(sanitization_spec_cid_bytes || raw_input_hash)
input_hash                 = keccak256(sanitized_input)
```

Expected: `input_hash != raw_input_hash` — verifier fetches spec, applies strip-HTML transform to raw input, confirms result matches `input_hash`.

### Case 3: Attestation forgery (should fail)

```
input_hash (claimed) = keccak256("transfer 100 ETH to attacker")
```

Verifier fetches attestation for this `input_hash`, recomputes `sanitization_pipeline_hash` from the claimed sanitized input and the committed spec, finds mismatch with the attested `sanitization_pipeline_hash`. Attestation rejected.

---

## Reference Implementation

A live WYRIWE-compliant gateway is deployed at:

```
GET https://gateway.ensub.org/agent/verify/:inputHash
```

Example query:
```
https://gateway.ensub.org/agent/verify/758d61f26a44448384e5c4468a0dcb7a2abe456067b0f7b505bc28b9411fe931
```

Source code: https://github.com/Echo-Merlini/hbs-attestation-poc

---

## Security Considerations

### Attestor key compromise

The `l4_signature` is only as trustworthy as the attestor's private key. Implementations that use WYRIWE attestations for settlement or dispute resolution SHOULD maintain an on-chain registry of authorised attestor addresses and support key rotation. A compromised attestor key allows forged attestations but does not break the hash commitments — a forged attestation for a hash with no matching input still cannot produce a valid preimage.

### Hash collision resistance

WYRIWE relies on keccak256 collision resistance. No practical collision attacks against keccak256 are known. If keccak256 is broken, all three hashes are affected equally; the triple-hash construction does not introduce additional collision surface.

### Sanitization spec CID stability

`sanitization_pipeline_hash` commits to a CID, not the spec content. If the IPFS content at the referenced CID becomes unavailable, step 3 of the verification procedure cannot be completed. Implementations SHOULD pin all referenced sanitization spec CIDs to ensure long-term verifiability.

### Replay and cross-domain attacks

The `timestamp` field in `WyriweAttestation` is informational and does not prevent replay. Settlement contracts that consume WYRIWE attestations MUST enforce their own replay protection (e.g. by recording consumed `inputHash` values on-chain). The EIP-712 `chainId` in the domain separator prevents cross-chain signature replay.

### Input availability

WYRIWE commits to the *hash* of the input, not the input itself. The raw input and sanitized input are not published by this standard. Parties who need to reproduce verification MUST retain the original inputs off-chain. WYRIWE does not define an input storage or retrieval mechanism.

---

## Copyright

Copyright and related rights waived via [CC0](../LICENSE.md).
