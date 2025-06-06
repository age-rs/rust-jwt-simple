[![GitHub CI](https://github.com/jedisct1/rust-jwt-simple/workflows/Rust/badge.svg)](https://github.com/jedisct1/rust-jwt-simple/actions)
[![Docs.rs](https://docs.rs/jwt-simple/badge.svg)](https://docs.rs/jwt-simple/)
[![crates.io](https://img.shields.io/crates/v/jwt-simple.svg)](https://crates.io/crates/jwt-simple)

<!-- @import "[TOC]" {cmd="toc" depthFrom=1 depthTo=6 orderedList=false} -->

<!-- code_chunk_output -->

- [JWT-Simple](#jwt-simple)
  - [Usage](#usage)
  - [Authentication (symmetric, `HS*` JWT algorithms) example](#authentication-symmetric-hs-jwt-algorithms-example)
    - [Keys and tokens creation](#keys-and-tokens-creation)
    - [Token verification](#token-verification)
  - [Signatures (asymmetric, `RS*`, `PS*`, `ES*` and `EdDSA` algorithms) example](#signatures-asymmetric-rs-ps-es-and-eddsa-algorithms-example)
    - [Key pairs and tokens creation](#key-pairs-and-tokens-creation)
      - [ES256](#es256)
      - [ES384](#es384)
  - [Advanced usage](#advanced-usage)
    - [Custom claims](#custom-claims)
    - [Peeking at metadata before verification](#peeking-at-metadata-before-verification)
    - [Creating and attaching key identifiers](#creating-and-attaching-key-identifiers)
    - [Mitigations against replay attacks](#mitigations-against-replay-attacks)
    - [Salted keys](#salted-keys)
    - [CWT (CBOR) support](#cwt-cbor-support)
    - [Specifying header options](#specifying-header-options)
    - [Validating content and signature types](#validating-content-and-signature-types)
  - [Working around compilation issues with the `boring` crate](#working-around-compilation-issues-with-the-boring-crate)
  - [Usage in Web browsers](#usage-in-web-browsers)
  - [Why yet another JWT crate](#why-yet-another-jwt-crate)

<!-- /code_chunk_output -->

# JWT-Simple

A new JWT (JSON Web Tokens) implementation for Rust that focuses on simplicity, while avoiding common JWT security pitfalls.

`jwt-simple` is unopinionated and supports all commonly deployed authentication and signature algorithms:

| JWT algorithm name | Description                           |
| ------------------ | ------------------------------------- |
| `HS256`            | HMAC-SHA-256                          |
| `HS384`            | HMAC-SHA-384                          |
| `HS512`            | HMAC-SHA-512                          |
| `BLAKE2B`          | BLAKE2B-256                           |
| `RS256`            | RSA with PKCS#1v1.5 padding / SHA-256 |
| `RS384`            | RSA with PKCS#1v1.5 padding / SHA-384 |
| `RS512`            | RSA with PKCS#1v1.5 padding / SHA-512 |
| `PS256`            | RSA with PSS padding / SHA-256        |
| `PS384`            | RSA with PSS padding / SHA-384        |
| `PS512`            | RSA with PSS padding / SHA-512        |
| `ES256`            | ECDSA over p256 / SHA-256             |
| `ES384`            | ECDSA over p384 / SHA-384             |
| `ES256K`           | ECDSA over secp256k1 / SHA-256        |
| `EdDSA`            | Ed25519                               |

`jwt-simple` can be compiled out of the box to WebAssembly/WASI. It is fully compatible with Fastly _Compute_ service.

Important: JWT's purpose is to verify that data has been created by a party knowing a secret key. It does not provide any kind of confidentiality: JWT data is simply encoded as Base64, and is not encrypted.

## Usage

`cargo.toml`:

```toml
[dependencies]
jwt-simple = "0.12"
```

Rust:

```rust
use jwt_simple::prelude::*;
```

Errors are returned as `jwt_simple::Error` values (alias for the `Error` type of the `thiserror` crate).

## Authentication (symmetric, `HS*` JWT algorithms) example

Authentication schemes use the same key for creating and verifying tokens. In other words, both parties need to ultimately trust each other, or else the verifier could also create arbitrary tokens.

### Keys and tokens creation

Key creation:

```rust
use jwt_simple::prelude::*;

// create a new key for the `HS256` JWT algorithm
let key = HS256Key::generate();
```

A key can be exported as bytes with `key.to_bytes()`, and restored with `HS256Key::from_bytes()`.

Token creation:

```rust
// create claims valid for 2 hours
let claims = Claims::create(Duration::from_hours(2));
let token = key.authenticate(claims)?;
```

-> Done!

### Token verification

```rust
let claims = key.verify_token::<NoCustomClaims>(&token, None)?;
```

-> Done! No additional steps required.

Key expiration, start time, authentication tags, etc. are automatically verified. The function fails with `JWTError::InvalidAuthenticationTag` if the authentication tag is invalid for the given key.

The full set of claims can be inspected in the `claims` object if necessary. `NoCustomClaims` means that only the standard set of claims is used by the application, but application-defined claims can also be supported.

Extra verification steps can optionally be enabled via the `ValidationOptions` structure:

```rust
let mut options = VerificationOptions::default();
// Accept tokens that will only be valid in the future
options.accept_future = true;
// Accept tokens even if they have expired up to 15 minutes after the deadline,
// and/or they will be valid within 15 minutes.
// Note that 15 minutes is the default, since it is very common for clocks to be slightly off.
options.time_tolerance = Some(Duration::from_mins(15));
// Reject tokens if they were issued more than 1 hour ago
options.max_validity = Some(Duration::from_hours(1));
// Reject tokens if they don't include an issuer from that set
options.allowed_issuers = Some(HashSet::from_strings(&["example app"]));

// see the documentation for the full list of available options

let claims = key.verify_token::<NoCustomClaims>(&token, Some(options))?;
```

Note that `allowed_issuers` and `allowed_audiences` are not strings, but sets of strings (using the `HashSet` type from the Rust standard library), as the application can allow multiple values.

## Signatures (asymmetric, `RS*`, `PS*`, `ES*` and `EdDSA` algorithms) example

A signature requires a key pair: a secret key used to create tokens, and a public key, that can only verify them.

Always use a signature scheme if both parties do not ultimately trust each other, such as tokens exchanged between clients and API providers.

### Key pairs and tokens creation

Key creation:

#### ES256

```rust
use jwt_simple::prelude::*;

// create a new key pair for the `ES256` JWT algorithm
let key_pair = ES256KeyPair::generate();

// a public key can be extracted from a key pair:
let public_key = key_pair.public_key();
```

#### ES384

```rust
use jwt_simple::prelude::*;

// create a new key pair for the `ES384` JWT algorithm
let key_pair = ES384KeyPair::generate();

// a public key can be extracted from a key pair:
let public_key = key_pair.public_key();
```

Keys can be exported as bytes for later reuse, and imported from bytes or, for RSA, from individual parameters, DER-encoded data or PEM-encoded data.

RSA key pair creation, using OpenSSL and PEM importation of the secret key:

```sh
openssl genrsa -out private.pem 2048
openssl rsa -in private.pem -outform PEM -pubout -out public.pem
```

```rust
let key_pair = RS384KeyPair::from_pem(private_pem_file_content)?;
let public_key = RS384PublicKey::from_pem(public_pem_file_content)?;
```

Token creation and verification work the same way as with `HS*` algorithms, except that tokens are created with a key pair, and verified using the corresponding public key.

Token creation:

```rust
// create claims valid for 2 hours
let claims = Claims::create(Duration::from_hours(2));
let token = key_pair.sign(claims)?;
```

Token verification:

```rust
let claims = public_key.verify_token::<NoCustomClaims>(&token, None)?;
```

Available verification options are identical to the ones used with symmetric algorithms.

## Advanced usage

### Custom claims

Claim objects support all the standard claims by default, and they can be set directly or via convenient helpers:

```rust
let claims = Claims::create(Duration::from_hours(2)).
    with_issuer("Example issuer").with_subject("Example subject");
```

But application-defined claims can also be used. These simply have to be present in a serializable type (this requires the `serde` crate):

```rust
#[derive(Serialize, Deserialize)]
struct MyAdditionalData {
   user_is_admin: bool,
   user_country: String,
}
let my_additional_data = MyAdditionalData {
   user_is_admin: false,
   user_country: "FR".to_string(),
};
```

Claim creation with custom data:

```rust
let claims = Claims::with_custom_claims(my_additional_data, Duration::from_secs(30));
```

Claim verification with custom data. Note the presence of the custom data type:

```rust
let claims = public_key.verify_token::<MyAdditionalData>(&token, None)?;
let user_is_admin = claims.custom.user_is_admin;
```

### Peeking at metadata before verification

Properties such as the key identifier can be useful prior to tag or signature verification in order to pick the right key out of a set.

```rust
let metadata = Token::decode_metadata(&token)?;
let key_id = metadata.key_id();
let algorithm = metadata.algorithm();
// all other standard properties are also accessible
```

**IMPORTANT:** neither the key ID nor the algorithm can be trusted. This is an unfixable design flaw of the JWT standard.

As a result, `algorithm` should be used only for debugging purposes, and never to select a key type.
Similarly, `key_id` should be used only to select a key in a set of keys made for the same algorithm.

At the bare minimum, verification using `HS*` must be prohibited if a signature scheme was originally used to create the token.

### Creating and attaching key identifiers

Key identifiers indicate to verifiers what public key (or shared key) should be used for verification.
They can be attached at any time to existing shared keys, key pairs and public keys:

```rust
let public_key_with_id = public_key.with_key_id(&"unique key identifier");
```

Instead of delegating this to applications, `jwt-simple` can also create such an identifier for an existing key:

```rust
let key_id = public_key.create_key_id();
```

This creates a text-encoded identifier for the key, attaches it, and returns it.

If an identifier has been attached to a shared key or a key pair, tokens created with them will include it.

### Mitigations against replay attacks

`jwt-simple` includes mechanisms to mitigate replay attacks:

- Nonces can be created and attached to new tokens using the `create_nonce()` claim function. The verification procedure can later reject any token that doesn't include the expected nonce (`required_nonce` verification option).
- The verification procedure can reject tokens created too long ago, no matter what their expiration date is. This prevents tokens from malicious (or compromised) signers from being used for too long.
- The verification procedure can reject tokens created before a date. For a given user, the date of the last successful authentication can be stored in a database, and used later along with this option to reject older (replayed) tokens.

### Salted keys

Symmetric keys, such as the ones used with the `HS256`, `HS384`, `HS512` and `BLAKE2B` algorithms, are simple and fast, but have a major downside: signature and verification use the exact same key. Therefore, an adversary having access to the verifier key can forge arbitrary, valid tokens.

Salted keys mitigate this issue in the following way:

- A random signer salt is created and attached to the shared key. This salt is meant to be known only by the signer.
- Another salt is computed from the signer salt and is meant to be used for verification.
- The verifier salt is used to verify the signer salt, which is included in tokens in the `salt` JWT header.

If the verifier has access to tokens, it can forge arbitrary tokens. But given only the verification code and keys, this is impossible. This greatly improves the security of symmetric keys used for verification on 3rd party servers, such as CDNs.

A salt binds to a key, and can be of any length. The `generate_with_salt()` function generates both a random symmetric key, and a 32-byte salt.

Example usage:

```rust
// Create a random key and a signer salt
let key = HS256Key::generate_with_salt();
let claims = Claims::create(Duration::from_secs(86400));
let token = key.authenticate(claims).unwrap();
```

A salt is a `Salt` enum, because it can be either a salt for signing, or a salt for verification.
It can be saved and restored:

```rust
// Get the salt
let salt = key.salt();
// Attach an existing salt to a key
key.attach_salt(salt)?;
```

Given a signer salt, the corresponding verifier salt can be computed:

```rust
// Compute the verifier salt, given a signer salt
let verifier_salt = key.verifier_salt()?;
```

The verifier salt doesn't have to be secret, and can even be hard-coded in the verification code.

Verification:

```rust
let verifier_salt = Salt::Verifier(verifier_salt_bytes);
key.attach_salt(verifier_salt)?;
let claims = key.verify_token::<NoCustomClaims>(&token, None)?;
```

### CWT (CBOR) support

The development code includes a `cwt` cargo feature that enables experimental parsing and validation of CWT tokens.

Please note that CWT doesn't support custom claims. The required identifiers [haven't been standardized yet](https://www.iana.org/assignments/cwt/cwt.xhtml).

Also, the existing Rust crates for JSON and CBOR deserialization are not safe. An untrusted party can send a serialized object that requires a lot of memory and CPU to deserialize. Band-aids have been added for JSON, but with the current Rust tooling, it would be tricky to implement for CBOR.

As a mitigation, we highly recommend rejecting tokens that would be too large in the context of your application. That can be done with the `max_token_length` verification option.

### Specifying header options

It is possible to change the content type (`cty`) and signature type (`typ`) fields of a signed JWT by using the `sign_with_options`/`authenticate_with_options` functions and passing in a `HeaderOptions` struct:

``` rust
let options = HeaderOptions {
   content_type: Some("foo".into()),
   signature_type: Some("foo+JWT".into()),
   ..Default::default()
};
key_pair.sign_with_options(claims, &options).unwrap();
```

By default, generated JWTs will have a signature type field containing the string "JWT", and the content type field will not be present.

### Validating content and signature types

By default, `jwt_simple` ignores the `content_type` field when doing validation, and checks `signature_type` to ensure it is either exactly `JWT` or ends in `+JWT`, case insensitive, if it is present. Both fields may instead be case-insensitively compared against an expected string:

```rust
options.required_signature_type = Some("JWT".into());
options.required_content_type = Some("foo+jwt".into());
```

When validating CWTs, note that CWTs do not have a `content_type` field in their header, and therefore attempting to match a specific one by setting `required_content_type` during validation will **always result in an error**.


## Working around compilation issues with the `boring` crate

As a temporary workaround for portability issues with one of the dependencies (the `boring` crate), this library can be compiled to use only Rust implementations.

In order to do so, import the crate with `default-features = false, features = ["pure-rust"]` in your Cargo configuration.

Do not do it unconditionally. This is only required for very specific setups and targets, and only until issues with the `boring` crate have been solved. The way to configure this in Cargo may also change in future versions.

Static builds targeting the `musl` library don't require that workaround. Just use [`cargo-zigbuild`](https://github.com/rust-cross/cargo-zigbuild) to build your project.

## Usage in Web browsers

The `wasm32-freestanding` target (still sometimes called `wasm32-unknown-unknown` in Rust) is supported (as in "it compiles").

However, using a native JavaScript implementation is highly recommended instead. There are high-quality JWT implementations in JavaScript, leveraging the WebCrypto API, that provide better performance and security guarantees than a WebAssembly module.

## Why yet another JWT crate

This crate is not an endorsement of JWT. JWT is [an awful design](https://tools.ietf.org/html/rfc8725), and one of the many examples that "but this is a standard" doesn't necessarily mean that it is good.

I would highly recommend [PASETO](https://github.com/paragonie/paseto) or [Biscuit](https://github.com/CleverCloud/biscuit) instead if you control both token creation and verification.

However, JWT is still widely used in the industry, and remains absolutely mandatory to communicate with popular APIs.

This crate was designed to:

- Be simple to use, even for people who are new to Rust
- Avoid common JWT API pitfalls
- Support features widely in use. I'd love to limit the algorithm choices to Ed25519, but other methods are required to connect to existing APIs, so we provide them (with the exception of the `None` signature method for obvious reasons).
- Minimize code complexity and external dependencies
- Automatically perform common tasks to prevent misuse. Signature verification and claims validation happen automatically instead of relying on applications.
- Still allow power users to access everything JWT tokens include if they really need to
- Work out of the box in a WebAssembly environment, so that it can be used in function-as-a-service platforms.
