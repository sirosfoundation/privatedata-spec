# Troubleshooting

## Encryption output mismatch

- Ensure compact JSON serialization (no pretty printing)
- Ensure base64url encoding without padding
- Ensure canonical top-level key order: mainKey, prfKeys, jwe

## Decryption fails with multi-passkey container

- Verify lookup is by credentialId match
- Verify hkdfSalt/hkdfInfo are read from selected PRF entry

## Metadata loss after round-trip

- Preserve original fields from decrypted state
- Do not synthesize credential metadata defaults if source has values
- Preserve keypair.did and do not recompute

## Settings mismatch

- openidRefreshTokenMaxAgeInSeconds should preserve existing value
- If default required, use string "0"

## Array shape mismatch

- presentations and credentialIssuanceSessions must stay arrays
- Avoid converting empty arrays to null or omitting them
