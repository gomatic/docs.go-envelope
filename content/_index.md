---
title: go-envelope
---

`go-envelope` is the gomatic ecosystem's **envelope (hybrid) encryption and signing** library (package `envelope`). Plaintext is sealed with AES-256-GCM under a per-message key that is itself wrapped to the recipient's RSA public key; messages are signed and verified with ed25519, and private keys are bundled under an Argon2id-derived key. Message persistence is a consumer concern — this package owns only the cryptography.

- **Source:** [`gomatic/go-envelope`](https://github.com/gomatic/go-envelope)
- **API reference:** [pkg.go.dev/github.com/gomatic/go-envelope](https://pkg.go.dev/github.com/gomatic/go-envelope)

## Install

```sh
go get github.com/gomatic/go-envelope
```

## Key generation and bundles

`GenerateKeyPair` creates an ed25519 key pair whose private key is encrypted with AES-256-GCM under a key derived from the passphrase via **Argon2id**; the passphrase and derived key are zeroed after use. `DecryptPrivateKey` recovers the private key from the bundle:

```go
bundle, err := envelope.GenerateKeyPair(envelope.Passphrase("correct horse battery staple"))
// bundle.PublicKey carries {Type, PEM, Fingerprint}; bundle.EncryptedPrivateKey and bundle.Salt round-trip through storage.

priv, err := envelope.DecryptPrivateKey(*bundle, passphrase)
```

`ParseEd25519PublicKey` is the single reader that turns a stored PEM public key (the canonical `ED25519 PUBLIC KEY` block written by `GenerateKeyPair`) back into a verifiable [`ed25519.PublicKey`](https://pkg.go.dev/crypto/ed25519#PublicKey), so the wire format lives in exactly one place alongside its writer.

## Signing and verification

```go
sig, err := envelope.Sign(envelope.Message(body), priv)
ok, err := envelope.Verify(envelope.Message(body), sig, publicKeyPEM)
```

## Hybrid encryption

`EncryptToRecipient` generates a random AES-256 key, encrypts the plaintext with AES-256-GCM, then wraps the AES key to the recipient's RSA public key with OAEP; `DecryptWithKey` unwraps and decrypts on the receiving side. The AES key is zeroed after use, the IV is unique per message, and the GCM auth tag validates integrity. RSA keys below the 2048-bit minimum are rejected with `ErrRSAKeyTooSmall`:

```go
payload, err := envelope.EncryptToRecipient(plaintext, recipientPubKeyPEM, fingerprint)
// payload: {Ciphertext, EncryptedKey, IV, RecipientFingerprint}

plain, err := envelope.DecryptWithKey(*payload, privateKeyPEM)
```

## Errors

Every failure the package can emit is a sentinel — a constant of [`gomatic/go-error`](https://github.com/gomatic/go-error)'s `errs.Const` — matchable with [`errors.Is`](https://pkg.go.dev/errors#Is), never by string comparison. Highlights: `ErrRSAKeyTooSmall`, `ErrDecodePublicKeyPEM`, `ErrNotRSAPublicKey`, `ErrInvalidPrivateKeySize`, `ErrDecryptPrivateKey`. The full list is in the [API reference](https://pkg.go.dev/github.com/gomatic/go-envelope#pkg-constants).

## Design

- **Typed parameters** — `Plaintext`, `Message`, `Signature`, `Passphrase`, `Fingerprint`, `PublicKeyPEM`, `PrivateKeyPEM` name the domain concepts instead of bare `[]byte`/`string`, so call sites cannot transpose arguments.
- **Key hygiene** — passphrases, derived keys, and per-message AES keys are zeroed after use.
- **Pure functions over value types** — no retained state and no injected infrastructure; everything is testable to 100% coverage under the shared gate.

## Provenance

Extracted from `xto-email/go-encryption` (originally `xto-email/xto`'s `internal/encryption`), renamed `encryption` → `envelope` on the move to gomatic. The encrypted message store that shipped alongside the crypto stayed with the consumer (`xtod`) — persistence is deliberately not part of this library.
