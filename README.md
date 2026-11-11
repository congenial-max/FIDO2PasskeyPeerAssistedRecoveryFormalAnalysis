# Peer-Assisted FIDO2 Passkey Recovery: Tamarin Formal Models

Symbolic formal verification models and proof artefacts for an enterprise peer-assisted FIDO2 passkey recovery protocol and its Double-Key Pair hardened variant in the Tamarin prover.

## Repository Contents

- `1_ECS_FIDO2PasskeyRecovery_Protocol.spthy` — baseline single-peer protocol under the eventually compromised server (ECS) model.
- `1_ICS_FIDO2PasskeyRecovery_Protocol.spthy` — baseline single-peer protocol under the initially compromised server (ICS) model.
- `2_ECS_FIDO2PasskeyRecovery_HardenedProtocol.spthy` — hardened protocol (Double-Key Pair, key wrapping, message tags and session nonces) under the ECS model.
- `*_dependencyGraph.spthy` and `*_log.log` — Tamarin proof scripts and run logs (`Tamarin 1.10.0`, `Maude 2.7.1`; invocation commands in `commandsForTerminal.txt`).
- `FIDO2_Passkey_Recovery.oracle`, `tamarin-cleandot.py`, `tamarincleandotlib.py` — goal-ranking oracle and graph helper scripts.
- `tuple_aligned_baseline/` — variant of the two baseline theories aligning the receiver's recovery-message tuple arity with the server's (`Tamarin 1.12.0`, `Maude 3.4`).
