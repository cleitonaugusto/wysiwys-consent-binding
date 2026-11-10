# WYSIWYS Consent Binding for Agent UIs

**An open, protocol-neutral primitive that binds what a human saw to what an agent executes.**

- **Status:** Draft RFC v0.3 · **Author:** Cleiton Augusto Correa Bezerra ([github.com/cleitonaugusto](https://github.com/cleitonaugusto)) · **DOI (v0.1 archived):** [10.5281/zenodo.23933180](https://doi.org/10.5281/zenodo.23933180)
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

**Normative byte layout (canonical v1).** All integers little-endian; all floats IEEE-754 `f64` little-endian; strings are length-prefixed UTF-8 (`u32` length, then bytes):

```text
"WYSIWYS\x01"                         8-byte magic + format version
u32(node_count)
for each node, in surface order:
  u8  interactive                     0 = static, 1 = interactive
  str id
  str content                         resolved displayed text/label/value ("" if none)
  u8  target_present                  0 | 1
  [ str target ]                      only if target_present == 1
  u8  geometry_present                0 | 1
  [ f64 x, f64 y, f64 w, f64 h ]      only if geometry_present == 1
str action                            the action name the consent concerns
```

Any implementation that reproduces these bytes and hashes them with `alg` gets the same `digest`. The conformance vectors (below) pin it.

### 4.2 Carrying the binding

When the renderer dispatches the action (or the client submits the human's decision), it includes:

```json
"consent": { "alg": "sha256", "digest": "<base64>", "nonce": 7, "sig": "<optional base64>" }
```

`sig`, when present, is a signature by a key the verifier trusts (renderer or device key) over the deterministic message:

```text
"WYSIWYS-SIG\x01"  str alg  str digest  str action  u64(nonce)
```

(same string/integer encoding as §4.1; the distinct prefix prevents collision with the surface bytes). `nonce` gives replay resistance. Signing is a deployment choice; the digest alone already lets a verifier that recomputes the surface detect drift, provided the binding reaches it over a channel the adversary cannot modify. Anyone who can edit the dispatched action can also recompute an unsigned digest over the edited surface. A verifier that is configured with a key, or that receives the binding over a channel the adversary can modify, MUST refuse a binding that carries no `sig`; accepting an unsigned binding in that case reduces the check to digest-only.

**Protocol placement.** In AG-UI the binding rides in `resume[].metadata` (the slot the spec describes as "a signature proving the human decision was not tampered with"), namespaced under a `wysiwys` key and tied to the `interruptId`/`toolCallId`. In A2UI it is a new optional `action.consent` field; in MCP-UI it is carried in the `postMessage` result; in Open-JSON-UI it accompanies the action event.

### 4.3 Verification contract — the dispatch gate

The core operation is a **gate at dispatch**. Before acting on a consequential action, the **verifier** recomputes the consent surface it is *about to act on* — at the instant of dispatch, from the resolved action target (the amount it will charge, the URL it will open, the tool args it will run) — and compares digests. It MUST recompute at this point and MUST NOT reuse the surface captured at approval time, which is exactly the value an adversary can change in between. **Anything but an exact match MUST refuse the action.**

This closes the render→act drift (T0→T1 / TOCTOU) *by construction*, and it needs **no trusted display**: the gate only has to run at the party that performs the call — the host / gateway / tool-dispatch point, which in an agent stack already sits downstream of the renderer. The digest proves *what the human was shown and approved*; the *economic truth* (funds actually moved) still comes signed from whoever performs the settlement — the two are complementary, not a substitute for each other.

### 4.4 Properties

What the binding gives is **integrity of the surface about to be acted on**, not **authenticity of what a human's eyes saw**:

- Proves: the executed action corresponds to the exact surface that was approved, and — when signed — that the approval is non-repudiable. Any post-approval drift is refused.
- Does **not** prove: that the displayed values were themselves true at approval time (a renderer that lied *before* the human approved is out of scope — see §8), or that settlement succeeded. Those are separate attestations.

### 4.5 Conformance

A set of normative conformance vectors — named surfaces with their expected `digest` under the canonical v1 layout — is published alongside this document as [`conformance/wysiwys-vectors.json`](./conformance/wysiwys-vectors.json). An implementation conforms if, for each vector's `surface`, it produces the listed `digest`. The vectors cover a baseline surface, a swapped displayed value, a swapped action target under an unchanged label, and a geometry-bound surface.

## 5. Protocol bindings (how it maps)

| Protocol | Where the binding lives |
| --- | --- |
| **A2UI** (Google) | A new optional `action.consent` field on the dispatched `action`. |
| **AG-UI** (CopilotKit) | **Define** what the existing `resume[].metadata` "signature proving the human decision was not tampered with" covers: the consent digest over the rendered generative-UI surface (A2UI/MCP-UI/Open-JSON-UI) tied to the `toolCallId`/`interruptId`. The hook already exists; this gives it a verifiable contract. |
| **MCP-UI / MCP Apps** (Microsoft, Shopify, Anthropic; unifies the OpenAI Apps SDK) | the consent binding travels with the iframe→host action message (`{ type: "tool", payload: { toolName, params } }`); the host recomputes the surface it is about to act on and refuses on mismatch — making an app-rendered surface a *verifiable* approval surface, not a host-side TODO. |
| **Open-JSON-UI** (OpenAI) | the consent binding accompanies the action event emitted from the rendered declarative surface (OpenAI's Apps SDK schema, now flowing through MCP Apps). |

Because **AG-UI is the runtime that carries A2UI / MCP-UI / Open-JSON-UI**, defining the binding at the AG-UI layer lets all three inherit WYSIWYS from one place — the highest-leverage point.

## 6. Compatibility & adoption

- **Additive and optional.** A `consent` field / defined metadata is ignored by parties that don't implement it; existing flows are unchanged.
- **Opt-in per surface** via a capability flag, so only consequential surfaces pay the (small) cost.
- **No change** to component catalogs, layout, rendering, or data-model semantics.
- **Incremental.** A renderer can emit the digest before any verifier checks it; value grows as both ends adopt.

## 7. Reference implementation

An open-source reference exists that resolves agent-rendered UI surfaces (e.g. a representative purchase surface) into an enforced tree, computes the consent digest over displayed content and the resolved action target, binds it to a signed, replayable receipt at approval time, and refuses on mismatch at execute time — demonstrated on two scenarios (amount swap via a data-model update; destination swap under an unchanged label). Available for review.

## 8. Threat model

**In scope — post-approval drift.** A dishonest or injected agent that renders one surface and acts on another; any change to the resolved action or the displayed values between the human's approval and the call — an `updateDataModel`, a swapped data-model path, a different server-side charge, a re-pointed click target. The dispatch gate (§4.3) recomputes and refuses, with no trusted display required.

**Out of scope — a renderer that lied *before* approval.** If the surface the human approved was already a lie, the binding faithfully attests that lie and the gate admits it. Catching that is *authenticity of what was displayed*, which is unachievable outside a surface the verifier owns (a trusted path / secure display, or a renderer/device key the verifier trusts — a deployment concern). This RFC gives integrity of the surface about to be acted on, and names this residual rather than implying it away.

Also out of scope: the truthfulness of the underlying data (a separate attestation from its source) and whether settlement succeeded (signed by whoever performs it).

## 9. Prior art & alignment

**WYSIWYS is not new; its application to agent tool-calls is.** The property has independent, battle-tested lineages worth naming — and the honest limits each one hit sharpen this one.

- **Payments — PSD2 dynamic linking.** Under the EU strong-customer-authentication rules, the authentication code must be *dynamically linked* to the specific amount and payee shown to the user, and any change to either invalidates the code. "Show X, settle Y" is therefore already a named non-conformance, written as a property of the authorization code rather than of the mandate. The limit is instructive: dynamic linking binds the code to the *values*, not to what was *rendered* on a screen the issuer does not control; payments moved the residual to *liability*, assigning loss to the party best placed to prevent it. The agent case is drawn differently — the party that recomputes (the host / gateway that dispatches the tool-call) is often itself downstream of the renderer, so the invalidation can bind to the **dispatched action**, not only the displayed values.
- **Web3 — EIP-712 / ERC-7730 clear signing.** "What you see is what you sign" is the hardware-wallet discipline. EIP-712 structured-data signing makes a signature cover typed, human-meaningful fields instead of opaque bytes; ERC-7730 (clear-signing metadata) standardizes how a wallet renders what is being signed, binding the display to the signed payload — the countermeasure to *blind signing*. This RFC is the same move for agent-rendered surfaces and tool-calls, where no wallet owns the display.
- **Agents on hardware wallets.** [nearai/ironclaw#6532](https://github.com/nearai/ironclaw/issues/6532) (a design issue opened July 2026, with no code on its main branch when reviewed in November 2026) has an agent that holds no key prepare a transaction for approval on a Ledger device, using ERC-7730 clear signing and an approval hash over the rendered text, signing bytes, signer and chain. It is the closest published design we know of for on-chain agents and has the same shape as this primitive. This RFC addresses the complementary case: agent-rendered surfaces and tool-calls where no wallet or secure screen exists.

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
- **Agentic on-chain execution.** Where an agent both renders a surface *and* signs/dispatches a transaction, the consent digest and the EIP-712 signed payload collapse into a single binding point — a natural home for WYSIWYS (agentic wallets, x402-style agent payments, autonomous DeFi). This area is moving quickly (see §9), and how a consent digest should compose with those signing flows remains open.

---

*This document is protocol-neutral by design. Per-protocol proposals specialize §5 for a single spec; this RFC is the shared primitive they implement.*
