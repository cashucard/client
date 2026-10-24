# @lnflash/cashu-client

Cashu NFC card client library for crypto primitives and [Nutshell](https://github.com/cashubtc/nutshell) mint HTTP requests.

Part of the [Cashu-First Flash Cards](https://github.com/cashucard/javacard)
project. The card holds Bitcoin-denominated ecash on its own chip and supports
spending by tap at the point of sale, without a phone, account or internet connection.

This library is the **host half**. The card stores proofs and signs. The host
library handles blinding, mint communication and verification.

| To understand… | Read |
|---|---|
| Why the project exists | [VISION](https://github.com/cashucard/javacard/blob/main/docs/VISION.md) |
| How the pieces fit | [ARCHITECTURE](https://github.com/cashucard/javacard/blob/main/docs/ARCHITECTURE.md) |
| Why the API refuses things | [DECISIONS](https://github.com/cashucard/javacard/blob/main/docs/DECISIONS.md) |
| What an attacker can do | [SECURITY-MODEL](https://github.com/cashucard/javacard/blob/main/docs/SECURITY-MODEL.md) |
| A term you don't know | [GLOSSARY](https://github.com/cashucard/javacard/blob/main/docs/GLOSSARY.md) |

## What's in here

| Module | Description |
|--------|-------------|
| `crypto` | NUT-00 `hash_to_curve`, NUT-03 BDHKE blinding/unblinding, NUT-10 P2PK secret serialization, denomination splitting |
| `card` | Reconstructs a spendable proof from a card's 78-byte slot layout using NUT-10 P2PK secret recovery |
| `cardFile` | Defines the interchange format shared by this library and the Python card driver |
| `mint` | Nutshell HTTP client for NUT-01 keysets and NUT-04 mint quotes and proof issuance |
| `witness` | Builds NUT-11 P2PK witnesses from the message a card signs, attaches the signature and verifies the result |
| `melt` | Handles NUT-05 melt quotes, execution and proof selection for card redemption |
| `swap` | Handles NUT-03 change, receiving and unlocking of card-locked proofs |
| `state` | Checks NUT-07 proof state for double spends |
| `dleq` | Verifies offline that a mint signed with its published key using NUT-12 discrete-log equality |
| `http` | Shared URL sanitising and response limits for every mint call |
| `types` | Shared TypeScript types (`CashuProof`, `CashuMeltQuote`, `CashuProofState`, …) |
| `errors` | Typed error classes with numeric codes |

### The card lifecycle

The library now supports the full round trip for P2PK-locked card proofs. Earlier
versions could fund a card but could not produce the *unlock* needed to spend it.

| Stage | Calls |
|---|---|
| **Load** a card | `requestMintQuote` → pay → `mintProofs` → `unblindSignature` + `proofDLEQFromBlindSignature` → `serializeCardFile` → cardctl `LOAD_PROOF` |
| **Verify** what's on it | `verifyProofDLEQ` (offline) or `checkProofStates` (online) |
| **Spend** at a terminal | `GET_PROOF` → `parseCardFile` → `reconstructProofsFromCard` → `requestMeltQuote` → `selectProofsForMelt` → card signs `p2pkMessageToSign` → `attachP2PKWitness` → `meltProofs` |
| **Make change** | `swapProofs` |

#### Crossing to the card: the card file

The card driver ([`tools/cardctl`](https://github.com/cashucard/javacard))
is dependency-free Python so it runs anywhere a reader does. This library is
TypeScript, and proofs cross the boundary as a file. The `cardFile` module owns
the schema, and both sides validate against it. A field mismatch fails before
the card can burn a slot with `SPEND_PROOF`.

```jsonc
{
  "version": 1,
  "mint": "https://forge.flashapp.me",
  "unit": "sat",
  "cardPubkey": "032994631ef9a4ba5b0db2f44b4d0d8a4b0eec49bed16091c23c171a8c553a03da",
  "slots": [
    {
      "keysetId": "0059534ce0bfa19a",
      "amount": 8,
      "nonce": "916c21b8c67da71e9d02f4e3adc6f30700c152e01a07ae30e3bcc6b55b0c9e5e",
      "C": "02c6047f9441ed7d6d3045406e95c07cd85c778e4b8cef3ca7abac09b95c709ee5",
      "spent": false
    }
  ],
  "note": "loaded at till 2"
}
```

```typescript
import {
  serializeCardFile,
  parseCardFile,
  cardFileTotal,
  reconstructProofsFromCard,
  sanitizeMintUrl,
} from "@lnflash/cashu-client"

// mint → card: hand this to cardctl to load. `blindingData` is what
// createBlindedMessage returned. It carries the 32-byte nonce the card
// stores. The proof contains no other copy of that nonce.
const json = serializeCardFile({
  mint: MINT_URL,
  unit: "sat",
  cardPubkey,
  slots: proofs.map((p, i) => ({
    keysetId: p.id,
    amount: p.amount,
    nonce: blindingData[i].nonce, // the 32-byte nonce; P2PK secret JSON is ~150 bytes
    C: p.C,
    spent: false, // required; the writer refuses a true
  })),
  note: "loaded at till 2",
})

// card → mint: what cardctl dumped, ready to spend. Compare against the
// *sanitised* URL. `card.mint` comes back canonical, and a raw constant with
// a trailing slash would reject a card from the right mint.
const card = parseCardFile(await fs.readFile("card.json", "utf8"))
if (card.mint !== sanitizeMintUrl(MINT_URL)) throw new Error("card is from a different mint")
const unspent = card.slots.filter(s => !s.spent)
const spendable = reconstructProofsFromCard(unspent, card.cardPubkey)
console.log(`${cardFileTotal(card)} ${card.unit} across ${unspent.length} slots`)
```

**The wire shape uses the card's vocabulary.** It stores `nonce` and a
16-character hex `keysetId`. A `secret` field could carry ~150 bytes of P2PK
JSON into a field that holds 32 bytes, so the parser rejects that field name.

**`serializeCardFile` is the only gate in the mint → card direction.** cardctl
does no curve math, so whatever this writes reaches the card. It therefore
round-trips the file through `parseCardFile` *and* through the redeem path
(`reconstructProofsFromCard`, result discarded) before returning: what it
writes is, by construction, what the spend path reads back. An on-prefix but
off-curve `C` would otherwise load fine, burn the slot on `SPEND_PROOF`, and be
rejected by the mint as an invalid proof.

**Validation preserves the file's meaning.** The parser rejects unknown fields
(bump `version` when adding a field), notes that are not strings or exceed 512
characters, non-v0 or half-length `keysetId` values, invalid amounts, missing
or non-boolean `spent` values, and mint URLs rejected by the HTTP sanitiser.
It also rejects duplicate `C` values and duplicate `nonce` values across slots.
Duplicate `C` values repeat the same signature. Duplicate nonces reconstruct
to one proof because the secret derives from the nonce and card key alone, even
when amounts and `C` values differ. Either duplicate can make the second slot
burn on `SPEND_PROOF` after the first proof redeems, while `cardFileTotal` had
counted both. Amounts of `2^32` or more exceed the four-byte `LOAD_PROOF` field.
The `mint` and `unit` fields return in canonical form. Pass `MINT_URL` through
`sanitizeMintUrl` before comparing it with `card.mint`.

**`spent` travels from card to mint.** `parseCardFile` preserves the bit from a
card dump. `serializeCardFile` rejects a `spent: true` slot because `LOAD_PROOF`
has no spent bit; loading a spent proof would make it spendable again. To top up
a card, dump it, remove spent slots, append new slots and serialize.
`cardFileTotal` counts only unspent slots.

Integration considerations:

**Reconstruct proofs from card slots.** A NUT-10 P2PK secret is ~150 bytes of
JSON, while a slot is 78 bytes total. The card stores the 32-byte *nonce*;
`reconstructProofsFromCard` rebuilds the secret from the nonce, the card's public
key and the fixed tag set. Without that step `GET_PROOF` gives you bytes and no
`CashuProof` to hand `selectProofsForMelt`. `reconstructProofsFromCard` rejects
malformed slots because the card may already have burned them on `SPEND_PROOF`.
Pass `{skipInvalid: true}` to get `{proofs, failures}` and read the rest of a
card around one corrupt slot. Check each failure's class first. A bad *card key* fails every slot, so
abort card processing when the card key is bad.

**Melt and swap consume their inputs.** After the mint accepts them, a lost
response requires checking `getMeltQuoteState` or `checkProofStates` before
taking further action. Do not retry the request.

**`meltProofs` rejects a proof whose P2PK witness is missing or invalid**,
because the card has already marked its slots spent when a terminal assembles a
melt. Local checks protect the proof after the card marks its slots spent. The
local check honours the secret's `sigflag` and `n_sigs` tags. It rejects tags it does
not implement or cannot parse, because a dropped tag is indistinguishable from
an absent one. This check matches the mint's requirements and protects the
slots from a rejected melt. `findUnsignedProofs` reports NUT-10 secrets the
library cannot verify, such as HTLCs or P2PK secrets with malformed bodies.

**The mint keeps overpayment when you omit change outputs.** `changeOutputs` on
`meltProofs` is optional, so `selectProofsForMelt` minimises the overpay rather
than taking the largest proof that fits. If the mint charges
a NUT-02 `input_fee_ppk`, pass it to `selectProofsForMelt` and `swapProofs`,
otherwise every request is short by exactly the fee and gets rejected. The rate
is declared *per keyset*, so once a rotation leaves a card holding proofs from
two of them, pass a `{[keysetId]: input_fee_ppk}` map instead of one number; a
a single rate applied to a mixed set is rejected to prevent mispricing.

**Error-returning calls use a `T | CashuMintError` result.** These functions
return errors instead of throwing, so check the result before use.
`allProofsUnspent` in particular returns `"UNSPENT" | "NOT_UNSPENT" |
CashuMintError` instead of a boolean. Since an error is a truthy object,
`if (await allProofsUnspent(...))` could read a mint timeout as "safe to accept,"
which reverses the double-spend check's purpose.

## Install

```bash
# yarn v1 (Git dependency)
yarn add github:cashucard/client#v0.5.0

# or via npm
npm install @lnflash/cashu-client
```

## Usage

```typescript
import {
  splitIntoDenominations,
  createBlindedMessage,
  unblindSignature,
  proofDLEQFromBlindSignature,
  verifyProofDLEQ,
  requestMintQuote,
  getMintKeysets,
  getMintKeyset,
  mintProofs,
} from "@lnflash/cashu-client"

const MINT_URL = "https://forge.flashapp.me"

// 1. Request a mint quote
const quote = await requestMintQuote(MINT_URL, 100, "usd") // 100 cents
if (quote instanceof Error) throw quote

// 2. Pay quote.paymentRequest (bolt11 invoice) externally

// 3. Fetch keyset keys
const keysets = await getMintKeysets(MINT_URL)
const usdKeyset = keysets.find(ks => ks.unit === "usd" && ks.active)
const keysetDetail = await getMintKeyset(MINT_URL, usdKeyset.id)

// 4. Build P2PK-locked blind messages for the card
const denominations = splitIntoDenominations(100)
const blindingData = denominations.map(amount =>
  createBlindedMessage(usdKeyset.id, amount, cardPubkeyHex)
)
const blindedMessages = blindingData.map((bd, i) => ({
  id: usdKeyset.id, amount: denominations[i], B_: bd.B_
}))

// 5. Mint proofs (with optional retry logic on "quote not paid")
const sigs = await mintProofs(MINT_URL, quote.quoteId, blindedMessages)

// 6. Unblind signatures → final proofs, carrying the mint's DLEQ across
//    (proofDLEQFromBlindSignature pairs the mint's e/s with the blinding
//    factor r, which is what makes the offline check in step 7 possible)
const proofs = sigs.map((sig, i) => ({
  id: sig.id,
  amount: sig.amount,
  secret: blindingData[i].secretStr,
  C: unblindSignature(sig.C_, blindingData[i].r, keysetDetail.keys[String(sig.amount)]),
  dleq: proofDLEQFromBlindSignature(sig, blindingData[i].r),
}))

// 7. Check available DLEQ proofs locally with the published mint key.
//    Undefined dleq means the mint emits none; that is a policy call for you.
const mintKey = amount => keysetDetail.keys[String(amount)]
const verified = proofs.every(p => !p.dleq || verifyProofDLEQ(p, mintKey(p.amount)))
```

## Compatibility

**0.4.0 changes `buildP2PKSecret` output for upper-case hex input.** It now
lower-cases the nonce and the card pubkey, so the serialization has one canonical
form. This version cannot reconstruct proofs minted by **0.3.0 or earlier with
an upper-case `cardPubkey`**. The mint commits to the secret as UTF-8 bytes using
`Y = hash_to_curve(secret)`. The canonical secret therefore produces a different
proof that the mint has never signed.

Cards funded that way stay redeemable via the escape hatch:

```typescript
// Try canonical first; fall back only if the mint rejects the proof as unknown.
const legacy = reconstructProofFromCard(slot, cardPubkey, {legacyHexCase: true})
```

The hatch freezes the case of `data` only. Pre-0.4.0 `createBlindedMessage`
generated the nonce in lower-case hex without reading one from a card. The mint
could only have seen the card key's original casing. Freezing the nonce's case
as well would create a secret no released version minted. The mint would reject
it after `SPEND_PROOF` burned the slot.

Use `legacyHexCase` only to spend proofs locked by the old code. See
[CHANGELOG.md](CHANGELOG.md).

## Spec

- [NUT-XX: Cashu NFC Card Protocol](https://github.com/cashucard/javacard/blob/main/spec/NUT-XX.md)
- [JavaCard applet](https://github.com/cashucard/javacard)

## License

MIT
