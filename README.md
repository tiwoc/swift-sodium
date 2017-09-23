# Sodium [![Carthage compatible](https://img.shields.io/badge/Carthage-compatible-4BC51D.svg?style=flat)](https://github.com/Carthage/Carthage)
[![Make a donation to support this project](https://img.shields.io/badge/donate-PayPal-green.svg?style=flat)](https://www.libsodium.org/donate)

Swift-Sodium
============

Swift-Sodium provides a safe and easy to use interface to perform
common cryptographic operations on iOS and OSX.

It leverages the [Sodium](https://download.libsodium.org/doc/) library,
and although Swift is the primary target, the framework can also be used in
Objective C.

Usage
=====

Add `Sodium.framework` as a dependency to your project, and import the module:
```swift
import Sodium
```

The Sodium library itself doesn't have to be installed on the system: the
repository already includes a precompiled library for armv7, armv7s,
arm64, as well as for the iOS simulator.

It targets Swift 4, introduced in Xcode 9.0.

The `libsodium-ios.a` file has been generated by the
[dist-build/ios.sh](https://github.com/jedisct1/libsodium/blob/master/dist-build/ios.sh)
script.
The `libsodium-osx.a` file has been generated by the
[dist-build/osx.sh](https://github.com/jedisct1/libsodium/blob/master/dist-build/osx.sh)
script.

Running these scripts on Xcode 9.0 (`9A235`) on the revision
`94550cefd5921c09a1ea352c759851b7ad0e592b` of libsodium generates files
identical to the ones present in this repository.

Public-key Cryptography
=======================

Authenticated Encryption
------------------------

```swift
let sodium = Sodium(())!
let aliceKeyPair = sodium.box.keyPair()!
let bobKeyPair = sodium.box.keyPair()!
let message = "My Test Message".data(using:.utf8)!

let encryptedMessageFromAliceToBob: Data =
    sodium.box.seal(message: message,
                    recipientPublicKey: bobKeyPair.publicKey,
                    senderSecretKey: aliceKeyPair.secretKey)!

let messageVerifiedAndDecryptedByBob =
    sodium.box.open(nonceAndAuthenticatedCipherText: encryptedMessageFromAliceToBob,
                    senderPublicKey: aliceKeyPair.publicKey,
                    recipientSecretKey: bobKeyPair.secretKey)
```

`seal()` automatically generates a nonce and prepends it to the
ciphertext. `open()` extracts the nonce and decrypts the ciphertext.

Optionally, `Box` provides the ability to utilize a user-defined nonce via
`seal(message: recipientPublicKey: senderSecretKey: nonce:)`.

The `Box` class also provides alternative functions and parameters to
deterministically generate key pairs, to retrieve the nonce and/or the
authenticator, and to detach them from the original message.

Anonymous Encryption (Sealed Boxes)
-----------------------------------

```swift
let sodium = Sodium()
let bobKeyPair = sodium.box.keyPair()!
let message = "My Test Message".data(using:.utf8)!

let encryptedMessageToBob =
    sodium.box.seal(message: message, recipientPublicKey: bobKeyPair.publicKey)!

let messageDecryptedByBob =
    sodium.box.open(anonymousCipherText: encryptedMessageToBob,
                    recipientPublicKey: bobKeyPair.publicKey,
                    recipientSecretKey: bobKeyPair.secretKey)
```

`seal()` generates an ephemeral keypair, uses the ephemeral secret
key in the encryption process, combines the ephemeral public key with
the ciphertext, then destroys the keypair. The sender can not decrypt
the resulting ciphertext. `open()` extracts the public key and decrypts
using the recipient's secret key. Message integrity is verified, but
the sender's identity cannot be correlated to the ciphertext.

Public-key signatures
=====================

Detached signatures
-------------------

```swift
let sodium = Sodium()
let message = "My Test Message".data(using:.utf8)!
let keyPair = sodium.sign.keyPair()!
let signature = sodium.sign.signature(message: message, secretKey: keyPair.secretKey)!
if sodium.sign.verify(message: message,
                      publicKey: keyPair.publicKey,
                      signature: signature) {
    // signature is valid
}
```

Attached signatures
-------------------

```swift
let sodium = Sodium()
let message = "My Test Message".data(using:.utf8)!
let keyPair = sodium.sign.keyPair()!
let signedMessage = sodium.sign.sign(message: message, secretKey: keyPair.secretKey)!
if let unsignedMessage = sodium.sign.open(signedMessage: signedMessage, publicKey: keyPair.publicKey) {
    // signature is valid
}
```

Secret-key authenticated encryption
===================================

Authenticated encryption for a sequence of messages
---------------------------------------------------

```swift
let sodium = Sodium()
let message1 = "Message 1".data(using:.utf8)!
let message2 = "Message 2".data(using:.utf8)!
let message3 = "Message 3".data(using:.utf8)!

let secretkey = sodium.secretStream.xchacha20poly1305.key()!

/* stream encryption */

let stream_enc = sodium.secretStream.xchacha20poly1305.initPush(secretKey: secretkey)!
let header = stream_enc.header()
let encrypted1 = stream_enc.push(message: message1)!
let encrypted2 = stream_enc.push(message: message2)!
let encrypted3 = stream_enc.push(message: message3, tag: .FINAL)!

/* stream decryption */

let stream_dec = sodium.secretStream.xchacha20poly1305.initPull(secretKey: secretkey, header: header)!
let (message1_dec, tag1) = stream_dec.pull(cipherText: encrypted1)!
let (message2_dec, tag2) = stream_dec.pull(cipherText: encrypted2)!
let (message3_dec, tag3) = stream_dec.pull(cipherText: encrypted3)!
```

Authenticated encryption for independent messages
-------------------------------------------------

```swift
let sodium = Sodium()
let message = "My Test Message".data(using:.utf8)!
let secretKey = sodium.secretBox.key()!
let encrypted: Data = sodium.secretBox.seal(message: message, secretKey: secretKey)!
if let decrypted = sodium.secretBox.open(nonceAndAuthenticatedCipherText: encrypted, secretKey: secretKey) {
    // authenticator is valid, decrypted contains the original message
}
```

Hashing
=======

Deterministic hashing
---------------------

```swift
let sodium = Sodium()
let message = "My Test Message".data(using:.utf8)!
let h = sodium.genericHash.hash(message: message)
```

Keyed hashing
-------------

```swift
let sodium = Sodium()
let message = "My Test Message".data(using:.utf8)!
let key = "Secret key".data(using:.utf8)!
let h = sodium.genericHash.hash(message: message, key: key)
```

Streaming
---------

```swift
let sodium = Sodium()
let message1 = "My Test ".data(using:.utf8)!
let message2 = "Message".data(using:.utf8)!
let key = "Secret key".data(using:.utf8)!
let stream = sodium.genericHash.initStream(key: key)!
stream.update(input: message1)
stream.update(input: message2)
let h = stream.final()
```

Short-output hashing (SipHash)
==============================

```swift
let sodium = Sodium()
let message = "My Test Message".data(using:.utf8)!
let key = sodium.randomBytes.buf(length: sodium.shortHash.KeyBytes)!
let h = sodium.shortHash.hash(message: message, key: key)
```

Random numbers generation
=========================

```swift
let sodium = Sodium()
let randomData = sodium.randomBytes.buf(length: 1000)!
let seed = "0123456789abcdef0123456789abcdef".data(using:.utf8)!
let stream = sodium.randomBytes.deterministic(length: 1000, seed: seed)!
```

Password hashing
================

```swift
let sodium = Sodium()
let password = "Correct Horse Battery Staple".data(using:.utf8)!
let hashedStr = sodium.pwHash.str(passwd: password,
                                  opsLimit: sodium.pwHash.OpsLimitInteractive,
                                  memLimit: sodium.pwHash.MemLimitInteractive)!

if sodium.pwHash.strVerify(hash: hashedStr, passwd: password) {
    // Password matches the given hash string
} else {
    // Password doesn't match the given hash string
}

if sodium.pwHash.strNeedsRehash(hash: hashedStr,
                                opsLimit: sodium.pwHash.OpsLimitInteractive,
                                memLimit: sodium.pwHash.MemLimitInteractive) {
    // Previously hashed password should be recomputed because the way it was
    // hashed doesn't match the current algorithm and the given parameters.
}
```

Key exchange
============

```swift
let sodium = Sodium()
let aliceKeyPair = sodium.keyExchange.keyPair()!
let bobKeyPair = sodium.keyExchange.keyPair()!

let sessionKeyPairForAlice = sodium.keyExchange.sessionKeyPair(publicKey: aliceKeyPair.publicKey,
    secretKey: aliceKeyPair.secretKey, otherPublicKey: bobKeyPair.publicKey, side: .CLIENT)!
let sessionKeyPairForBob = sodium.keyExchange.sessionKeyPair(publicKey: bobKeyPair.publicKey,
    secretKey: bobKeyPair.secretKey, otherPublicKey: aliceKeyPair.publicKey, side: .SERVER)!

let aliceToBobKeyEquality = sodium.utils.equals(sessionKeyPairForAlice.tx, sessionKeyPairForBob.xx) // true
let bobToAliceKeyEquality = sodium.utils.equals(sessionKeyPairForAlice.rx, sessionKeyPairForBob.tx) // true
```

Authentication tags
===================

The `sodium.auth.tag()` function computes an authentication tag (HMAC) using
a message and a key. Parties knowing the key can then verify the authenticity
of the message using the same parameters and the `sodium.auth.verify()`
function.

Authentication tags are not signatures: the same key is used both for
computing and verifying a tag. Therefore, verifiers can also compute
tags for arbitrary messages.

```swift
let sodium = Sodium()
let input = "test".data(using:.utf8)!
let key = sodium.auth.key()!
let tag = sodium.auth.tag(message: input, secretKey: key)!
let tagIsValid = sodium.auth.verify(message: input, secretKey: key, tag: tag)
```

Key derivation
==============

The `sodium.keyDerivation.derive()` function generates a subkey using
an input (master) key, an index, and a 8 bytes string identifying the
context. Up to (2^64) - 1 subkeys can be generated for each context,
by incrementing the index.

```swift
let sodium = Sodium()
let secretKey = sodium.keyDerivation.keygen()!

let subKey1 = sodium.keyDerivation.derive(secretKey: secretKey,
                                          index: 0, length: 32,
                                          context: "Context!")
let subKey2 = sodium.keyDerivation.derive(secretKey: secretKey,
                                          index: 1, length: 32,
                                          context: "Context!")
```

Utilities
=========

Zeroing memory
--------------

```swift
let sodium = Sodium()
var dataToZero = "Message".data(using:.utf8)!
sodium.utils.zero(&dataToZero)
```

Constant-time comparison
------------------------

```swift
let sodium = Sodium()
let secret1 = "Secret key".data(using:.utf8)!
let secret2 = "Secret key".data(using:.utf8)!
let equality = sodium.utils.equals(secret1, secret2)
```

Padding
-------

```swift
let sodium = Sodium()
var data = "test".toData()!

// make data.count a multiple of 16
sodium.utils.pad(data: &data, blockSize: 16)!

// restore original size
sodium.utils.unpad(data: &data, blockSize: 16)!
```

Constant-time hexadecimal encoding
----------------------------------

```swift
let sodium = Sodium()
let data = "Secret key".data(using:.utf8)!
let hex = sodium.utils.bin2hex(data)
```

Hexadecimal decoding
--------------------

```swift
let sodium = Sodium()
let data1 = sodium.utils.hex2bin("deadbeef")
let data2 = sodium.utils.hex2bin("de:ad be:ef", ignore: " :")
```

Constant-time base64 encoding
-----------------------------

```swift
let sodium = Sodium()
let b64 = sodium.utils.bin2base64("data".toData()!)!
let b64_2 = sodium.utils.bin2base64("data".toData()!, variant: .URLSAFE_NO_PADDING)!
```

Base64 decoding
---------------

```swift
let data1 = sodium.utils.base642bin(b64)
let data2 = sodium.utils.base642bin(b64, ignore: " \n")
let data3 = sodium.utils.base642bin(b64_2, variant: .URLSAFE_NO_PADDING, ignore: " \n")
```

Helpers to build custom constructions
=====================================

Only use the functions below if you know that you absolutely need
them, and know how to use them correctly.

Unauthenticated encryption
--------------------------

The `sodium.stream.xor()` function combines (using the XOR operation)
an arbitrary-long input with the output of a deterministic key stream
derived from a key and a nonce. The same operation applied twice
produces the original input.

No authentication tag is added to the output. The data can be tampered
with; an adversary can flip arbitrary bits.

In order to encrypt data using a secret key, the `SecretBox` class is
likely to be what you are looking for.

In order to generate a deterministic stream out of a seed, the
`RandomBytes.deterministic_rand()` function is likely to be what you
need.

```swift
let sodium = Sodium()
let input = "test".data(using:.utf8)!
let key = sodium.stream.key()!
let (output, nonce) = sodium.stream.xor(input: input, secretKey: key)!
let twice = sodium.stream.xor(input: output, nonce: nonce, secretKey: key)!

XCTAssertEqual(input, twice)
```
