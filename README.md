# SIROS Private Data Encryption Specification & Conformance

Normative specification and test suite for the SIROS wallet private data blob format, ensuring cross-client compatibility between TypeScript/JavaScript, Kotlin, and Swift SDK implementations.

## Directory Structure

```
privatedata-spec/
├── SPEC.md                           # Normative specification
├── README.md                         # This file
├── docs/
│   ├── SERIALIZATION.md             # Normative serialization rules
│   ├── INTEGRATION.md               # SDK integration guide
│   └── TROUBLESHOOTING.md           # Common implementation issues
├── test-vectors/
│   ├── README.md                    # Test vector documentation
│   ├── fixtures/
│   │   ├── single-credential-v3.json
│   │   ├── multi-passkey-v3.json
│   │   ├── legacy-v1-symmetric.json
│   │   └── metadata-preservation.json
│   ├── edge-cases/
│   │   ├── max-size-container.json
│   │   ├── unicode-credentials.json
│   │   └── empty-presentations.json
│   └── vectors.jsonl                # Complete test vector suite (line-delimited JSON)
└── conformance/
    ├── README.md                    # Conformance runner guide
    ├── conformance-runner.sh        # Main test orchestrator
    ├── validators/
    │   ├── validate-encryption.sh   # Verify encrypt output
    │   ├── validate-decryption.sh   # Verify decrypt output
    │   └── validate-metadata.sh     # Verify metadata preservation
    └── client-interfaces/
        ├── ts-interface.sh          # wallet-frontend integration
        ├── kotlin-interface.sh      # siros-sdk-kotlin integration
        └── swift-interface.sh       # siros-sdk-swift integration
```

## Quick Start

### 1. Review the Specification

Start with [SPEC.md](SPEC.md) for normative requirements on:
- Blob transport and encoding (tagged binary JSON)
- Container model (mainKey, prfKeys, jwe)
- Cryptographic processing (ECDH, AES-KW, A256GCM)
- WalletStateContainer schema
- Concurrency control (ETag-based)

### 2. Understand Test Vectors

Test vectors in `test-vectors/vectors.jsonl` follow this structure:

```json
{
  "id": "single-credential-v3-001",
  "description": "Single passkey credential with full metadata",
  "inputs": {
    "credentialId": "<base64url>",
    "prfOutput": "<base64url>",
    "hkdfSalt": "<base64url>",
    "hkdfInfo": "eDiplomas PRF"
  },
  "expected": {
    "container": "<serialized-json-bytes>",
    "state": { "S": { "schemaVersion": 3, ... } },
    "mainKeyPublic": "<uncompressed-p256-point>",
    "prfKeyEntry": { "credentialId": "...", ... }
  },
  "sections": ["4", "5", "6"],
  "tags": ["basic", "single-passkey"]
}
```

Each client must:
1. **Encrypt**: Given inputs, produce container matching `expected.container`
2. **Decrypt**: Given container, parse to state matching `expected.state`
3. **Round-trip**: Decrypt → re-encrypt → verify bytes match

### 3. Run Conformance Tests

```bash
# Run all tests against all three clients
./conformance/conformance-runner.sh

# Run tests for specific client
./conformance/conformance-runner.sh --client swift
./conformance/conformance-runner.sh --client kotlin
./conformance/conformance-runner.sh --client ts

# Run specific test vector
./conformance/conformance-runner.sh --vector single-credential-v3-001
```

### 4. Integrate with Your SDK

See [INTEGRATION.md](docs/INTEGRATION.md) for language-specific setup:

- **TypeScript**: Add conformance target to `package.json`
- **Kotlin**: Add conformance task to `build.gradle`
- **Swift**: Add conformance scheme to `Package.swift`

Each SDK must expose two entry points:
- `<client> encrypt <plaintext-json-file>` → outputs encrypted container
- `<client> decrypt <container-json-file>` → outputs decrypted WalletStateContainer

## Normative Requirements

### For All Implementations

- **MUST** implement JWE with `alg=A256GCMKW, enc=A256GCM`
- **MUST** derive HKDF-SHA-256 with normative info string `"eDiplomas PRF"`
- **MUST** preserve full WalletStateContainer (events, lastEventHash, presentations, credentialIssuanceSessions)
- **MUST** preserve credential metadata (format, kid, instanceId, batchId)
- **MUST** preserve keypair DID (not recompute)
- **MUST** use "0" as default for `openidRefreshTokenMaxAgeInSeconds`
- **MUST** support optimistic concurrency via ETag headers

### Test Coverage

All implementations MUST pass:
- ✅ Encrypt canonical vector → byte-for-byte match
- ✅ Decrypt canonical vector → schema match
- ✅ Round-trip: decrypt → re-encrypt → byte comparison
- ✅ Multi-passkey container handling (credential-bound PRF selection)
- ✅ Metadata field preservation
- ✅ Event history preservation
- ✅ Legacy format acceptance (read-only)
- ✅ Edge case handling (unicode, large containers, empty fields)

## CI/CD Integration

Each client repository SHOULD implement conformance gating:

```yaml
# .github/workflows/conformance.yml
conformance:
  runs-on: ubuntu-latest
  steps:
    - uses: actions/checkout@v3
      with:
        repository: sirosfoundation/privatedata-spec
        path: spec
    - name: Run conformance tests
      run: |
        spec/conformance/conformance-runner.sh --client ${{ matrix.client }}
```

Conformance tests are **gating**: PRs that break compatibility cannot merge.

## Reference Implementation

`wallet-frontend` is the source of truth for all behavior:
- Encryption logic: `src/services/keystore.ts`
- State schema: `src/services/WalletStateSchemaVersion3.ts`
- Serialization: `src/util.ts`

## Support

For issues or questions:
1. Check [TROUBLESHOOTING.md](docs/TROUBLESHOOTING.md) for common problems
2. Review [SERIALIZATION.md](docs/SERIALIZATION.md) for normative encoding details
3. Open an issue with reference to the test vector and client version

## License

This specification and test suite are part of the SIROS project.
See LICENSE file for details.
