# SDK Integration Guide

This guide describes how to wire TypeScript, Kotlin, and Swift clients to the conformance runner.

## Required Commands Per Client

Each client repository MUST expose:

1. Encrypt command
- Input: plaintext state JSON + PRF input JSON
- Output: encrypted container JSON to stdout

2. Decrypt command
- Input: encrypted container JSON
- Output: decrypted WalletStateContainer JSON to stdout

## Environment Variables Used by Runner

- WALLET_FRONTEND_PATH
- SIROS_SDK_KOTLIN_PATH
- SIROS_SDK_SWIFT_PATH

## Minimum Compatibility Contract

- Preserve events and lastEventHash
- Preserve S.credentials metadata fields
- Preserve S.keypairs did field
- Preserve S.presentations and S.credentialIssuanceSessions
- Use openidRefreshTokenMaxAgeInSeconds="0" default semantics
- Select PRF entry by credentialId match

## CI Recommendation

Add conformance runner invocation as a blocking job for PR merges.
