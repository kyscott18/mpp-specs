---
title: Monad charge Intent for HTTP Payment Authentication
abbrev: Monad Charge
docname: draft-monad-charge-00
version: 00
category: info
ipr: noModificationTrust200902
submissiontype: IETF
consensus: true

author:
  - name: Kyle Scott
    ins: K. Scott
    email: kscott@monad.foundation
    org: Monad Foundation

normative:
  RFC2119:
  RFC3339:
  RFC4648:
  RFC8174:
  RFC8259:
  RFC8785:
  I-D.httpauth-payment:
    title: "The 'Payment' HTTP Authentication Scheme"
    target: https://datatracker.ietf.org/doc/draft-ietf-httpauth-payment/
    author:
      - name: Jake Moxey
    date: 2026-01
  I-D.payment-intent-charge:
    title: "'charge' Intent for HTTP Payment Authentication"
    target: https://datatracker.ietf.org/doc/draft-payment-intent-charge/
    author:
      - name: Jake Moxey
      - name: Brendan Ryan
      - name: Tom Meagher
    date: 2026

informative:
  EIP-20:
    title: "Token Standard"
    target: https://eips.ethereum.org/EIPS/eip-20
    author:
      - name: Fabian Vogelsteller
      - name: Vitalik Buterin
    date: 2015-11
  EIP-3009:
    title: "Transfer With Authorization"
    target: https://eips.ethereum.org/EIPS/eip-3009
    author:
      - name: Peter Jihoon Kim
      - name: Kevin Britz
      - name: David Knott
    date: 2020-09
  MONAD-DOCS:
    title: "Monad Documentation"
    target: https://docs.monad.xyz/
    author:
      - org: Monad Foundation
    date: 2025
---

--- abstract

This document defines the "charge" intent for the "monad" payment method
in the Payment HTTP Authentication Scheme {{I-D.httpauth-payment}}. It
specifies how clients and servers exchange one-time ERC-20 token transfers
on the Monad blockchain.

--- middle

# Introduction

The `charge` intent represents a one-time payment of a specified amount,
as defined in {{I-D.payment-intent-charge}}. The server may settle the
payment any time before the challenge `expires` auth-param timestamp.

Monad is a high-performance EVM-compatible Layer-1 blockchain with
sub-second finality (~800ms) {{MONAD-DOCS}}. This
specification defines the request schema, credential formats, and
settlement procedures for charge transactions on Monad.

Two credential types are supported: `type="authorization"` (pull mode),
where the client signs an ERC-3009 {{EIP-3009}} authorization off-chain
and the server broadcasts it, and `type="hash"` (push mode), where the
client broadcasts the transaction itself and presents the on-chain
transaction hash for server verification.

## Charge Flow

The following diagram illustrates the Monad charge flow in push mode
(client broadcasts):

~~~
   Client                        Server                     Monad Network
      |                             |                             |
      |  (1) GET /api/resource      |                             |
      |-------------------------->  |                             |
      |                             |                             |
      |  (2) 402 Payment Required   |                             |
      |      intent="charge"        |                             |
      |<--------------------------  |                             |
      |                             |                             |
      |  (3) Broadcast ERC-20       |                             |
      |      transfer tx            |                             |
      |------------------------------------------------------>   |
      |                             |                             |
      |  (4) Transaction confirmed  |                             |
      |<------------------------------------------------------   |
      |                             |                             |
      |  (5) Authorization: Payment |                             |
      |      (with txHash)          |                             |
      |-------------------------->  |                             |
      |                             |  (6) Verify tx receipt      |
      |                             |-------------------------->  |
      |                             |  (7) Receipt confirmed      |
      |                             |<--------------------------  |
      |  (8) 200 OK + Receipt       |                             |
      |<--------------------------  |                             |
      |                             |                             |
~~~

The following diagram illustrates the pull mode (server broadcasts via
ERC-3009):

~~~
   Client                        Server                     Monad Network
      |                             |                             |
      |  (1) GET /api/resource      |                             |
      |-------------------------->  |                             |
      |                             |                             |
      |  (2) 402 Payment Required   |                             |
      |      intent="charge"        |                             |
      |<--------------------------  |                             |
      |                             |                             |
      |  (3) Sign ERC-3009          |                             |
      |      TransferWithAuth       |                             |
      |                             |                             |
      |  (4) Authorization: Payment |                             |
      |      (with authorization)   |                             |
      |-------------------------->  |                             |
      |                             |  (5) transferWithAuth tx    |
      |                             |-------------------------->  |
      |                             |  (6) Transfer complete      |
      |                             |<--------------------------  |
      |  (7) 200 OK + Receipt       |                             |
      |<--------------------------  |                             |
      |                             |                             |
~~~

# Requirements Language

{::boilerplate bcp14-tagged}

# Terminology

ERC-20
: The standard interface for fungible tokens on EVM-compatible
  blockchains {{EIP-20}}. ERC-20 defines `transfer`, `transferFrom`,
  `approve`, and related operations.

ERC-3009
: An extension that adds gasless transfer authorization to ERC-20
  tokens {{EIP-3009}}. ERC-3009 defines `transferWithAuthorization` and
  `receiveWithAuthorization`, enabling off-chain signed approvals that
  a third party can submit on-chain.

# Request Schema

The `request` parameter in the `WWW-Authenticate` challenge contains a
base64url-encoded JSON object. The JSON MUST be serialized using JSON
Canonicalization Scheme (JCS) {{RFC8785}} before base64url encoding,
per {{I-D.httpauth-payment}}.

This specification implements the shared request fields defined in
{{I-D.payment-intent-charge}}.

## Shared Fields

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `amount` | string | REQUIRED | Amount in base units (stringified number) |
| `currency` | string | REQUIRED | ERC-20 token address (e.g., `"0x7547..."`) |
| `recipient` | string | REQUIRED | Recipient address |
| `description` | string | OPTIONAL | Human-readable payment description |
| `externalId` | string | OPTIONAL | Merchant's reference (order ID, invoice number, etc.) |

Challenge expiry is conveyed by the `expires` auth-param in
`WWW-Authenticate` per {{I-D.httpauth-payment}}.

## Method Details

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `methodDetails.chainId` | number | OPTIONAL | Monad chain ID (default: 143) |

**Example:**

~~~json
{
  "amount": "1000000",
  "currency": "0x754704Bc059F8C67012fEd69BC8A327a5aafb603",
  "recipient": "0x742d35Cc6634C0532925a3b844Bc9e7595f8fE00",
  "methodDetails": {
    "chainId": 143
  }
}
~~~

The client fulfills this by either:

1. **Push mode**: Broadcasting an ERC-20 `transfer(recipient, amount)`
   transaction to Monad and returning the transaction hash.
2. **Pull mode**: Signing an ERC-3009 `TransferWithAuthorization` message
   and returning the authorization parameters and signature.

# Credential Schema

The credential in the `Authorization` header contains a base64url-encoded
JSON object per {{I-D.httpauth-payment}}.

## Credential Structure

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `challenge` | object | REQUIRED | Echo of the challenge from the server |
| `payload` | object | REQUIRED | Monad-specific payload object |
| `source` | string | OPTIONAL | Payer identifier as a DID (e.g., `did:pkh:eip155:143:0x...`) |

The `source` field, if present, SHOULD use the `did:pkh` method with the
chain ID applicable to the challenge and the payer's Ethereum address.

## Hash Payload (type="hash")

When `type` is `"hash"`, the client has already broadcast the ERC-20
`transfer` transaction to the Monad network. The `hash` field contains
the transaction hash for the server to verify onchain.

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `hash` | string | REQUIRED | Transaction hash with `0x` prefix |
| `type` | string | REQUIRED | `"hash"` |

**Example:**

~~~json
{
  "challenge": {
    "id": "kM9xPqWvT2nJrHsY4aDfEb",
    "realm": "api.example.com",
    "method": "monad",
    "intent": "charge",
    "request": "eyJ...",
    "expires": "2025-02-05T12:05:00Z"
  },
  "payload": {
    "hash": "0x1a2b3c4d5e6f7890abcdef1234567890abcdef1234567890abcdef1234567890",
    "type": "hash"
  },
  "source": "did:pkh:eip155:143:0x1234567890abcdef1234567890abcdef12345678"
}
~~~

## Authorization Payload (type="authorization")

When `type` is `"authorization"`, the client has signed an ERC-3009
`TransferWithAuthorization` message off-chain. The server calls
`transferWithAuthorization` on the token contract to execute the transfer.

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `type` | string | REQUIRED | `"authorization"` |
| `from` | string | REQUIRED | Payer address |
| `to` | string | REQUIRED | Recipient address |
| `value` | string | REQUIRED | Transfer amount in base units (stringified) |
| `validAfter` | string | REQUIRED | Unix timestamp (stringified); authorization not valid before this time |
| `validBefore` | string | REQUIRED | Unix timestamp (stringified); authorization not valid after this time |
| `nonce` | string | REQUIRED | Unique `bytes32` nonce for replay protection |
| `signature` | string | REQUIRED | EIP-712 signature of the `TransferWithAuthorization` typed data |

**Example:**

~~~json
{
  "challenge": {
    "id": "kM9xPqWvT2nJrHsY4aDfEb",
    "realm": "api.example.com",
    "method": "monad",
    "intent": "charge",
    "request": "eyJ...",
    "expires": "2025-02-05T12:05:00Z"
  },
  "payload": {
    "type": "authorization",
    "from": "0x1234567890abcdef1234567890abcdef12345678",
    "to": "0x742d35Cc6634C0532925a3b844Bc9e7595f8fE00",
    "value": "1000000",
    "validAfter": "0",
    "validBefore": "1738756800",
    "nonce": "0xa1b2c3d4e5f6...32 bytes...00",
    "signature": "0x...EIP-712 signature..."
  },
  "source": "did:pkh:eip155:143:0x1234567890abcdef1234567890abcdef12345678"
}
~~~

# Settlement Modes

## Push Mode (Client Broadcasts)

In push mode, the client broadcasts the ERC-20 `transfer` transaction
directly to the Monad network and provides the transaction hash to the
server:

~~~
   Client                           Server                        Monad Network
      |                                |                                |
      |  (1) Broadcast ERC-20          |                                |
      |      transfer(to, amount)      |                                |
      |------------------------------------------------------->        |
      |                                |                                |
      |  (2) Transaction confirmed     |                                |
      |      (~800ms finality)         |                                |
      |<-------------------------------------------------------        |
      |                                |                                |
      |  (3) Authorization:            |                                |
      |      Payment <credential>      |                                |
      |      (type="hash")             |                                |
      |------------------------------->|                                |
      |                                |                                |
      |                                |  (4) eth_getTransactionReceipt |
      |                                |------------------------------->|
      |                                |  (5) Receipt returned          |
      |                                |<-------------------------------|
      |                                |                                |
      |                                |  (6) Verify Transfer event     |
      |                                |                                |
      |  (7) 200 OK                    |                                |
      |      Payment-Receipt: <receipt>|                                |
      |<-------------------------------|                                |
      |                                |                                |
~~~

1. Client broadcasts ERC-20 `transfer(recipient, amount)` to Monad
2. Transaction included in block with fast finality (~800ms)
3. Client submits credential with `type="hash"` containing the tx hash
4. Server fetches the transaction receipt
5. Server verifies the `Transfer` event log matches the challenge
6. Server returns a receipt whose `reference` is the transaction hash

## Pull Mode (Server Broadcasts via ERC-3009)

In pull mode, the client signs an ERC-3009 `TransferWithAuthorization`
message. The server calls `transferWithAuthorization` and pays gas:

~~~
   Client                           Server                        Monad Network
      |                                |                                |
      |  (1) Sign EIP-712             |                                |
      |      TransferWithAuthorization |                                |
      |                                |                                |
      |  (2) Authorization:            |                                |
      |      Payment <credential>      |                                |
      |      (type="authorization")    |                                |
      |------------------------------->|                                |
      |                                |                                |
      |                                |  (3) Verify params match       |
      |                                |                                |
      |                                |  (4) eth_sendRawTxSync         |
      |                                |------------------------------->|
      |                                |                                |
      |                                |  (5) Transfer complete         |
      |                                |      (~800ms finality)         |
      |                                |<-------------------------------|
      |                                |                                |
      |  (6) 200 OK                    |                                |
      |      Payment-Receipt: <receipt>|                                |
      |<-------------------------------|                                |
      |                                |                                |
~~~

1. Client signs an EIP-712 `TransferWithAuthorization` typed data message
2. Client submits credential with `type="authorization"` containing the
   authorization parameters and signature
3. Server validates all parameters match the challenge
4. Server calls `transferWithAuthorization` on the token contract
5. Transaction confirmed with fast finality (~800ms)
6. Server returns a receipt whose `reference` is the transaction hash

### ERC-3009 Token Requirements

Pull mode requires the ERC-20 token to implement ERC-3009 {{EIP-3009}}.
Clients MUST NOT use pull mode with tokens that do not support
ERC-3009. Known supported tokens include USDC.

### EIP-712 Domain

The client signs the `TransferWithAuthorization` message using EIP-712
with the following domain:

| Field | Value |
|-------|-------|
| `name` | Token name (e.g., `"USD Coin"`) |
| `version` | Token version (e.g., `"1"`) |
| `chainId` | Monad chain ID |
| `verifyingContract` | Token contract address |

The typed data uses the `TransferWithAuthorization` primary type:

~~~
TransferWithAuthorization(address from,address to,uint256 value,uint256 validAfter,uint256 validBefore,bytes32 nonce)
~~~

## Transaction Verification {#transaction-verification}

Before accepting a credential, servers MUST verify:

### For hash credentials:

1. Fetch the transaction receipt via `eth_getTransactionReceipt`
2. Parse ERC-20 `Transfer` event logs from the receipt
3. Verify a `Transfer` event exists where:
   - The emitting contract matches the `currency` token address
   - The `from` field matches the `source` payer address
   - The `to` field matches the `recipient`
   - The `value` matches the `amount`
4. Verify the transaction hash has not been used in a previous credential

### For authorization credentials:

1. Verify `payload.to` matches the challenge `recipient`
2. Verify `payload.value` matches the challenge `amount`
3. Verify `validBefore` has not passed
4. Verify the authorization signature has not been used in a previous
   credential
5. Call `transferWithAuthorization` on the token contract with the provided
   parameters and signature components (`v`, `r`, `s`)

## Receipt Generation

Upon successful settlement, servers MUST return a `Payment-Receipt` header
per {{I-D.httpauth-payment}}. Servers MUST NOT include a
`Payment-Receipt` header on error responses; failures are communicated via
HTTP status codes and Problem Details.

The receipt payload for Monad charge:

| Field | Type | Description |
|-------|------|-------------|
| `method` | string | `"monad"` |
| `reference` | string | Transaction hash of the settlement transaction |
| `status` | string | `"success"` |
| `timestamp` | string | {{RFC3339}} settlement time |
| `externalId` | string | OPTIONAL. Echoed from the challenge request |

# Security Considerations

## Transaction Replay

EVM transactions include chain ID and nonce that prevent replay attacks:

- Chain ID binding prevents cross-chain replay
- Nonce consumption prevents same-chain replay
- ERC-3009 authorizations include their own `nonce` (bytes32) for
  replay protection, separate from the account nonce

While on-chain mechanisms prevent double-spending, servers MUST also
track consumed credentials at the application layer:

- **Hash credentials**: Servers MUST record each accepted transaction
  hash and reject subsequent credentials that reuse it. Without this,
  a client could submit the same confirmed transfer to multiple
  endpoints or repeatedly to the same server.
- **Authorization credentials**: Servers MUST record each accepted
  authorization signature (or a hash thereof) and reject duplicates
  before broadcasting. Without this, concurrent submissions of the
  same authorization could cause the server to broadcast duplicate
  `transferWithAuthorization` transactions, wasting gas on reverted
  calls.

## Amount Verification

Clients MUST parse and verify the `request` payload before signing:

1. Verify `amount` is reasonable for the service
2. Verify `currency` is the expected token address
3. Verify `recipient` is controlled by the expected party

## ERC-3009 Authorization Security

When using pull mode with ERC-3009 authorizations:

- The `validBefore` timestamp SHOULD be set to the challenge `expires`
  time to limit the authorization window
- The `validAfter` timestamp SHOULD be set to `0` (immediately valid)
- A random `bytes32` nonce MUST be generated for each authorization to
  prevent replay

## Authorization Front-Running

The EIP-712 typed data signed by the client is `TransferWithAuthorization`.
Any party that obtains the signature can call `transferWithAuthorization`
— which has no `msg.sender` restriction — to execute the transfer before
the server does.

The funds still arrive at the correct `to` address and cannot be
redirected, but the server's subsequent `transferWithAuthorization` call
will revert (the ERC-3009 nonce is already consumed), wasting gas. The
server may also fail to correlate the transfer with the client's request,
since it arrived via a different transaction.

Mitigations:

- Authorization credentials are transmitted over HTTPS directly to the
  server, not broadcast to a public mempool, limiting the interception
  window.
- Servers SHOULD submit `transferWithAuthorization` promptly after
  validation to minimize exposure.
- Servers MAY monitor for `Transfer` events matching a pending
  authorization to detect front-run settlements and still grant access.

## Server-Paid Gas (Pull Mode)

Servers accepting pull mode credentials pay gas fees for broadcasting
`transferWithAuthorization`. This creates financial risk:

**Denial of Service**: Malicious clients could submit authorizations
that fail on-chain (e.g., insufficient token balance), causing the
server to pay gas without receiving payment. Servers SHOULD implement
rate limiting and MAY require client authentication before accepting
authorization credentials.

**Gas Token Exhaustion**: Servers MUST monitor their native token
balance and reject new payment requests when balance is insufficient.

# IANA Considerations

## Payment Method Registration

This document registers the following payment method in the "HTTP Payment
Methods" registry established by {{I-D.httpauth-payment}}:

| Method Identifier | Description | Reference |
|-------------------|-------------|-----------|
| `monad` | Monad blockchain ERC-20 token transfer | This document |

Contact: Monad Foundation (<kscott@monad.foundation>)

## Payment Intent Registration

This document registers the following payment intent in the "HTTP Payment
Intents" registry established by {{I-D.httpauth-payment}}:

| Intent | Applicable Methods | Description | Reference |
|--------|-------------------|-------------|-----------|
| `charge` | `monad` | One-time ERC-20 transfer | This document |

--- back

# ABNF Collected

~~~ abnf
monad-charge-challenge = "Payment" 1*SP
  "id=" quoted-string ","
  "realm=" quoted-string ","
  "method=" DQUOTE "monad" DQUOTE ","
  "intent=" DQUOTE "charge" DQUOTE ","
  "request=" base64url-nopad

monad-charge-credential = "Payment" 1*SP base64url-nopad

; Base64url encoding without padding per RFC 4648 Section 5
base64url-nopad = 1*( ALPHA / DIGIT / "-" / "_" )
~~~

# Example

**Challenge:**

~~~http
HTTP/1.1 402 Payment Required
WWW-Authenticate: Payment id="kM9xPqWvT2nJrHsY4aDfEb",
  realm="api.example.com",
  method="monad",
  intent="charge",
  request="eyJhbW91bnQiOiIxMDAwMDAwIiwiY3VycmVuY3kiOiIweDc1NDcwNEJjMDU5RjhDNjcwMTJmRWQ2OUJDOEEzMjdhNWFhZmI2MDMiLCJyZWNpcGllbnQiOiIweDc0MmQzNUNjNjYzNEMwNTMyOTI1YTNiODQ0QmM5ZTc1OTVmOGZFMDAiLCJtZXRob2REZXRhaWxzIjp7ImNoYWluSWQiOjE0M319",
  expires="2025-01-06T12:00:00Z"
Cache-Control: no-store
~~~

The `request` decodes to:

~~~json
{
  "amount": "1000000",
  "currency": "0x754704Bc059F8C67012fEd69BC8A327a5aafb603",
  "recipient": "0x742d35Cc6634C0532925a3b844Bc9e7595f8fE00",
  "methodDetails": {
    "chainId": 143
  }
}
~~~

This requests a transfer of 1.00 USDC (1000000 base units, 6 decimals).

**Credential (pull mode):**

~~~http
GET /api/resource HTTP/1.1
Host: api.example.com
Authorization: Payment eyJjaGFsbGVuZ2UiOnsiaWQiOiJrTTl4UHFXdlQybkpySHNZNGFEZkViIn0sInBheWxvYWQiOnsidHlwZSI6ImF1dGhvcml6YXRpb24iLCJmcm9tIjoiMHgxMjM0NTY3ODkwYWJjZGVmMTIzNDU2Nzg5MGFiY2RlZjEyMzQ1Njc4IiwidG8iOiIweDc0MmQzNUNjNjYzNEMwNTMyOTI1YTNiODQ0QmM5ZTc1OTVmOGZFMDAiLCJ2YWx1ZSI6IjEwMDAwMDAiLCJ2YWxpZEFmdGVyIjoiMCIsInZhbGlkQmVmb3JlIjoiMTczNTk5MjAwMCIsIm5vbmNlIjoiMHhhYmNkZWYuLi4iLCJzaWduYXR1cmUiOiIweGFiY2RlZi4uLiJ9LCJzb3VyY2UiOiJkaWQ6cGtoOmVpcDE1NToxNDM6MHgxMjM0NTY3ODkwYWJjZGVmMTIzNDU2Nzg5MGFiY2RlZjEyMzQ1Njc4In0
~~~
