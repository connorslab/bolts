# Sideflash: Authenticated Hybrid Addresses for BOLT12

**Status:** draft for discussion. **BOLT number:** unassigned.

This proposal describes an experimental wallet-level address envelope for bitcoin
(XBT). It requests feedback on scope, encoding and authentication. It does not
request a feature-bit allocation or claim production readiness.

## Table of Contents

- [Motivation](#motivation)
- [Scope](#scope)
- [Address envelope](#address-envelope)
- [Authentication](#authentication)
- [Writer requirements](#writer-requirements)
- [Reader requirements](#reader-requirements)
- [Lightning payment behavior](#lightning-payment-behavior)
- [Cross-compatibility profile](#cross-compatibility-profile)
- [Security considerations](#security-considerations)
- [Implementation and test evidence](#implementation-and-test-evidence)
- [Open questions](#open-questions)

## Motivation

A recipient can have a native payment destination and a reusable Lightning
offer. Publishing them separately leaves each wallet to decide how to associate
them and which route to use. Sideflash binds both destinations in one signed
address. A compatible Lightning wallet can extract the offer without running a
wallet for the native payment system or contacting an address directory.

The proposed prefix is `sfl`; the `1` in `sfl1...` is the separator, not a version.
A wallet that supports the native route can select it when applicable. Otherwise,
it can use the authenticated offer under the delivery requirements of the
supported cross-compatibility profile.

## Scope

This proposal covers representation, destination binding and BOLT12 extraction.
It does not change channel transactions, routing, payment hashes or BOLT12's
invoice negotiation. No new peer message or Lightning feature bit is allocated.
The address's own version and required-feature fields are separate from BOLT9
feature fields.

The external system's settlement, inventory and recovery construction is outside
this address specification. A valid envelope does not establish those guarantees.
Implementations must distinguish address support from payment-delivery support.

## Address envelope

The current development profile uses Bech32m with HRP `sfl` around a deterministic
CBOR map. It limits text to 1023 ASCII characters and payloads to 633 bytes. This
exceeds the original 90-character address convention but remains within the
Bech32m code length. Encodings longer than the limit require a future profile;
they must not be truncated or silently converted into online lookups.

The following table records the implemented development encoding, not an assigned
stable format. CBOR uses definite lengths, shortest integer/length forms and
ascending unsigned integer keys. Duplicate keys, extra keys and trailing data
are invalid. Amounts are not encoded in this envelope.

| Key | CBOR type | Development meaning |
| --- | --- | --- |
| 0 | unsigned integer | Version, currently 0 |
| 1 | unsigned integer | Required envelope features, currently 0 |
| 2 | byte string, 32 bytes | Genesis hash in Lightning chain-hash byte order |
| 3 | byte string, 32 bytes | Explicit fork/profile discriminator |
| 4 | byte string, 33 bytes | Compressed destination service identity key |
| 5 | byte string | Native destination; see cross-compatibility profile |
| 6 | byte string | Exact decoded BOLT12 offer TLV bytes |
| 7 | unsigned integer | Binding revision |
| 8 | unsigned integer | Inclusive validity start, Unix seconds |
| 9 | unsigned integer | Exclusive expiry, Unix seconds |
| 10 | byte string, 64 bytes | Recipient BIP340 authorization |
| 11 | byte string, 64 bytes | Service BIP340 acknowledgment |

The development discriminator is SHA256 of the ASCII bytes
`Sideflash/XBT/blake2b-unified-sighash/v0`, without a terminating NUL. Mainnet and
regtest use their respective genesis hashes with this discriminator. Other test
networks are not supported by this profile. This is a proposed application-level
identifier, not a globally assigned chain identifier or a transaction sighash.

## Authentication

Let `C(n)` be the canonical CBOR map containing fields 0 through n, with its map
length encoded accordingly. The development signature digests are:

```
recipient_digest = SHA256("Sideflash/recipient/v0" || 0x00 || C(9))
service_digest   = SHA256("Sideflash/server/v0"    || 0x00 || C(10))
```

Field 10 signs `recipient_digest` under the native destination's receiving
authority. Field 11 signs `service_digest` under field 4's x-only key. The service
acknowledgment therefore commits to the recipient signature as well as the
destinations. The service does not need the recipient's secret key.

These are application signatures, not BOLT12 signature fields. They do not
replace any BOLT12 signature or invoice checks. Whether to adopt BIP340-style
tagged hashes instead of these explicit domain prefixes remains an open design
question before a stable encoding is selected.

## Writer requirements

An address writer:

- MUST set the version and required-envelope-feature fields to 0 for this profile.
- MUST encode the fields using the canonical rules above and respect both limits.
- MUST include a structurally valid offer with the correct chain and required
  network features under [BOLT12](../12-offer-encoding.md).
- MUST obtain authorization from the native receiving authority over both routes.
- MUST obtain the destination service acknowledgment before publishing an address.
- MUST NOT publish an address whose validity interval is empty or already expired.
- MUST produce either entirely lowercase or entirely uppercase text.
- SHOULD produce uppercase QR data and lowercase text for copying.
- MUST NOT advertise payment-delivery capabilities that its receiver cannot provide.

## Reader requirements

An address reader:

- MUST reject input above the text limit before decoding and above the payload
  limit before parsing nested fields or invoking an offer parser.
- MUST reject an incorrect HRP/checksum, mixed case, invalid padding, invalid
  field types or lengths, noncanonical CBOR, duplicate/extra keys or trailing data.
- MUST reject unsupported versions, required features or native receiving policies.
- MUST reject a chain/profile mismatch, expired or not-yet-valid binding.
- MUST reject an invalid recipient signature or service acknowledgment.
- MUST derive the recipient verification key from the supported native policy;
  an arbitrary unattested key supplied alongside the address is insufficient.
- MUST verify the full service identity against authenticated recipient/service
  context; a short native fingerprint alone is insufficient.
- MUST validate the offer under BOLT12, including its chain, expiry and features.
- MUST preserve the exact signed offer TLV bytes during extraction.
- MUST NOT treat successful decoding as proof of registration, freshness, liquidity
  or successful delivery in another payment system.

A reader can decode the offer locally, but verification of current revocation or
availability may require communication with the receiving service. The initial
wire profile contains no resolver URL, redirect or delegated binding authority.

## Lightning payment behavior

After successful verification, a Lightning wallet reconstructs an ordinary
`lno1` offer from field 6 using BOLT12's encoding, without adding the envelope's
Bech32m checksum to the offer. It requests and validates a fresh invoice through
the normal BOLT12 flow. Existing chain, feature, amount, expiry and signature
requirements remain in force. In particular, the envelope's discriminator is
not a substitute for this repository's `option_blake2b` checks.

A payer MUST authorize the amount and fee limits before payment. It MUST NOT
automatically pay a replacement invoice or switch routes while a previous
payment's outcome is unresolved. Lightning settlement proves payment of the
invoice, not the external system's delivery obligation.

A receiving service that promises external-system delivery MUST provide the
negotiated delivery guarantees before accepting irreversible settlement. If a
profile needs additional payer-side preparation, a generic offer extractor is
not sufficient and MUST NOT advertise that profile as supported. Whether all
preparation can occur on the receiving side is an explicit implementation and
security requirement, not a property obtained from the address signatures.

## Cross-compatibility profile

The current prototype uses an Ark pubkey destination as its native route. Field 5
contains a network byte (0 for mainnet, 1 for regtest), a native policy-address
version byte (1), and the canonical native address payload without its HRP,
five-bit version or checksum. That payload contains the service fingerprint,
length-prefixed receiving policy and delivery information. A parser must bound
each nested length by the remaining field before allocation.

For this development profile, the reader derives the recipient key from the
pubkey policy, checks its network and service fingerprint against the full
identity in the envelope, and rejects all other policies. The experimental
native codec is documented in the [reference implementation](https://github.com/connorslab/paperclip-asp/blob/9b5ddc3/lib/src/address.rs).
This external codec dependency needs a frozen normative definition before
standardization; this draft is not yet a standalone native-policy specification.

If the payer's Ark service identity matches the full destination identity, the
wallet can use the native transfer path. Otherwise, it can use Lightning only
under a supported delivery profile. Recipient inventory, conditional VTXOs,
preimage ownership and emergency recovery belong in a companion specification.
The initial target requires an online recipient. Offline delivery and delegated
signing are not capabilities of the current prototype.

## Security considerations

The checksum detects transcription errors, not recipient substitution. Both
signatures can be valid on an attacker's replacement address. Applications must
authenticate the recipient channel or otherwise pin the intended destination.

An embedded offer can become unavailable or revoked before envelope expiry.
Revision fields alone do not prove freshness or prevent a malicious service from
equivocating. Refusal must not cause fallback to an unrelated offer. A future
resolution mechanism needs authenticated freshness and explicit rotation rules.

Reusable addresses are linkable. Blinded offer paths do not conceal the signed
service identity or native destination from a reader. The current profile is not
an anonymity improvement over publishing those fields separately.

Input size limits and strict parsing apply before cryptography. Implementations
should rate-limit invoice requests and avoid third-party address-decoding
services. This proposal does not authorize requests to arbitrary network URLs.

## Implementation and test evidence

[JSON fixtures](sideflash/vectors.json) contain frozen mainnet and regtest
development addresses. They use known test keys and expired validity intervals;
never send money to them. `verify_at` specifies the historical test clock.

The [prototype and Python fixture checker](https://github.com/connorslab/sideflash)
agree on canonical bytes, signatures and extraction. The checker uses fixture
keys as expected authorities and is not a complete independent policy parser.
Synthetic QR decoding passes for both case forms at M and Q correction levels;
physical camera testing remains outstanding.

Existing payment/recovery primitives have separate isolated test evidence. They
do not establish an end-to-end Sideflash payment. No two independently developed
complete Sideflash implementations or independent security audit are claimed.

## Open questions

1. Is a standalone proposal here the right scope, or should the address envelope
   remain a companion specification with only BOLT12 integration guidance here?
2. Should the stable envelope use Lightning TLV encoding rather than CBOR?
3. Should authentication use tagged hashes and a more general recipient-policy
   identifier? Any change needs a new development profile and vectors.
4. How should full service identity be authenticated in minimal Lightning clients?
5. Can a compact format support larger blinded offers without mandatory lookups?
6. What fresh-status and receiver-managed-delivery requirements are necessary
   before a wallet can advertise safe cross-compatibility?

No BOLT number or new Lightning feature bits are assigned by this draft.
