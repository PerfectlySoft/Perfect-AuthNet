# Perfect AuthNet

<p align="center">
    <img src="https://img.shields.io/badge/Swift-6.2%2B-orange.svg?style=flat" alt="Swift 6.2+">
    <img src="https://img.shields.io/badge/Platforms-macOS%2012%2B%20%7C%20Linux-lightgray.svg?style=flat" alt="Platforms macOS 12+ | Linux">
    <a href="LICENSE" target="_blank">
        <img src="https://img.shields.io/badge/License-Apache%202.0-lightgrey.svg?style=flat" alt="License Apache 2.0">
    </a>
</p>

A small Swift client for the [Authorize.net](https://developer.authorize.net/) payment gateway's
JSON API (`request.api`). It doesn't support the legacy AIM/SIM name-value interface.

- No package dependencies: Foundation only (`URLSession` and `Codable`).
- Charges Accept.js and Accept Mobile payment nonces (`opaqueData`), so raw card numbers never
  reach your server.
- Swift 6 strict concurrency; every public type is `Sendable`.

## Requirements

- Swift 6.2 or later (`swift-tools-version: 6.2`, Swift 6 language mode).
- macOS 12 or later, or Linux (`FoundationNetworking`). CI covers Swift 6.2 and 6.4 on Ubuntu
  24.04 and Xcode 26.0.1 and 26.6 on macOS, in debug and release, with warnings as errors.

## Using in your project

```swift
.package(url: "https://github.com/PerfectlySoft/Perfect-AuthNet.git", from: "1.0.0"),
```

```swift
.product(name: "PerfectAuthNet", package: "Perfect-AuthNet"),
```

## Quick start

```swift
import PerfectAuthNet

let client = AuthNetClient(
    loginID: "your_api_login_id",
    transactionKey: "your_transaction_key",
    environment: .sandbox          // .production when you go live
)

// Charge a nonce produced by Accept.js or Accept Mobile on the client.
let result = try await client.charge(
    amount: 49.99,
    payment: .nonce(value: opaqueDataValueFromClient),
    order: OrderInfo(invoiceNumber: "INV-1001", description: "Order 1001")
)

if result.approved {
    print("charged: transaction \(result.transactionID)")
} else {
    print("declined: \(result.error ?? "unknown reason")")
}
```

A declined card isn't an error. `charge` returns a `TransactionResult` with `approved == false`.
`AuthNetError` is thrown only for transport failures, malformed responses, and gateway-level
rejections such as bad credentials or validation failures.

## Payment input

```swift
// Preferred: a nonce from the Accept.js or Accept Mobile SDK.
.nonce(value: "<opaque data value>")
.opaqueData(descriptor: "COMMON.ACCEPT.INAPP.PAYMENT", value: "<value>")

// Server-to-server or testing only. You take on the PCI scope.
.creditCard(number: "4111111111111111", expiration: "2030-12", code: "123")
```

## Operations

```swift
// Authorize and capture in one step (a standard charge).
try await client.charge(amount: 49.99, payment: .nonce(value: nonce))

// Authorize now, capture later.
let auth = try await client.authorize(amount: 49.99, payment: .nonce(value: nonce))
try await client.capture(transactionID: auth.transactionID)             // full amount
try await client.capture(transactionID: auth.transactionID, amount: 40) // or less

// Refund a settled transaction (needs the original card's last four digits and expiration).
try await client.refund(transactionID: txnID, amount: 49.99,
                        cardLastFour: "1111", expiration: "2030-12")

// Void an unsettled transaction.
try await client.void(transactionID: txnID)
```

## Result

```swift
public struct TransactionResult {
    let responseCode: ResponseCode   // .approved, .declined, .error or .heldForReview
    let transactionID: String        // for a later capture, refund or void
    let authCode: String?
    let accountNumber: String?       // masked, e.g. "XXXX1111"
    let accountType: String?         // e.g. "Visa"
    let avsResultCode: String?
    let cvvResultCode: String?
    let message: String?             // "code: description"
    let error: String?               // "code: text" for declines and errors
    var approved: Bool               // responseCode == .approved
}
```

## Testing

The offline tests (request encoding, response decoding, BOM handling, amount formatting) use a
mocked `URLSession` and need no network:

```sh
swift test
```

The live tests run against the Authorize.net sandbox (`apitest.authorize.net`) when you supply
sandbox credentials:

```sh
AUTHNET_TESTS=1 AUTHNET_LOGIN_ID=... AUTHNET_TRANSACTION_KEY=... swift test
```

## Notes

- The JSON API prefixes its responses with a UTF-8 byte-order mark. The client strips it before
  decoding.
- To pin certificates or inject a test session, pass your own `URLSession` to
  `AuthNetClient(urlSession:)`.
- Not yet covered: customer profiles (CIM, saved cards), recurring billing (ARB), and webhooks.
- This package hasn't handled production traffic yet. Test against the sandbox before going
  live.

## License

Apache 2.0. See [LICENSE](LICENSE).
