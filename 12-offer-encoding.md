# BOLT #12: Flexible Protocol for Lightning Payments

# Table of Contents

  * [Limitations of BOLT 11](#limitations-of-bolt-11)
  * [Payment Flow Scenarios](#payment-flow-scenarios)
  * [Encoding](#encoding)
  * [Signature calculation](#signature-calculation)
  * [Offers](#offers)
  * [Invoice Requests](#invoice-requests)
  * [Invoices](#invoices)
  * [Invoice Errors](#invoice-errors)

# Limitations of BOLT 11

The BOLT 11 invoice format has proven popular but has several
limitations:

1. The entangling of bech32 encoding makes it awkward to send
   in other forms (e.g. inside the lightning network itself).
2. The signature applying to the entire invoice makes it impossible
   to prove an invoice without revealing its entirety.
3. Fields cannot generally be extracted for external use: the `h`
   field was a boutique extraction of the `d` field only.
4. The lack of the 'it's OK to be odd' rule makes backward compatibility
   harder.
5. The 'human-readable' idea of separating amounts proved fraught:
   `p` was often mishandled, and amounts in pico-bitcoin are harder
   than the modern satoshi-based counting.
6. Developers found the bech32 encoding to have an issue with extensions,
   which means we want to replace or discard it anyway.
7. The `payment_secret` designed to prevent probing by other nodes in
   the path was only useful if the invoice remained private between the
   payer and payee.
8. Invoices must be given per user and are actively dangerous if two
   payment attempts are made for the same user.


# Payment Flow Scenarios

Here we use "user" as shorthand for the individual user's lightning
node and "merchant" as the shorthand for the node of someone who is
selling or has sold something.

There are two basic payment flows supported by BOLT 12:

The general user-pays-merchant flow is:
1. A merchant publishes an *offer* ("send me money"), such as on a web page or a QR code.
2. Every user requests a unique *invoice* over the lightning network
   using an *invoice_request* message.
3. The merchant replies with the *invoice*.
4. The user makes a payment to the merchant indicated by the invoice.

The merchant-pays-user flow (e.g. ATM or refund):
1. The merchant provides a user-specific *offer* ("take my money") on a web page or QR code
   with an amount (for a refund, also a reference to the to-be-refunded
   invoice).
2. The user sends an *invoice* for the amount in the *offer*
3. The merchant makes a payment to the user indicated by the invoice.

## Payment Proofs and Payer Proofs

Note that the normal lightning "proof of payment" can only demonstrate that an
invoice was paid (by showing the preimage of the `payment_hash`), not who paid
it.  The merchant can claim an invoice was paid, and once revealed, anyone can
claim they paid the invoice, too.[1]

Providing a key in *invoice_request* allows a user to prove that they were the one
to request the invoice.  In addition, the Merkle construction of the BOLT 12
invoice signature allows the user to reveal invoice fields in case
of a dispute selectively.

# Encoding

Each of the forms documented here are in
[TLV](01-messaging.md#type-length-value-format) format.

The supported ASCII encoding is the human-readable prefix, followed by a
`1`, followed by a bech32-style data string of the TLVs in order,
optionally interspersed with `+` (for indicating additional data is to
come).  There is no checksum, unlike bech32m.

## Requirements

Readers of a bolt12 string:
- if it encounters a `+` followed by zero or more whitespace characters between 
  two bech32 characters:
  - MUST remove the `+` and whitespace.

## Rationale

The use of bech32 is arbitrary but already exists in the bitcoin
world.  We currently omit the six-character trailing checksum: QR
codes have their own checksums anyway, and errors don't result in loss
of funds, simply an invalid offer (or inability to parse).

The use of `+` (which is ignored) allows use over limited
text fields like Twitter:

```
lno1xxxxxxxx+

yyyyyyyyyyyy+

zzzzz
```

See [format-string-test.json](bolt12/format-string-test.json).

# Signature Calculation

All signatures are created as per
[BIP-340](https://github.com/bitcoin/bips/blob/master/bip-0340.mediawiki)
and tagged as recommended there.  Thus we define H(`tag`,`msg`) as
SHA256(SHA256(`tag`) || SHA256(`tag`) || `msg`), and SIG(`tag`,`msg`,`key`)
as the signature of H(`tag`,`msg`) using `key`.

Each form is signed using one or more *signature TLV elements*: TLV
types 240 through 1000 (inclusive).  For these,
the tag is "lightning" || `messagename` || `fieldname`, and `msg` is the
Merkle-root; "lightning" is the literal 9-byte ASCII string,
`messagename` is the name of the TLV stream being signed (i.e. "invoice_request" or "invoice") and the `fieldname` is the TLV field containing the
signature (e.g. "signature").

The formulation of the Merkle tree is similar to that proposed in
[BIP-341](https://github.com/bitcoin/bips/blob/master/bip-0341.mediawiki),
with each TLV leaf paired with a nonce leaf to avoid
revealing adjacent nodes in proofs.

The Merkle tree's leaves are, in TLV-ascending order for each tlv:
1. The H("LnLeaf",tlv).
2. The H("LnNonce"||first-tlv,tlv-type) where first-tlv is the numerically-first TLV entry in the stream, and tlv-type is the "type" field (1-9 bytes) of the current tlv.

The Merkle tree inner nodes are H("LnBranch", lesser-SHA256||greater-SHA256);
this ordering means proofs are more compact since left/right is
inherently determined.

If there is not exactly a power of 2 leaves, then the tree depth will
be uneven, with the deepest tree on the lowest-order leaves.

e.g. consider the encoding of an `invoice` `signature` with TLVs TLV1, TLV2, and TLV3 (of types 1, 2 and 3 respectively):

```
L1=H("LnLeaf",TLV1)
L1nonce=H("LnNonce"||TLV1,1)
L2=H("LnLeaf",TLV2)
L2nonce=H("LnNonce"||TLV1,2)
L3=H("LnLeaf",TLV3)
L3nonce=H("LnNonce"||TLV1,3)

Assume L1 < L1nonce, L2 > L2nonce and L3 > L3nonce.

   L1    L1nonce                      L2   L2nonce                L3   L3nonce
     \   /                             \   /                       \   /
      v v                               v v                         v v
L1A=H("LnBranch",L1||L1nonce) L2A=H("LnBranch",L2nonce||L2)  L3A=H("LnBranch",L3nonce||L3)
                 
Assume L1A < L2A:

       L1A   L2A                                 L3A=H("LnBranch",L3nonce||L3)
         \   /                                    |
          v v                                     v
  L1A2A=H("LnBranch",L1A||L2A)                   L3A=H("LnBranch",L3nonce||L3)
  
Assume L1A2A > L3A:

  L1A2A=H("LnBranch",L1A||L2A)          L3A
                          \            /
                           v          v
                Root=H("LnBranch",L3A||L1A2A)

Signature = SIG("lightninginvoicesignature", Root, nodekey)
```

# Offers

Offers are a precursor to an invoice: readers will either request an invoice
(or multiple) or send an invoice based on the offer.  An offer can be much longer-lived than a
particular invoice, so it has some different characteristics; in particular the amount can be in a non-lightning currency.  It's
also designed for compactness to fit inside a QR code easily.

Note that the non-signature TLV elements get mirrored into
invoice_request and invoice messages, so they each have specific and
distinct TLV ranges.

The human-readable prefix for offers is `lno`.

## TLV Fields for Offers

1. `tlv_stream`: `offer`
2. types:
    1. type: 2 (`offer_chains`)
    2. data:
        * [`...*chain_hash`:`chains`]
    1. type: 4 (`offer_metadata`)
    2. data:
        * [`...*byte`:`data`]
    1. type: 6 (`offer_currency`)
    2. data:
        * [`...*utf8`:`iso4217`]
    1. type: 8 (`offer_amount`)
    2. data:
        * [`tu64`:`amount`]
    1. type: 10 (`offer_description`)
    2. data:
        * [`...*utf8`:`description`]
    1. type: 12 (`offer_features`)
    2. data:
        * [`...*byte`:`features`]
    1. type: 14 (`offer_absolute_expiry`)
    2. data:
        * [`tu64`:`seconds_from_epoch`]
    1. type: 16 (`offer_paths`)
    2. data:
        * [`...*blinded_path`:`paths`]
    1. type: 18 (`offer_issuer`)
    2. data:
        * [`...*utf8`:`issuer`]
    1. type: 20 (`offer_quantity_min`)
    2. data:
        * [`tu64`:`min`]
    1. type: 22 (`offer_quantity_max`)
    2. data:
        * [`tu64`:`max`]
    1. type: 24 (`offer_node_id`)
    2. data:
        * [`point`:`node_id`]
    1. type: 26 (`offer_send_invoice`)
    1. type: 28 (`is_trusted`)

1. subtype: `blinded_path`
2. data:
   * [`point`:`first_node_id`]
   * [`point`:`blinding`]
   * [`byte`:`num_hops`]
   * [`num_hops*onionmsg_path`:`path`]

## Requirements For Offers

A writer of an offer:
  - MUST NOT set any tlv fields greater or equal to 80, or tlv field 0.
  - MUST set `offer_node_id` to the node's public key to request the invoice from.
  - MUST set `offer_description` to a complete description of the purpose
    of the payment.
  - if the chain for the invoice is not solely bitcoin:
    - MUST specify `offer_chains` the offer is valid for.
  - otherwise:
    - MAY omit `offer_chains`, implying that bitcoin is only chain.
  - if a specific minimum `offer_amount` is required for successful payment:
    - MUST set `offer_amount` to the amount expected (per item).
    - if the currency for `offer_amount` is that of all entries in `chains`:
      - MUST specify `amount` in multiples of the minimum lightning-payable unit
        (e.g. milli-satoshis for bitcoin).
    - otherwise:
      - MUST specify `offer_currency` `iso4217` as an ISO 4712 three-letter code.
      - MUST specify `offer_amount` in the currency unit adjusted by the ISO 4712
        exponent (e.g. USD cents).
  - MAY set `offer_metadata` for its own use.
  - if it supports bolt12 features:
    - MUST set `offer_features`.`features` to the bitmap of bolt12 features.
  - if the offer expires:
    - MUST set `offer_absolute_expiry` `seconds_from_epoch` to the number of seconds
      after midnight 1 January 1970, UTC that invoice_request should not be
      attempted.
  - if it is connected only by private channels:
    - MUST include `offer_paths` containing one or more paths to the node from
      publicly reachable nodes.
  - otherwise:
    - MAY include `offer_paths`.
  - if it includes `offer_paths`:
    - SHOULD ignore any invoice_request which does not use the path.
  - if it sets `offer_issuer`:
    - SHOULD set it to identify the issuer of the invoice clearly.
    - if it includes a domain name:
      - SHOULD begin it with either user@domain or domain
      - MAY follow with a space and more text
  - if it can supply more than one item for a single invoice
    - if the minimum quantity is more than 1:
      - MUST set that minimum in `offer_quantity_min`
    - if the maximum quantity is known:
      - MUST set that maximum in `offer_quantity_max`
    - if neither:
      - MUST set `offer_quantity_min` to 1 to indicate `quantity` is supported.
    - if both:
      - MUST set `offer_quantity_min` less than or equal to `offer_quantity_max`.
    - MUST NOT set `offer_quantity_min` or `offer_quantity_max` less than 1.

A reader of an offer:
  - if the offer contains any unknown TLV fields greater or equal to 80:
    - MUST NOT respond to the offer.
  - if `offer_features` contains unknown _odd_ bits that are non-zero:
    - MUST ignore the bit.
  - if `offer_features` contains unknown _even_ bits that are non-zero:
    - MUST NOT respond to the offer.
    - SHOULD indicate the unknown bit to the user.
  - if `offer_description` is not set:
    - MUST NOT respond to the offer.
  - if `offer_node_id` is not set:
    - MUST NOT respond to the offer.
  - if it uses `offer_amount` to provide the user with a cost estimate:
    - MUST warn user if amount of actual invoice differs significantly
        from that expectation.
  - SHOULD not respond to an offer if the current time is after
    `offer_absolute_expiry`.
  - if `is_trusted` is set:
    - MUST not respond to the offer unless the reader trusts the offer issuer and does not need proof of payment.
  - FIXME: more!

## Rationale

The entire offer is reflected in the invoice_request, both for
completeness (so all information will be returned in the invoice), and
so that the offer node can be stateless.  This makes `offer_metadata`
particularly useful, since it can contain an authentication cookie to
validate the other fields.

A signature is unnecessary, and makes for a longer string (potentially
limiting QR code use on low-end cameras); if the offer has an error, no
invoice will be given (or, for `send_invoice` offers, accepted) since
the `offer_id` already covers all the non-signature fields.

# Invoice Requests

Invoice Requests are a request for an invoice; the human-readable prefix for
invoices is `lnr`.  It mirrors all the fields from the offer, except
`offer_send_invoice` which cannot cause invoice_requests.

Note: the `invoice_request_metadata` is numbered 0 (not in the
80-159 range for other invoice_request fields) as this is the first
TLV element, which ensures payer-provided entropy is used in hashing
for [Signature Calculation](#signature-calculation).

## TLV Fields for `invoice_request`

1. `tlv_stream`: `invoice_request`
2. types:
    1. type: 0 (`invoice_request_metadata`)
    2. data:
        * [`...*byte`:`blob`]
    1. type: 2 (`offer_chains`)
    2. data:
        * [`...*chain_hash`:`chains`]
    1. type: 4 (`offer_metadata`)
    2. data:
        * [`...*byte`:`data`]
    1. type: 6 (`offer_currency`)
    2. data:
        * [`...*utf8`:`iso4217`]
    1. type: 8 (`offer_amount`)
    2. data:
        * [`tu64`:`amount`]
    1. type: 10 (`offer_description`)
    2. data:
        * [`...*utf8`:`description`]
    1. type: 12 (`offer_features`)
    2. data:
        * [`...*byte`:`features`]
    1. type: 14 (`offer_absolute_expiry`)
    2. data:
        * [`tu64`:`seconds_from_epoch`]
    1. type: 16 (`offer_paths`)
    2. data:
        * [`...*blinded_path`:`paths`]
    1. type: 18 (`offer_issuer`)
    2. data:
        * [`...*utf8`:`issuer`]
    1. type: 20 (`offer_quantity_min`)
    2. data:
        * [`tu64`:`min`]
    1. type: 22 (`offer_quantity_max`)
    2. data:
        * [`tu64`:`max`]
    1. type: 24 (`offer_node_id`)
    2. data:
        * [`point`:`node_id`]
    1. type: 28 (`is_trusted`)
    1. type: 80 (`invoice_request_chain`)
    2. data:
        * [`chain_hash`:`chain`]
    1. type: 82 (`invoice_request_amount`)
    2. data:
        * [`tu64`:`msat`]
    1. type: 84 (`invoice_request_features`)
    2. data:
        * [`...*byte`:`features`]
    1. type: 86 (`invoice_request_quantity`)
    2. data:
        * [`tu64`:`quantity`]
    1. type: 88 (`invoice_request_payer_key`)
    2. data:
        * [`point`:`key`]
    1. type: 89 (`invoice_request_payer_note`)
    2. data:
        * [`...*utf8`:`note`]
    1. type: 240 (`signature`)
    2. data:
        * [`bip340sig`:`sig`]

## Requirements for Invoice Requests

The writer:
  - if responding to an offer:
    - MUST copy all fields from the offer (including unknown fields).
  - otherwise:
    - MUST set `offer_node_id` to the (possibly blinded) public key of the node to request the invoice from.
    - MUST set `description` to a complete description of the purpose of the payment.
    - MUST set `is_trusted`.
    - MUST NOT expect to receive anything in exchange of paying the invoice.
  - MUST NOT set any tlv fields greater or equal to 160.
  - MUST set `invoice_request_metadata` to an unpredictable series of bytes.
  - MUST set `invoice_request_payer_key` to a transient public key.
  - MUST remember the secret key corresponding to `invoice_request_payer_key`.
  - if `offer_chains` is set:
    - MUST set `invoice_request_chain` to one of `offer_chains` unless that chain is bitcoin, in which case it MAY omit `invoice_request_chain`.
  - otherwise:
    - if it sets `invoice_request_chain` it MUST set it to bitcoin.
  - MUST set `signature`.`sig` as detailed in [Signature Calculation](#signature-calculation) using the `invoice_request_payer_key`.
  - if `offer_quantity_min` or `offer_quantity_max` are present:
    - MUST set `invoice_request_quantity`
    - MUST set it within that (inclusive) range.
  - otherwise:
    - MUST NOT set `invoice_request_quantity`
  - if `offer_amount`:
    - MUST specify `invoice_request_amount`.`msat` in multiples of the minimum lightning-payable unit
      (e.g. milli-satoshis for bitcoin) for `chain` (or for bitcoin, if there is no `chain`).
  - otherwise:
    - MAY omit `invoice_request_amount`.
    - if it sets `invoice_request_amount`:
      - MUST specify `invoice_request_amount`.`msat` as greater or equal to amount expected by `offer_amount` (and, if present, `offer_currency`).
  - if it supports bolt12 features:
    - MUST set `invoice_request_features`.`features` to the bitmap of features.

The reader:
  - MUST fail the request if `invoice_request_payer_key` is not present.
  - MUST fail the request if any fields have type greater or equal to 160.
  - if `invoice_request_features` contains unknown _odd_ bits that are non-zero:
    - MUST ignore the bit.
  - if `invoice_request_features` contains unknown _even_ bits that are non-zero:
    - MUST fail the request.
  - if `invoice_request_chain` is not present:
    - MUST fail the request if bitcoin is not a supported chain.
  - otherwise:
    - MUST fail the request if `invoice_request_chain`.`chain` is not a supported chain.
  - MUST fail the request if `invoice_request_features` contains unknown even bits.
  - MUST fail the request if `offer_send_invoice` is present.
  - if `is_trusted` is set:
    - MAY respond with an invoice with the understanding that the requester does not expect anything in exchange of paying the invoice.
  - otherwise:
    - MUST fail the request if the offer fields do not exactly match a valid, unexpired offer.
  - MUST fail the request if `invoice_request_signature` is not correct as detailed in [Signature Calculation](#signature-calculation) using the `invoice_request_payer_key`.
  - if `offer_quantity_min` or `offer_quantity_max` is present:
    - MUST fail the request if there is no `invoice_request_quantity` field.
    - MUST fail the request if `invoice_request_quantity` is not within that (inclusive) range.
  - otherwise:
    - MUST fail the request if there is an `invoice_request_quantity` field.
  - if `offer_amount` is present:
    - MUST calculate the *base invoice amount* using the `offer_amount`:
      - if `offer_currency` is not the `invoice_request_chain` currency, convert to the
        `invoice_request_chain` currency.
      - if `invoice_request_quantity` is present, multiply by `invoice_request_quantity`.`quantity`.
    - if `invoice_request_amount` is present:
      - MUST fail the request if `invoice_request_amount`.`msat` is less than the *base invoice amount*.
      - MAY fail the request if `invoice_request_amount`.`msat` exceeds the *base invoice amount*.
  - otherwise (no `offer_amount`):
    - MUST fail the request if it does not contain `invoice_request_amount`.

## Rationale

`invoice_request_metadata` might typically contain information about the derivation of the
`invoice_request_payer_key`.  This should not leak any information (such as using a simple
BIP-32 derivation path); a valid system might be for a node to maintain a base
payer key and encode a 128-bit tweak here.  The payer_key would be derived by
tweaking the base key with SHA256(payer_base_pubkey || tweak).  It's also
the first entry (if present), ensuring an unpredictable nonce for hashing.

`invoice_request_payer_note` allows you to compliment, taunt, or otherwise engrave
graffiti into the invoice for all to see.

Users can give a tip (or obscure the amount sent) by specifying an
`invoice_request_amount` in their invoice request, even though the offer specifies an
`offer_amount`.  The recipient will only accept this if
the invoice request amount exceeds the amount it's expecting (i.e. its
`offer_amount` after any currency conversion, multiplied by `invoice_request_quantity`, if
any).

Users should be able to send tips or pay friends without needing a preexisting offer.
In that case the payer can't expect a proof that they are entitled to receive something and they signal this by setting `is_trusted`.

# Invoices

Invoices are a payment request, and when the payment is made, 
it can be combined with the invoice to form a cryptographic receipt.

The human-readable prefix for invoices is `lni`.  The recipient can send it in
response to an `invoice_request` or an `offer` with `offer_send_invoice`
using the `onion_message` `invoice` field.

1. `tlv_stream`: `invoice`
2. types:
    1. type: 0 (`invoice_request_metadata`)
    2. data:
        * [`...*byte`:`blob`]
    1. type: 2 (`offer_chains`)
    2. data:
        * [`...*chain_hash`:`chains`]
    1. type: 4 (`offer_metadata`)
    2. data:
        * [`...*byte`:`data`]
    1. type: 6 (`offer_currency`)
    2. data:
        * [`...*utf8`:`iso4217`]
    1. type: 8 (`offer_amount`)
    2. data:
        * [`tu64`:`amount`]
    1. type: 10 (`offer_description`)
    2. data:
        * [`...*utf8`:`description`]
    1. type: 12 (`offer_features`)
    2. data:
        * [`...*byte`:`features`]
    1. type: 14 (`offer_absolute_expiry`)
    2. data:
        * [`tu64`:`seconds_from_epoch`]
    1. type: 16 (`offer_paths`)
    2. data:
        * [`...*blinded_path`:`paths`]
    1. type: 18 (`offer_issuer`)
    2. data:
        * [`...*utf8`:`issuer`]
    1. type: 20 (`offer_quantity_min`)
        * [`tu64`:`min`]
    1. type: 22 (`offer_quantity_max`)
    2. data:
        * [`tu64`:`max`]
    1. type: 24 (`offer_node_id`)
    2. data:
        * [`point`:`node_id`]
    1. type: 26 (`offer_send_invoice`)
    1. type: 28 (`is_trusted`)
    1. type: 80 (`invoice_request_chain`)
    2. data:
        * [`chain_hash`:`chain`]
    1. type: 82 (`invoice_request_amount`)
    2. data:
        * [`tu64`:`msat`]
    1. type: 84 (`invoice_request_features`)
    2. data:
        * [`...*byte`:`features`]
    1. type: 86 (`invoice_request_quantity`)
    2. data:
        * [`tu64`:`quantity`]
    1. type: 88 (`invoice_request_payer_key`)
    2. data:
        * [`point`:`key`]
    1. type: 89 (`invoice_request_payer_note`)
    2. data:
        * [`...*utf8`:`note`]
    1. type: 160 (`invoice_paths`)
    2. data:
        * [`...*blinded_path`:`paths`]
    1. type: 162 (`invoice_blindedpay`)
    2. data:
        * [`...*blinded_payinfo`:`payinfo`]
    1. type: 164 (`invoice_created_at`)
    2. data:
        * [`tu64`:`timestamp`]
    1. type: 166 (`invoice_relative_expiry`)
    2. data:
        * [`tu32`:`seconds_from_creation`]
    1. type: 168 (`invoice_payment_hash`)
    2. data:
        * [`sha256`:`payment_hash`]
    1. type: 170 (`invoice_amount`)
    2. data:
        * [`tu64`:`msat`]
    1. type: 172 (`invoice_fallbacks`)
    2. data:
        * [`...*fallback_address`:`fallbacks`]
    1. type: 174 (`invoice_features`)
    2. data:
        * [`...*byte`:`features`]
    1. type: 240 (`signature`)
    2. data:
        * [`bip340sig`:`sig`]

1. subtype: `blinded_payinfo`
2. data:
   * [`u32`:`fee_base_msat`]
   * [`u32`:`fee_proportional_millionths`]
   * [`u16`:`cltv_expiry_delta`]
   * [`u64`:`htlc_minimum_msat`]
   * [`u64`:`htlc_maximum_msat`]
   * [`u16`:`flen`]
   * [`flen*byte`:`features`]

1. subtype: `fallback_address`
2. data:
   * [`byte`:`version`]
   * [`u16`:`len`]
   * [`len*byte`:`address`]

## Requirements

A writer of an invoice:
  - if creating an `invoice` for an offer with `offer_send_invoice`:
     - MUST copy all fields from the offer (including unknown fields).
  - otherwise (responding to an `invoice_request`):
     - MUST copy all non-signature fields from the invoice_request (including unknown fields).
  - MUST set `invoice_created_at` to the number of seconds since Midnight 1
    January 1970, UTC when the offer was created.
  - MUST set `invoice_payment_hash` to the SHA256 hash of the
    `payment_preimage` that will be given in return for payment.
  - MUST specify exactly one signature TLV element: `signature`.
    - MUST set `sig` to the signature using `offer_node_id` as described in [Signature Calculation](#signature-calculation).
  - if it supports bolt12 features:
    - MUST set `invoice_features`.`features` to the bitmap of features.
  - if the expiry for accepting payment is not 7200 seconds after `invoice_created_at`:
    - MUST set `invoice_relative_expiry`.`seconds_from_creation` to the number of
      seconds after `invoice_created_at` that payment of this invoice should not be attempted.
  - if it accepts onchain payments:
    - MAY specify `invoice_fallbacks`
    - MUST specify `invoice_fallbacks` in order of most-preferred to least-preferred
      if it has a preference.
    - for the bitcoin chain, it MUST set each `fallback_address` with
      `version` as a valid witness version and `address` as a valid witness
      program
  - MUST include `invoice_paths` containing one or more paths to the node.
    - MUST specify `invoice_paths` in order of most-preferred to least-preferred if it has a preference.
    - MUST include `invoice_blindedpay` with exactly one `blinded_payinfo` for each `blinded_path` in `paths`, in order.
    - MUST set `features` in each `blinded_payinfo` to match `encrypted_data_tlv`.`allowed_features` (or empty, if no `allowed_features`).
    - SHOULD ignore any payment which does not use one of the paths.
  - if responding to an `invoice_request`:
    - if `invoice_request_payer_key` and offer are identical to a previous `invoice_request`:
      - MAY simply reuse the previous invoice.
    - otherwise:
      - MUST NOT reuse a previous invoice.
    - if `invoice_request_amount` is present:
      - MUST set `invoice_amount` to `invoice_request_amount`
    - otherwise: (no `invoice_request_amount`, so must have `offer_amount`):
      - MUST set `invoice_amount`.`msat` to the *base invoice amount*.
  - otherwise (responding to a `offer_send_invoice` offer):
    - MUST fail the request if `offer_send_invoice` is not present.
    - MUST fail the request if the offer fields not do exactly match a valid, unexpired offer.
    - if `offer_quantity_min` or `offer_quantity_max` are present:
      - MUST set `invoice_request_quantity`
      - MUST set it within that (inclusive) range.
    - otherwise:
      - MUST NOT set `invoice_request_quantity`
    - MUST set `invoice_request_payer_key` to `offer_node_id`.
    - MUST set `invoice_request_metadata`
	  - SHOULD set it to at least 16 random bytes.

A reader of an invoice:
  - MUST reject the invoice if `signature` is not a valid signature using `offer_node_id` as described in [Signature Calculation](#signature-calculation).
  - MUST reject the invoice if `invoice_amount` is not present.
  - MUST reject the invoice if `offer_description` is not present.
  - MUST reject the invoice if `invoice_created_at` is not present.
  - MUST reject the invoice if `invoice_payment_hash` is not present.
  - if `invoice_features` contains unknown _odd_ bits that are non-zero:
    - MUST ignore the bit.
  - if `invoice_features` contains unknown _even_ bits that are non-zero:
    - MUST reject the invoice.
  - if `invoice_relative_expiry` is present:
    - MUST reject the invoice if the current time since 1970-01-01 UTC is greater than `invoice_created_at` plus `seconds_from_creation`.
  - otherwise:
    - MUST reject the invoice if the current time since 1970-01-01 UTC is greater than `invoice_created_at` plus 7200.
  - MUST reject the invoice if `invoice_paths` is not present or is empty.
  - MUST reject the invoice if `invoice_blindedpay` is not present.
  - MUST reject the invoice if `invoice_blindedpay` does not contain exactly one `blinded_payinfo` per `invoice_paths`.`blinded_path`.
  - MUST reject the invoice if `features` in any `blinded_payinfo` has any unknown even bits set.
  - SHOULD confirm authorization if `invoice_amount`.`msat` is not within the amount range authorized.
  - if the invoice is a reply to an `invoice_request`:
     - MUST reject the invoice if all fields less than type 160 do not exactly match the `invoice_request`
  - otherwise, if responding to an `offer_send_invoice` offer:
    - MUST fail the request if `offer_send_invoice` is not present.
    - MUST fail the request if the offer fields not do exactly match a valid, unexpired offer.
    - if `invoice_request_chain` is not present:
       - MUST reject the invoice if bitcoin is not a supported chain for the offer.
    - otherwise:
      - MUST reject the invoice if `invoice_request_chain` is not a supported chain for the offer.
    - if `offer_quantity_min` or `offer_quantity_max` is present:
      - MUST reject the invoice if there is no `invoice_request_quantity` field.
      - MUST reject the invoice if `invoice_request_quantity` is not within that (inclusive) range.
    - otherwise:
      - MUST reject the invoice if there is an `invoice_request_quantity` field.
  - otherwise: (not an `invoice_request` reply, nor for `offer_send_invoice`):
    - if `invoice_request_chain` is not present:
       - MUST reject the invoice if bitcoin is not a supported chain.
    - otherwise:
       - MUST reject the invoice if `invoice_request_chain` is not a supported chain.
  - for the bitcoin chain, if the invoice specifies `invoice_fallbacks`:
    - MUST ignore any `fallback_address` for which `version` is greater than 16.
    - MUST ignore any `fallback_address` for which `address` is less than 2 or greater than 40 bytes.
    - MUST ignore any `fallback_address` for which `address` does not meet known requirements for the given `version`

## Rationale

Because the messaging layer is unreliable, it's quite possible to
receive multiple requests for the same offer.  As it's the caller's
responsibility not to reuse `invoice_request_payer_key`
the writer doesn't have to check all the fields are duplicates before
simply returning a previous invoice.  Note that such caching is optional,
and should be carefully limited when e.g. currency conversion is involved,
or if the invoice has expired.

The invoice duplicates fields rather than committing to the previous offer or
invoice_request.  This flattened format simplifies storage at some space cost, as
the payer need only remember the invoice for any refunds or proof.

The reader of the invoice cannot trust the invoice correctly reflects the
offer and invoice_request fields, hence the requirements to check that they
are correct.

Note that the recipient of the invoice can determine the expected
amount from either the offer it received, or the invoice_request it
sent, so often already has authorization for the expected amount.

The default `invoice_relative_expiry` of 7200 seconds, which is generally a
sufficient time for payment, even if new channels need to be opened.

Blinded paths provide an equivalent to `payment_secret` and `payment_metadata` used in BOLT 11.
Even if `offer_node_id` is public, we force the use of blinding paths to keep these features.
If the recipient does not care about the added privacy offered by blinded paths, they can create a path of length 1 with only themselves.

Rather than provide detailed per-hop-payinfo for each hop in a blinded path, we aggregate the fees and CLTV deltas.
This avoids trivially revealing any distinguishing non-uniformity which may distinguish the path.

# Invoice Errors

Informative errors can be returned in an onion message `invoice_error`
field (via the onion `reply_path`) for either `invoice_request` or
`invoice`.

## TLV Fields for `invoice_error`

1. `tlv_stream`: `invoice_error`
2. types:
    1. type: 1 (`erroneous_field`)
    2. data:
        * [`tu64`:`tlv_fieldnum`]
    1. type: 3 (`suggested_value`)
    2. data:
        * [`...*byte`:`value`]
    1. type: 5 (`error`)
    2. data:
        * [`...*utf8`:`msg`]

## Requirements

A writer of an invoice_error:
  - MUST set `error` to an explanatory string.
  - MAY set `erroneous_field` to a specific field number in the
    `invoice` or `invoice_request` which had a problem.
  - if it sets `erroneous_field`:
    - MAY set `suggested_value`.
    - if it sets `suggested_value`:
      - MUST set `suggested_value` to a valid field for that `tlv_fieldnum`.
  - otherwise:
    - MUST NOT set `suggested_value`.

A reader of an invoice_error:
   FIXME!

## Rationale

Usually an error message is sufficient for diagnostics, however there
is at least one case where it should be programmatically parsable.  An
offer which sets `offer_send_invoice` can also specify a currency,
which opens the possibility for a disagreement on exchange rate.  In
this case, the `suggested_value` reflects its expected value, and the
sender can send a new invoice.

# FIXME: Possible future extensions:

1. The offer can require delivery info in the `invoice_request`.
2. An offer can be updated: the response to an `invoice_request` is another offer,
   perhaps with a signature from the original `offer_node_id`
3. Any empty TLV fields can mean the value is supposed to be known by
   other means (i.e. transport-specific), but is still hashed for sig.
4. We could upgrade to allow multiple offers in one invoice_request and
   invoice, to make a shopping list.
7. All-zero offer_id == gratuitous payment.
8. Streaming invoices?
9. Re-add recurrence.
10. Re-add `offer_refund_for` for `offer_send_invoice` to support proofs.
11. Re-add `invoice_replace` for requesting replacement of a (stuck-payment) 
    invoice with a new one.

[1] https://www.youtube.com/watch?v=4SYc_flMnMQ
