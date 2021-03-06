# Noise Extension: Disco

* author:     David Wong (david.wong@nccgroup.trust)
* revision:   `disco-draft-2`
* extending:  `noise-revision-32`
* date:       2017-07-20

## 1. Introduction

### 1.1. Motivation

[Noise](http://noiseprotocol.org/) is a framework for crypto protocols based on Diffie-Hellman key agreement. One of its most interesting property is that every new message depends on all the previous ones. This is done by continuously hashing messages being sent and received, as well as continuously deriving new keys based on the continuous hash and the previous keys. This interesting property stops at the end of the handshake.  
[Strobe](http://strobe.sourceforge.io/) is a protocol framework based on a [duplex construction](http://sponge.noekeon.org/). It naturally benefits from the same property, effectivelly absorbing every operation to influence the next ones. The Strobe specification is comparable to Noise, while focusing on the symmetric part of a protocol. By merging both protocols into one, Disco achieves the following goals:

* The Noise specification can be greatly simplified by removing all the symmetric cryptographic algorithms and symmetric objects. These can be replaced by a single Strobe object.
* Implementations of Noise with the Disco extension will consequently greatly benefit from this simplification, allowing for a drastic reduction of the codebase, facilitating security audits.
* Messages will continue to rely on every previous messages that were sent or received, even in the symmetric part of the protocol.
* The Strobe functions will allow for more flexible and complex symmetric protocols following the handshake.
* Implementations of Noise with the Disco extension will also benefit from the other Strobe functions, which provide on top of a single primitive the following functions: generation of random numbers, derivation of keys, hashing, encryption and authentication.

### 1.2. How to Read This Document

This specification is an extension of the Noise protocol framework revision 32. It relies for the most part on Noise's specification, while heavily modifying its foundations. Major changes are listed in the next section.  
To implement the Disco extension, a Strobe implementation respecting the functions of the [section 3.2 of this document](#3-2-functions) is required. None of the [cryptographic algorithms of Noise](http://noiseprotocol.org/noise.html#crypto-functions) are required. Furthermore, the [CipherState](http://noiseprotocol.org/noise.html#the-cipherstate-object) is not necessary while the [SymmetricState](http://noiseprotocol.org/noise.html#the-symmetricstate-object) has been simplified by Strobe calls. When implementing Noise with the Disco extension, simply ignore the CipherState section of Noise and implement the SymmetricState described in [section 4 of this document](#-4-modifications-to-the-symmetricstate). For advanced features, refer to [section 5](#5-modifications-to-advanced-features).

### 1.3. Major changes

The following list summarizes the major changes brought by this extension:

* Protocol names don't have the symmetric algorithms, but instead the version of Strobe.
* The CipherState object has been removed.
* The SymmetricState object has been simplified with Strobe's calls.
* The Handshake object makes calls to Strobe functions, affecting a unique Strobe state.
* The Handshake returns two Strobe states.

## 2. Protocol naming

The name of a Noise protocol extended with Disco follows the same convention, but replaces the symmetric cryptographic algorithms by the version of Strobe used:

```
Noise_[PATTERN]_[KEYEXCHANGE]_STROBEvX.Y.Z
```

For example, with the current version of [Strobe](https://strobe.sourceforge.io/) being STROBEv1.0.2:

```
Noise_XX_25519_STROBEv1.0.2
```

<!-- TODO: maybe change this to Noise_[PATTERN]_[KEYEXCHANGE]_STROBEvX.Y.Z_PERMUTATION -->

## 3. Strobe Functions

A Strobe state depends on the following associated constants:

* **`R`**: The blocksize of the Strobe state (computed as `N - (2*sec)/8 - 2`, [see section 4 of the Strobe specification](https://strobe.sourceforge.io/specs/#params)).
* **`TAGLEN`**: A constant specifying the size in bytes of the authentication tags generated by `send_MAC`. For security reasons, `TAGLEN` must be 16 or greater.

While a Strobe state responds to many functions ([see Strobe's specification](https://strobe.sourceforge.io/)), only the following ones need to be implemented in order for the Disco extension to work properly:

**`InitializeStrobe(protocol_name)`**: Initialize the Strobe object with a custom protocol name.

**`KEY(key)`**: Permutes the Strobe's state and replaces the new state with the key.

**`PRF(output_len)`**: Permutes the Strobe's state and removes `output_len` bytes from the new state. Outputs the removed bytes to the caller.

**`send_ENC(plaintext)`**: Permutes the Strobe's state and XOR the plaintext with the new state to encrypt it. The new state is replaced by the resulting ciphertext, while the resulting ciphertext is output to the caller.

**`recv_ENC(ciphertext)`**: Permutes the Strobe's state and XOR the ciphertext with the new state to decrypt it. The new state is replaced by the ciphertext, while the resulting plaintext is output to the caller.

**`AD(additionalData)`**: Absorbs the additional data in the Strobe's state.

**`send_MAC(output_length)`**: Permutes the Strobe's state and retrieves the next `output_length` bytes from the new state.

**`recv_MAC(tag)`**: Permutes the Strobe's state and compare (in **constant-time**) the received tag with the next `TAGLEN` bytes from the new state.

**`RATCHET(length)`**: Permutes the Strobe's state and set the next `length` bytes from the new state to zero.

The following **meta** functions:

**`meta_AD(additionalData)`**:
   XOR the additional data in the Strobe's state.

The following function which is not specified in Strobe:

**`Clone()`**: Returns a copy of the Strobe state.


## 4. Modifications to the Symmetric State

A SymmetricState object contains a **Strobe state** and responds to the following functions:

**`InitializeSymmetric(protocol_name)`**: Calls `InitializeStrobe(protocol_name)` on the Strobe state.

**`MixKey(input_key_material)`**: Calls `AD(input_key_material)` on the Strobe state.

**`MixHash(data)`**: Calls `AD(data)` on the Strobe state.

**`MixKeyAndHash(input_key_material)`**: Calls `AD(data)` on the Strobe state.

**`EncryptAndHash(plaintext)`**: `send_ENC(plaintext)` followed by `send_MAC(TAGLEN)` on the Strobe state.

**`DecryptAndHash(ciphertext)`**: `recv_ENC(ciphertext[:-TAGLEN])` followed by `recv_MAC(ciphertext[-TAGLEN:])` on the Strobe state. Abort the connection if the tag is incorrect.

**`Split()`**: Returns a pair of Strobe states for encrypting transport messages by executing the following steps:

* Let `s1` be the Strobe state and `s2` the result returned by `Clone()`.
* Calls `meta_AD("initiator")` on `s1` and `meta_AD("responder")` on `s2`.
* Calls `RATCHET(R)` on `s1` and on `s2`.
* Returns the pair `(s1, s2)`.

## 5. Modifications to Advanced Features

### 5.1 Channel Binding

Right before calling `Split()`, a binding value could be obtained from the StrobeState by calling `PRF()`.

### 5.2 Rekey

To enable this, Strobe supports a `RATCHET()` function.

## 6. Security Considerations

The same security considerations that apply to Noise and Strobe are to be considered.


