# SPEC — Cross-Agent Handoff Attestor

Status: **public behaviour specification, version 1.0.** Specification release only.
No implementation, Tenable listing, Exchange submission, PR, or Contribution Agreement
acceptance is authorized by this document.

Product form: MCP server + skill.
Written from product behaviour and capability intent. No harvested body was quoted,
translated, or structurally reproduced in producing this specification.

---

## 1. Purpose

When agent A hands work to agent B, B normally has no way to check what A was actually
permitted to delegate. Scope silently widens across a chain of handoffs, and a
prompt-injected upstream agent can hand downstream agents more authority than it ever
held.

This product makes a handoff a **signed, verifiable attestation**: issuer, subject,
task scope, constraints, provenance chain, freshness, and single-use nonce. The
receiver verifies before acting, and the resulting receipt is byte-compatible with the
Agent Action Evidence Ledger so a whole delegation chain lands in one verifiable export.

Non-goal: transporting the handoff, authenticating agents, or managing keys.

## 2. Definitions

- **Attestation** — the signed record of one handoff.
- **Scope** — the set of capabilities the subject is permitted to exercise, expressed as
  an explicit list of capability tokens plus optional resource patterns.
- **Resource-pattern containment** — only literals and patterns with one trailing `*` are
  valid. A child pattern `C` is contained by parent pattern `P` exactly when either: (a) `P`
  is literal and `C` is the identical literal; or (b) `P` ends in `*`, and after removing
  that final `*` to obtain prefix `p`, `C` is either a literal beginning with `p`, or a
  trailing-`*` pattern whose prefix begins with `p`. Empty prefixes are allowed only for
  the pattern `*`. No path normalization, separator interpretation, case folding, or
  percent-decoding occurs before comparison; containment is over Unicode code points.
- **Constraint** — an additional restriction that must hold at exercise time
  (e.g. maximum spend, environment, read-only flag).
- **Chain** — the ordered list of prior attestation digests that led to this one.
- **Key adapter** — caller-supplied `sign(bytes) → signature` and
  `verify(bytes, signature, keyRef) → boolean`.

## 3. Inputs

### 3.1 `issue_handoff`

| Field | Type | Required | Notes |
|---|---|---|---|
| `issuer` | string ≤ 256 | yes | opaque agent label |
| `subject` | string ≤ 256 | yes | receiving agent label |
| `task` | string ≤ 1024 | yes | human-readable statement of work |
| `scope.capabilities` | array of tokens, 1–256 | yes | `^[a-z][a-z0-9._-]{0,63}$` |
| `scope.resources` | array of patterns, ≤ 256 | no | literal or single-`*`-suffix patterns |
| `constraints` | object of `{ value, comparator }`, ≤ 64 keys | no | comparator is `equal`, `max-number`, `min-number`, `subset`, or `boolean-require` |
| `parent` | verified parent attestation or digest plus caller-supplied parent record | no | absent means root; issuance refuses an unresolved parent |
| `notBefore` / `expiresAt` | RFC 3339 UTC | yes (`expiresAt`) | `expiresAt` ≤ 24 h ahead by default |
| `keyRef` | string ≤ 256 | yes | identifies the signing key to the adapter |

### 3.2 `verify_handoff`
`{ attestation, ancestors?, keyRef?, now?, replayStore }` — `ancestors` is the ordered
root-to-parent attestation set needed to verify continuity and narrowing. The verifier is a
pure function of these inputs plus the caller-supplied replay store. The replay store is
keyed by `(subject, nonce)` and supports atomic check-and-record.

### 3.3 `check_scope`
`{ parent, child }` — answers whether `child` is a scope subset of `parent`.

### 3.4 `describe_keys`
Returns the registered adapter's algorithm identifier and key references it will accept.
Never returns key material.

## 4. Outputs

### 4.1 Attestation
```
{
  "format": "cahs-attestation/1",
  "canonicalization": "1",
  "issuer": "planner-1",
  "subject": "executor-3",
  "task": "summarize the incident tickets for 2026-09",
  "scope": { "capabilities": ["ticket.read", "doc.write"],
             "resources": ["tickets/2026-09*"] },
  "constraints": { "environment": "staging", "maxToolCalls": 40 },
  "parent": "1a0b…",
  "chainDepth": 2,
  "issuedAt": "2026-09-11T18:00:00Z",
  "notBefore": "2026-09-11T18:00:00Z",
  "expiresAt": "2026-09-11T19:00:00Z",
  "nonce": "b7c1…",
  "digest": "7e41…",
  "signature": { "alg": "Ed25519", "keyRef": "planner-1/2026-09", "value": "…" }
}
```

### 4.2 Verification result
```
{ "valid": false,
  "checks": { "signature": "pass", "freshness": "pass", "replay": "pass",
              "scopeSubset": "fail", "constraintCompat": "pass",
              "chainContinuity": "pass" },
  "failures": [ { "check": "scopeSubset",
                  "detail": "capability 'secret.read' not present in parent scope" } ] }
```

### 4.3 Receipt
A ledger-shaped entry (`kind: "attestation"`) whose canonical bytes are produced by the
**same canonicalization** as the Agent Action Evidence Ledger, so it verifies inside a
ledger export unchanged.

## 5. Invariants

1. **Monotonic narrowing.** A child attestation's scope must be a subset of its parent's:
   every child capability token appears in the parent, and every child resource pattern is
   contained by a parent pattern under §2’s exact rule. Widening is always a verification
   failure.
2. **Constraint compatibility.** Each child constraint must be equal to or stricter than
   the parent's, per that constraint's declared comparator. An unknown constraint key in
   the child is a failure, never an ignore.
3. **Freshness.** `notBefore ≤ now < expiresAt`, evaluated against caller-supplied `now`.
   A child must satisfy `child.notBefore ≥ parent.notBefore` and
   `child.expiresAt ≤ parent.expiresAt`. Every ancestor must verify and be unexpired at issue
   time; verification reports each ancestor’s historical signature/window validity.
4. **Single use.** A nonce accepted once is refused thereafter for the same subject. Replay
   identity is exactly `(subject, nonce)` and acceptance is atomically recorded.
5. **Signature binding.** The signature covers the canonical bytes of every field above
   except `signature` itself; changing any covered field invalidates it.
6. **Chain continuity.** If `parent` is present, `chainDepth = parentDepth + 1`, and the
   complete root-to-parent chain must be presentable and verifiable by the receiver. The
   maximum accepted `chainDepth` is 64; deeper issuance or verification fails closed.
7. **Verification is total.** Every check reports `pass`, `fail`, or `unknown`. A check
   that cannot be evaluated is `unknown` and makes the overall result invalid — never a
   silent pass.
8. **No execution.** The package never runs a handed-off task.
9. **No key material** is generated, stored, logged, or exported by the package. The
   bundled in-memory development adapter is explicitly marked non-production and is
   refused when a production flag is set.

## 6. State transitions

```
DRAFT --issue_handoff--> ISSUED(signed, nonce minted)
ISSUED --verify_handoff(now < notBefore)--> NOT_YET_VALID   (invalid)
ISSUED --verify_handoff(valid window, unseen nonce)--> ACCEPTED
ACCEPTED --verify_handoff(same nonce)--> REPLAYED           (invalid)
ISSUED --verify_handoff(now >= expiresAt)--> EXPIRED        (invalid)
ACCEPTED --issue_handoff(parent=this)--> ISSUED(chainDepth+1)
```
State lives with the caller's replay store; the package holds no cross-call state.

## 7. Failure modes

| Condition | Behaviour |
|---|---|
| Adapter signing error | `E_SIGN`; no attestation returned |
| Unknown `keyRef` at verify | `checks.signature = "unknown"`, result invalid |
| Malformed attestation | `E_INPUT` with JSON pointer; no partial verification result |
| `expiresAt` beyond configured maximum lifetime | `E_INPUT` at issue time |
| Missing replay store at verify | `checks.replay = "unknown"`, result invalid |
| Parent digest unavailable to the receiver | `checks.chainContinuity = "unknown"`, invalid |
| Clock skew beyond tolerance | reported in `checks.freshness` with observed skew |
| Parent invalid, expired at issue, or wider child lifetime | issuance refused; verification marks the corresponding check `fail` |
| Chain depth above 64 | `E_LIMIT` at issue; verification invalid |
| Unknown comparator or incomparable value type | `E_INPUT` at issue; `constraintCompat: unknown` at verification |

## 8. Security boundaries

**In scope:** privilege escalation across handoffs, replayed handoffs, stale handoffs,
tampered handoffs, and forged chain lineage.

**Out of scope:** key distribution and rotation, agent identity provisioning, transport
security, and confidentiality of the task text — an attestation is integrity-protected,
not encrypted, and callers must not place secrets in `task` or `constraints`.

**Refused capabilities:** task execution, outbound calls, credential handling, key
generation in production mode.

## 9. Worked example

1. `planner-1` holds scope `["ticket.read","doc.write"]` on `tickets/2026-09*`.
2. It issues a handoff to `executor-3` with the same capabilities restricted to
   `tickets/2026-09-0*`. Verification: all checks pass; nonce recorded.
3. `executor-3` issues a further handoff to `tool-runner-9` adding `secret.read`.
4. The receiver's `verify_handoff` returns `valid: false`, `scopeSubset: "fail"`,
   detail naming `secret.read`. `tool-runner-9` refuses the work.
5. Both attestations, the acceptance, and the refusal are recorded as receipts and
   verify inside a single Agent Action Evidence Ledger export.

## 10. Externally testable properties

| ID | Property |
|---|---|
| P1 | Any single-bit change to a covered field invalidates the signature. |
| P2 | Adding a capability or broadening a resource pattern relative to the parent always fails `scopeSubset`. |
| P3 | Loosening any constraint relative to the parent always fails `constraintCompat`. |
| P4 | The same nonce verified twice for one subject yields `replay: "fail"` on the second. |
| P5 | Verification at `now = expiresAt` fails; at `expiresAt - 1 ms` passes. |
| P6 | Substituting a different conforming key adapter changes no verification outcome other than signature validity. |
| P7 | Receipt canonical bytes are byte-identical to those produced by the Evidence Ledger canonicalizer for the same logical record. |
| P8 | With no replay store supplied, no attestation is ever reported valid. |
| P9 | No exported symbol, log line, or error contains key material or a private key reference beyond `keyRef`. |
| P10 | Production mode refuses the development in-memory key adapter. |
| P11 | Every literal/wildcard containment case in the published table follows §2 exactly, including `tickets/2026-*` containing `tickets/2026-09*` but not the reverse. |
| P12 | A child lifetime never begins before or ends after its parent, and chain depth 65 is refused. |
