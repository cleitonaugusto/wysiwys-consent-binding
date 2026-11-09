# Local model of an agent permission chain

A small simulation of the chain [publicly reported](https://www.giskard.ai/knowledge/how-grok-got-prompt-injected-an-x-user-drained-150-000-from-an-ai-wallet) for the May 2026 Grok/Bankr wallet drain, followed by the same chain under default-deny permissions and a [dispatch gate](../../wysiwys-consent-binding.md).

**It is a model.** It contains none of Bankr's or xAI's code, touches no live service, wallet or chain, and every "transfer" is a boolean. Real systems are more complicated than five steps.

```sh
cargo run
cargo test   # checks the digest against the RFC's public conformance vectors
```

| Case | What it shows |
| --- | --- |
| A | Authority inferred from a received asset, and translated text treated as a command: the transfer runs. |
| B | Authority must be signed by the owner, so a received asset grants nothing. |
| C | Transfers are granted, but only to a payee the owner named. |
| D | The capability is valid, but no human approval is bound to the call. |
| E | A human approved the call, then the destination is swapped. |

The gate in `src/main.rs` is written from sections 4.1 to 4.3 of the RFC. Capabilities come from [`cleimesh-aat`](https://github.com/cleitonaugusto/cleimesh/tree/v0.1.5/cleimesh-aat).

Limits: the gate stops autonomous transfers too, because no human approved them, and a verification call can itself be removed unless the executor cannot act without a valid token.

The code in this directory is licensed under MIT OR Apache-2.0. The rest of this repository is CC-BY-4.0.
