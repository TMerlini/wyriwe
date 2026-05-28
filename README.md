# WYRIWE — What You Read Is What You Execute

**Input-provenance profile for verifiable AI inference**

> *A lightweight commitment scheme that proves the input a model received is the input the user intended — without trusting the agent, the gateway, or the execution environment.*

---

## The Problem

On-chain AI agent systems can verify *what model ran* and *what output was produced*, but have no standard way to verify *what input was actually fed to the model*. An agent can sanitize, rewrite, or substitute the input between the user's request and model execution — and no existing standard detects this.

WYRIWE closes that gap.

---

## The Triple-Hash Scheme

Every WYRIWE-compliant execution produces three hashes, in order:

```
raw_input_hash          = keccak256(raw_user_input)
sanitization_pipeline_hash = keccak256(sanitization_spec_cid || raw_input_hash)
input_hash              = keccak256(sanitized_input)
```

| Field | Description |
|---|---|
| `raw_input_hash` | Hash of the user's original input, before any transformation |
| `sanitization_pipeline_hash` | Commitment to the sanitization transform applied (spec CID + raw hash) |
| `input_hash` | Hash of the sanitized input actually fed to the model |

**The invariant:** a verifier who knows `raw_input_hash`, `sanitization_pipeline_hash`, and the sanitization spec can independently verify that `input_hash` is the correct output of that pipeline applied to that input. No party needs to be trusted.

---

## The IDENTITY_SENTINEL Branching Invariant

When no sanitization is applied (identity transform):

```
sanitization_pipeline_hash = keccak256(IDENTITY_SENTINEL_CID || raw_input_hash)
input_hash                 = raw_input_hash
```

`IDENTITY_SENTINEL_CID` is a stable IPFS reference to the identity-transform specification:

```
ipfs://QmTst97dG8i9tFrutdetqMbVhSHqJGJaxMmPzWCcVVTWDU
```

This makes the no-sanitization case explicit and verifiable — `input_hash == raw_input_hash` is a provable claim, not an assumption.

---

## EIP-712 Attestation Profile

A WYRIWE attestation is an EIP-712 signed struct:

```solidity
struct WyriweAttestation {
    bytes32 agentId;              // ERC-8004 agent identity
    address registry;             // ERC-8004 registry address
    bytes32 modelHash;            // Hash of model weights / manifest
    bytes32 rawInputHash;         // keccak256(raw user input)
    bytes32 sanitizationPipelineHash; // keccak256(spec_cid || raw_input_hash)
    bytes32 inputHash;            // keccak256(sanitized input)
    bytes32 outputHash;           // keccak256(model output)
    uint256 timestamp;            // Unix timestamp of execution
}
```

The EIP-712 domain:

```solidity
EIP712Domain({
    name:    "ERC8004AttestationGateway",
    version: "1",
    chainId: 1  // Ethereum mainnet
})
```

The `l4_signature` is `eth_sign` over the EIP-712 digest of the above struct, produced by the gateway attestor.

---

## Stack Position

WYRIWE is the **Input Trust Layer** — layer 3 in the four-layer AI inference trust stack:

| Layer | Standard | Responsibility |
|---|---|---|
| L1 | Model manifest hash | Proves which model ran |
| L2 | ERC-8004 | Proves which agent ran it |
| L3 | **WYRIWE** | Proves what input was fed to the model |
| L4 | ERC-8263 / OCP | Proves the execution happened and output is committed |

---

## Integration

| Standard | Integration point |
|---|---|
| **ERC-8263** | WYRIWE is the formal input-provenance profile for the `proofHash` construction; `input_hash` is the canonical shared key |
| **ERC-8274** | `IProofVerifier.verify(modelHash, inputHash, outputHash, proof)` — `inputHash` is WYRIWE's `input_hash`; `proof` encodes the full attestation struct |
| **ERC-8183** | Outcome envelope `commitmentRef` field maps to WYRIWE's `input_hash` committed at job funding time; §11 reserves the producer-facing disposition shape without fixing field names |
| **ERC-8004** | `agentId` in the attestation struct is the ERC-8004 agent identity anchor |
| **OCP** | WYRIWE attestation is the concrete input-commitment profile that OCP proof envelopes can carry |

---

## Reference Implementation

Live gateway with queryable WYRIWE attestations:

```
GET https://gateway.ensub.org/agent/verify/:inputHash
```

Returns a signed `WyriweAttestation` struct for any `inputHash` recorded by the gateway.

Example:
```
https://gateway.ensub.org/agent/verify/758d61f26a44448384e5c4468a0dcb7a2abe456067b0f7b505bc28b9411fe931
```

Source (attestation PoC): https://github.com/Echo-Merlini/hbs-attestation-poc

---

## Status

WYRIWE was proposed and formalised by [Tiago Merlini](https://github.com/TMerlini) in the ERC-8263 alignment thread.

It is incorporated by reference in:
- [ERC-8263 v0.2](https://ethereum-magicians.org/t/erc-8263-onchain-proof-layer-for-ai-agents/28577) — §proofHash Constructions appendix
- [ERC-8004 + ERC-8263 + OCP Composition Note](https://gist.github.com/damonzwicker/8742e742bdc627b8e2179c00b81289dc)

---

*WYRIWE — What You Read Is What You Execute*
*Tiago Merlini — dinamic.eth*
