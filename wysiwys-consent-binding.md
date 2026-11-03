# WYSIWYS Consent Binding for Agent UIs

**An open, protocol-neutral primitive that binds what a human saw to what an agent executes.**

- **Status:** Draft RFC v0.2 · **Author:** Cleiton Augusto Correa Bezerra ([github.com/cleitonaugusto](https://github.com/cleitonaugusto)) · **DOI (v0.1 archived):** [10.5281/zenodo.23933180](https://doi.org/10.5281/zenodo.23933180)
- **Applies to:** any protocol that renders agent-generated UI and acts on a human decision — A2UI (Google), AG-UI (CopilotKit), MCP-UI (Microsoft + Shopify), Open-JSON-UI (OpenAI), and custom equivalents.

## 1. Abstract

Agent UI stacks let an AI agent render a surface, obtain a human's approval, and then execute or settle an action. Across today's specs, *what the human saw and approved* is **not bound** to *what the action dispatches or settles*. An agent (or an injection that reaches it) can display one thing and act on another — "show $100, settle $10,000"; "show *Open the invoice PDF*, open a look-alike host" — within a fully conformant surface, with no artifact proving what was shown.

This RFC defines a small, additive, protocol-neutral primitive — a **consent binding** — that carries a signed digest of the displayed surface with the human's decision, so the executing/settling side can verify the decision corresponds to the exact surface the human approved. It is the "what you see is what you sign" (WYSIWYS) property from trusted-path and hardware-wallet security, applied to agent tool-calls.

## 2. Problem

- **Displayed ≠ dispatched.** Displayed values come from a data model / state; the dispatched payload (`context`, `args`, a resume `payload`) is an independent field. Nothing ties them.
- **State drifts between render and act (T0→T1).** An update between the snapshot the human saw and the moment of execution is invisible to the approval.
- **Existing hooks are advisory or undefined.** User-activation / `isTrusted` proves a *gesture* happened, not that the human saw what they authorized. Some protocols expose a "signature proving the decision was not tampered with" slot but leave **what it covers** undefined and uninterpreted. Consent is typically "SHOULD obtain", app-defined.

The gap is **cross-cutting**: it is the same in every generative-UI-for-agents protocol, and most acute for the agentic-payments/commerce surfaces these protocols target.

## 3. Terminology

- **Consent surface** — the laid-out, *resolved* (post-data-binding) content a human is approving: the interactive control plus the content it visually concerns (labels, amounts, payee, destination), taken as actually displayed.
- **Consent digest** — a deterministic hash over a canonical encoding of the consent surface and the action's resolved target.
- **Consent binding** — the consent digest (and optionally a signature over it) carried with the dispatched action or the human's decision.
- **Verifier** — the component that executes/settles and checks the binding before acting.

## 4. The primitive

### 4.1 Consent surface → consent digest

For a user-activated action, the renderer computes, **after data-model/state resolution**:

```
consent_digest = H( canonical(
  for each placed node in the consent surface:
    id, role(interactive?), resolved_text/label/value,
  action: { name, resolved_target }   // openUrl url, event name, resolved context/args
  [, geometry]                         // optional: binds overlap/placement too
) )
```

`H` is a named hash (default `sha256`). `canonical` is deterministic: fixed field order, length-prefixed strings, explicit algorithm id. Geometry MAY be included so clickjacking-style placement changes are covered.

### 4.2 Carrying the binding

When the renderer dispatches the action (or the client submits the human's decision), it includes:

```json
"consent": { "alg": "sha256", "digest": "<base64>", "sig": "<optional base64>" }
```

`sig`, when present, is a signature over `(alg, digest, action_id, nonce)` by a key the verifier trusts (renderer or device key). Signing is a deployment choice; the digest alone already lets a verifier that recomputes the surface detect drift.

### 4.3 Verification contract

Before acting on a consequential action, the **verifier** recomputes the consent surface it *intends to act on* (the amount it will charge, the URL it will open, the tool args it will run) and compares digests. **A mismatch MUST refuse the action.** The digest proves *what the human was shown and approved*; the *economic truth* (funds actually moved) still comes signed from whoever performs the settlement — the two are complementary, not a substitute for each other.

### 4.4 Properties

- Proves: the executed action corresponds to the exact surface the human approved (integrity + non-repudiation of consent).
- Does **not** prove: that the displayed values were themselves true, or that settlement succeeded — those are separate attestations.

## 5. Protocol bindings (how it maps)

| Protocol | Where the binding lives |
| --- | --- |
| **A2UI** (Google) | A new optional `action.consent` field on the dispatched `action`. |
| **AG-UI** (CopilotKit) | **Define** what the existing `resume[].metadata` "signature proving the human decision was not tampered with" covers: the consent digest over the rendered generative-UI surface (A2UI/MCP-UI/Open-JSON-UI) tied to the `toolCallId`/`interruptId`. The hook already exists; this gives it a verifiable contract. |
| **MCP-UI** (MS + Shopify) | For the iframe-rendered surface, the host carries the consent digest in the `postMessage` result of the user action. |
| **Open-JSON-UI** (OpenAI) | The consent digest accompanies the action event emitted from the rendered declarative surface. |

Because **AG-UI is the runtime that carries A2UI / MCP-UI / Open-JSON-UI**, defining the binding at the AG-UI layer lets all three inherit WYSIWYS from one place — the highest-leverage point.

## 6. Compatibility & adoption

- **Additive and optional.** A `consent` field / defined metadata is ignored by parties that don't implement it; existing flows are unchanged.
- **Opt-in per surface** via a capability flag, so only consequential surfaces pay the (small) cost.
- **No change** to component catalogs, layout, rendering, or data-model semantics.
- **Incremental.** A renderer can emit the digest before any verifier checks it; value grows as both ends adopt.

## 7. Reference implementation

An open-source reference exists that resolves agent-rendered UI surfaces (e.g. a representative purchase surface) into an enforced tree, computes the consent digest over displayed content and the resolved action target, binds it to a signed, replayable receipt at approval time, and refuses on mismatch at execute time — demonstrated on two scenarios (amount swap via a data-model update; destination swap under an unchanged label). Available for review.

## 8. Threat model (brief)

In scope: a dishonest or injected agent that renders one surface and acts on another; drift between display and settlement. Out of scope: a compromised renderer that both displays and signs maliciously (mitigated by a trusted renderer/device key, a deployment concern); the truthfulness of displayed data (a separate attestation from its source).

## 9. Prior art & alignment (AP2 / FIDO)

The agentic-payments world has independently converged on the same property, which validates this primitive and shows where it fits. Google's **Agent Payments Protocol (AP2)** — whose core specification work is continuing in the **FIDO Alliance** — defines a **Trusted Surface**: *"a secure, non-agentic interface that renders Mandate Content to the User for authorization and consent,"* producing a tamper-evident, signed Mandate (a Verifiable Digital Credential). The confirmation UI the user sees (merchant, amount, line items) is carried in the signed transaction data and hash-bound to the mandate, and AP2's threat model explicitly treats the agent as a potential attacker. In other words, AP2 is the **payment-specific instantiation** of the property this RFC generalizes.

What AP2 does **not** define is *how an agent-generated UI surface (A2UI, AG-UI, MCP-UI, Open-JSON-UI) becomes a Trusted Surface* — it assumes a non-agentic surface (a banking app, a hosted 3DS page, a redirect). This RFC is complementary on two fronts:

1. **The bridge.** The consent binding is a mechanism by which an agent-rendered surface can satisfy AP2's Trusted-Surface requirement: it binds exactly what was displayed to the authorization, so the verifier can guarantee the surface.
2. **The generalization.** It extends the same WYSIWYS property to *every* consequential action — not only payments — where no payment-mandate standard applies (opening a URL, deleting data, posting, transferring).

This RFC therefore aligns with, rather than competes with, AP2/FIDO: same property, broader scope, and the missing link for agent-rendered UI.

## 10. Open questions

- Minimal vs full consent surface: default scope and how a surface widens it.
- Whether `sig` is in-scope for each protocol or left to deployments.
- Canonicalization details shared across protocols (so a digest is portable).
- Interaction with streaming surface/state updates between render and dispatch.
- Alignment with AP2 mandate hashing (`sd_hash` / `checkout_hash`) so a consent digest can feed a payment mandate directly.

---

*This document is protocol-neutral by design. Per-protocol proposals specialize §5 for a single spec; this RFC is the shared primitive they implement.*
