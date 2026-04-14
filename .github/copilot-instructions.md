# Copilot Instructions

## Project Overview

`lambda_argon` is a Go library (single file: `lib.go`) providing an Argon2id password hashing wrapper optimized for AWS Lambda. The public surface is two functions: `Hash` and `Match`. Parameters are locked to a package-level constant tuned for Lambda's default memory/CPU profile (~1 second hash time).

## Commands

```bash
just build    # Cross-compile for Linux/amd64 (the Lambda target)
just test     # Run all tests with verbose output
just format   # Format with gofmt -s -w -l
```

Run a single test or sub-test:
```bash
go test -run TestFunctionName -v ./...
go test -run TestFunctionName/sub_test_name -v ./...
```

The CI config (`travis.yml`) also runs with the race detector: `go test -race .`

## Call Chain

### `Hash(password) → (string, error)`
1. `verifyPasswordRequirements` — rejects if `len < 12` or `len > 128`
2. `createHash(password, lambdaParams)`:
   - `generateRandomBytes(64)` via `crypto/rand` — cryptographically secure salt
   - `argon2.IDKey(password, salt, t=15, m=65536, p=4, keyLen=512)` — derives 512-byte key
   - Both salt and key are base64-encoded with `base64.RawStdEncoding` (no padding)
   - Returns PHC string: `$argon2id$v=19$m=65536,t=15,p=4$<86-char salt>$<683-char key>`

### `Match(password, hash) → (bool, error)`
1. `verifyPasswordRequirements` — same length checks
2. `comparePasswordAndHash` → `checkHash` → `decodeHash`:
   - Splits on `$` — must yield exactly 6 parts → `ErrInvalidHashFormat`
   - Part[1] must be `"argon2id"` → `ErrUnsupportedArgonVersion`
   - Parses `v=%d` and checks against `argon2.Version` (19) → `ErrNonMatchingArgonVersions`
   - Scans `m=%d,t=%d,p=%d` from the hash itself (params are self-describing)
   - Decodes salt and key with `base64.RawStdEncoding.Strict()` — strict mode means any non-base64 character (e.g., from tampering) returns an error rather than silently ignoring it
   - `SaltLength` and `KeyLength` are derived from the actual decoded byte lengths, not from the hash string
3. Recomputes the key with extracted params, then compares using `subtle.ConstantTimeEq` (lengths) and `subtle.ConstantTimeCompare` (contents) to prevent timing attacks

## Lambda-Specific Considerations

- `make build` cross-compiles for `GOOS=linux GOARCH=amd64` — the Lambda execution environment
- The Lambda function needs **64 MB of memory above its normal allocation** for the hashing operation
- With default Lambda settings, `Hash` takes approximately 1 second — this is intentional

## Key Conventions

### Password length enforcement is public-API-only
`verifyPasswordRequirements` is only called from `Hash` and `Match`. The private functions `createHash`, `checkHash`, `comparePasswordAndHash`, and `decodeHash` accept any password length. Tests exploit this to exercise internals with passwords outside the 12–128 range.

### Self-describing hashes enable backward compatibility
`decodeHash` reads `m`, `t`, `p` from the hash string itself, then re-derives `SaltLength` and `KeyLength` from the decoded bytes. A hash created with old params verifies correctly against a newer codebase. The private `checkHash` also returns the decoded `*params`, enabling callers to detect when a stored hash was created with old params and needs rehashing.

### All 5 sentinel errors and their exact triggers

| Error | Trigger |
|---|---|
| `ErrInvalidHashFormat` | `strings.Split(hash, "$")` yields ≠ 6 parts |
| `ErrUnsupportedArgonVersion` | Variant field ≠ `"argon2id"` |
| `ErrNonMatchingArgonVersions` | Numeric version ≠ `argon2.Version` (19) |
| `ErrPasswordTooShort` | `len(password) < 12` |
| `ErrPasswordTooLong` | `len(password) > 128` |

All errors carry the `lambda_argon:` prefix and must be checked with `errors.Is`.

### `params` struct is unexported
Callers cannot configure hashing parameters. The only instance is the package-level `lambdaParams` var. Tests in the same package (`package lambda_argon`) reference `lambdaParams` directly.

### Base64 encoding
- **Encoding** (in `createHash`): `base64.RawStdEncoding` — no padding
- **Decoding** (in `decodeHash`): `base64.RawStdEncoding.Strict()` — no padding, rejects invalid characters

### Test conventions
- Test file is `package lambda_argon` (not `_test`) for white-box access to private functions
- Assertions use `github.com/jgroeneveld/trial/assert`
- Sub-test names follow `t.Run("verify <what>", ...)` pattern
- Hard-coded hash/password pairs are included to guard against regressions in `decodeHash` and `checkHash`
