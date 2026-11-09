# WYSIWYS Consent Binding for Agent UIs

**An open, protocol-neutral primitive that binds *what a human saw* to *what an agent executes*.**

[![DOI](https://zenodo.org/badge/DOI/10.5281/zenodo.23933180.svg)](https://doi.org/10.5281/zenodo.23933180)
License: CC-BY-4.0 · Author: **Cleiton Augusto Correa Bezerra** ([@cleitonaugusto](https://github.com/cleitonaugusto))

> 📄 **Full draft RFC:** [`wysiwys-consent-binding.md`](./wysiwys-consent-binding.md)
> 🔖 **Citable, archived version (v0.1):** [10.5281/zenodo.23933180](https://doi.org/10.5281/zenodo.23933180)

## The problem

Agent UI stacks let an AI agent render a surface, obtain a human's approval, and then execute or settle an action. Across today's protocols, **what the human saw and approved is not bound to what the action dispatches or settles.** An agent — or a prompt injection that reaches it — can display one thing and act on another ("show $100, settle $10,000"; "show *Open the invoice PDF*, open a look-alike host") within a fully conformant surface, with no artifact proving what was shown. It is most acute for the agentic-payments/commerce surfaces these protocols target.

## The primitive

A small, additive, opt-in **consent binding**: at a user-activated action, derive a deterministic digest over the *resolved displayed surface* and the *action's resolved target*, carry it (optionally signed) with the human's decision, and have the verifier recompute the surface it intends to act on and **refuse on mismatch**. This is the "what you see is what you sign" (WYSIWYS) property from trusted-path and hardware-wallet security, applied to agent tool-calls.

## How it maps across the stack

| Protocol | Where the binding lives |
| --- | --- |
| **A2UI** (Google) | a new optional `action.consent` field on the dispatched action |
| **AG-UI** (CopilotKit) | define what the existing `resume[].metadata` "tamper-proof signature" covers — the consent digest over the rendered surface, tied to the `toolCallId`/`interruptId` |
| **MCP-UI** (Microsoft + Shopify) | the consent digest in the `postMessage` result of the user action |
| **Open-JSON-UI** (OpenAI) | the consent digest accompanies the action event |

Because **AG-UI is the runtime that carries A2UI / MCP-UI / Open-JSON-UI**, defining the binding there lets all of them inherit WYSIWYS from one place.

## Alignment with AP2 / FIDO

Google's **Agent Payments Protocol (AP2)** — now continuing in the **FIDO Alliance** — independently converged on this property for payments: its **Trusted Surface** renders mandate content for human authorization, producing a tamper-evident signed mandate, with the agent treated as a potential attacker. AP2 is the *payment-specific* instantiation. What AP2 does not define is **how an agent-generated UI surface becomes a Trusted Surface** — this RFC is that bridge, and it generalizes the property to *every* consequential action, not only payments.

## Status

Draft RFC v0.3 — feedback and per-protocol proposals welcome via issues. A runnable reference implementation of the binding exists and is available for review.

---

*Cite as:* Correa Bezerra, Cleiton Augusto. *WYSIWYS Consent Binding for Agent UIs.* 2026. https://doi.org/10.5281/zenodo.23933180
