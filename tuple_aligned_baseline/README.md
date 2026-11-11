# Tuple-Aligned Baseline Models

In the original baseline theories (`1_ECS_FIDO2PasskeyRecovery_Protocol.spthy` and `1_ICS_FIDO2PasskeyRecovery_Protocol.spthy`), `Server_Sends_Recovery_Data` emits 3-tuples `<'recovery_fido', nonce, encrypted_fido_key>` and `<'recovery_package', nonce, sk2>`, whereas `Initiator_Receives_Recovery_Data` matched 2-tuples without the `nonce` field.

In this subfolder, `Initiator_Receives_Recovery_Data` is updated to match the 3-tuples `<'recovery_fido', rcv_nonce1, encrypted_fido_key>` and `<'recovery_package', rcv_nonce2, sk2>` directly (leaving the received nonces unchecked, as intended in the unhardened baseline).

**Effect on results:** None. Re-running both theories with `tamarin-prover` produces the exact same verdicts and proof-step counts as the original baseline (`19 / 36 / 87 / 21` steps under ECS; `12 / 8 / 12 / 13` steps under ICS).
