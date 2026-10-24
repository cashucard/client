<a id="cashu-client--audit-report"></a>

# cashu-client: Audit Report

**Package:** `@lnflash/cashu-client` v0.2.0, a TypeScript Cashu (ecash) BDHKE crypto library and Nutshell mint HTTP client
**Date:** 2026-06-22
**Reviewed:** `main` after the `fix/protocol-hardening` merge
**Method:** Four reviews covered crypto correctness, mint-client security, API quality, and dependencies, builds and tests. `hash_to_curve` was independently verified against NUT-00 vectors. The build and tests ran.

This report records the package as reviewed on 2026-06-22. Follow-ups below describe the state at that time.

> This PR fixed the highest-value, well-contained findings. DLEQ verification and the `@noble/curves` migration remained as follow-ups for focused review.

## Overall verdict

**The cryptographic primitives are sound. The trust model needed stronger checks.** The BDHKE math is correct, and spec test vectors now pin the `hash_to_curve` endianness fix. The client previously lacked request and response binding and DLEQ verification. This PR closes the contained fund-safety gaps and hardens inputs. DLEQ verification remained a follow-up.

<a id="-verified-correct-no-change-needed"></a>
<a id="verified-correct-no-change-needed"></a>

## Review findings

- **`hash_to_curve` follows NUT-00**: its domain separator, SHA256 loop, `0x02` prefix and **little-endian** counter match the published vectors in `test/crypto.spec.ts`. The endianness fix is pinned by those tests.
- **Blinding/unblinding math** (`B'=Y+rG`, `C=C'−rK`) and **point negation** are exact.
- **Randomness**: blinding scalar `r` and nonce use CSPRNG with `0<r<n` rejection, fresh per output.
- **NUT-10 secret via `JSON.stringify`** (no injection); axios timeout + `maxRedirects:0`; path params sanitized; no logging / no secret leakage.
- **Tooling**: `strict` on; zero `any`/`@ts-ignore`/`console`/TODO; **0 production-dependency vulnerabilities**; clean supply chain (WASM crypto dependency; native dependencies and install scripts are absent).

<a id="-fixed-in-this-pr"></a>
<a id="fixed-in-this-pr"></a>

## Implemented fixes

| Finding | Severity | Fix |
|---------|----------|-----|
| `splitIntoDenominations` dropped every bit above 2¹⁵ (`1<<bit` is 32-bit; loop was 0–15) → wrong split / **fund loss** for amounts ≥ 65536 | **High** | Decompose with `2**bit` over the full safe-integer range (`crypto.ts`). |
| `mintProofs` trusted mint responses positionally and did not check that returned `amount`/keyset `id` matched the requested output, which could produce proofs worth less than paid or unspendable proofs | **High** | Bind each signature to its output: assert `id` + `amount` match and `C_` is a valid compressed point; reject the batch otherwise (`mint.ts`). |
| `getMintKeyset` returned keys without verifying the keyset `id` matched the request → wrong `K` during unblinding → unspendable proofs | **High** | Assert `ks.id === keysetId` and `keys` is a non-empty object (`mint.ts`). |
| `sanitizeMintUrl` used string-prefix matching → `https://…@internal`, cloud-metadata, link-local hosts passed (**SSRF**) | **High** | Parse with the `URL` API; enforce `https:` (loopback may use `http:`); reject embedded credentials; block metadata/link-local hosts (`mint.ts`). |
| `requestMintQuote` sent `amount`/`unit` unvalidated (negative/0/NaN/fractional) | High | Validate `amount` is a positive integer and `unit` matches `^[a-z]{3,4}$` (`mint.ts`). |
| `unblindSignature` didn't range-check `r` (exported; zero `r` → `C=C_`, silent no-op) | Medium | Validate `secp.isPrivate(r)` after the point checks (`crypto.ts`). |
| No response-size cap → hostile mint could exhaust memory | Medium | `maxContentLength`/`maxBodyLength` = 1 MB on axios (`mint.ts`). |
| NUT-00 known-answer tests were absent, so the suite proved self-consistency without checking interoperability | Low | Added the official `0x00*32` and `0x00..01` `hash_to_curve` vectors (`test/crypto.spec.ts`). |
| CI lacked a guard to keep committed `dist/` in sync | Low | CI step `git diff --exit-code dist` fails PRs where the committed build drifts. |
| No `engines`/`.nvmrc` | Low | Added `engines: node>=18` + `.nvmrc`. |

All verified: `npm run build` clean, **31/31 tests pass** (29 + 2 new spec vectors), committed `dist/` rebuilt and in sync.

<a id="-remaining-follow-ups-documented-not-in-this-pr"></a>
<a id="remaining-follow-ups-documented-not-in-this-pr"></a>

## Remaining follow-ups as of 2026-06-22

- **NUT-12 DLEQ verification** *(High)*: The client did not prove that the mint signed with its advertised key. A malicious mint could use this gap as a per-user-key **tagging/deanonymization oracle**. The work requires `e,s` proof handling and warrants its own PR. Amount and keyset binding reduce fund-loss risk while privacy verification remains outstanding.
- **Migrate `tiny-secp256k1` → `@noble/curves`** *(Medium)*: The audited (Cure53/ToB), constant-time, pure-JS library avoids a WASM loader and was the ecosystem standard. A migration needs its own PR and re-verification against the spec-vector tests.
- **Error-handling convention** *(Medium)*: `mint.ts` returns `T | CashuMintError` unions while `crypto.ts` throws. The README examples omit narrowing. Unify the convention with typed errors and map the mint's NUT error `code` into the error subclasses, which were not thrown at the time.
- **Runtime response validation** *(Medium)*: Replace remaining unchecked `as` casts of mint JSON (quote shape, `getMintKeysets` elements) with a validator such as `zod`. Enable `noUncheckedIndexedAccess`.
- **Packaging** *(Low)*: Add an `exports` map, consider a dual ESM/CJS build and `sideEffects:false`, and update the README install pin from `#v0.1.0`.
- **Endpoint coverage**: `swap`, `melt` and `info` endpoints were not implemented at the time, so the library could not spend or receive ecash on its own.

<a id="bottom-line"></a>

## Summary

The BDHKE primitives are correct and pinned by spec vectors. This PR closes the contained fund-safety and input-hardening gaps, including denomination overflow, request/response binding, keyset-ID binding, SSRF and amount validation. At the time of this report, **DLEQ verification** remained a dedicated follow-up. Tokens had integrity checks for amount and keyset, without privacy verification against a hostile mint.

---

*Audit generated with Claude Code.*
