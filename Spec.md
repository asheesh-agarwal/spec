# Fernet Spec

This document describes version 0x81 & 0x82 of the fernet2 format.

Conceptually, fernet takes a user-provided *message* (an arbitrary
sequence of bytes), a *key* (256 bits), and the current
time, and produces a *token*, which contains the message in a form
that can't be read or altered without the key.

Fernet2 takes a user-provided *message* (an arbitrary
sequence of bytes), a *key* (256 bits), and the *associated data* 
(an arbitary sequence of bytes) and produces a *token*, which contains 
the message in a form that can't be read or altered without the key 
and associated data.

PwFernet takes a user-provided *message* (an arbitrary
sequence of bytes), a *password* (arbitary sequence of bytes which is used 
to derive keys), and the *associated data* (an arbitary sequence of bytes) 
and produces a *token*, which contains the message in a form that 
can't be read or altered without the password and associated data.

To facilitate convenient interoperability between fernet and fernet2, 
this spec defines the external format of both tokens and keys.

All encryption in this version is done with AES 128 in CBC mode.

All base 64 encoding is done with the "URL and Filename Safe"
variant, defined in [RFC 4648](http://tools.ietf.org/html/rfc4648#section-5) as "base64url".

## Key Format

A fernet2 *key* is the base64url encoding of the following
fields:

    Signing-key ‖ Encryption-key

- *Signing-key*, 128 bits
- *Encryption-key*, 128 bits

This *key* is further used to derive strong cryptographic keys for 
data signing and message encryption.

A PwFernet *password* is the data bytes greater than 8-bytes 
which will be used to generate strong cryptographic keys for 
data signing and message encryption.

## Token Format

A fernet *token* is the base64url encoding of the
concatenation of the following fields:

    Version ‖ IV ‖ Ciphertext ‖ HMAC

- *Version*, 8 bits
- *IV*, 128 bits
- *Ciphertext*, variable length, multiple of 128 bits
- *HMAC*, 256 bits

Fernet2 and PwTokens tokens are not self-delimiting. It is assumed 
that the transport will provide a means of finding the length of 
each complete fernet token.

## Token Fields

### Version

This field denotes which version of the format is being used by
the token. Currently there are two versions defined, with the
value 128 (0x81 for Fernet2 & 0x82 PwFernet).

### IV

The 128-bit Initialization Vector used in AES encryption and
decryption of the Ciphertext. This is also used as salt while
deriving keys.

When generating new fernet tokens, the IV must be chosen uniquely
for every token. With a high-quality source of entropy, random
selection will do this with high probability.

### Ciphertext

This field has variable size, but is always a multiple of 128
bits, the AES block size. It contains the original input message,
padded and encrypted.

### HMAC

This field is the 256-bit SHA256 HMAC, under signing-key, of the
concatenation of the following fields:

    Version ‖ IV ‖ Ciphertext ‖ Associated Data

Note that the HMAC input is the entire rest of the token verbatim,
and that this input is *not* base64url encoded.

## Generating

Given a key or password and message, generate a fernet token with the
following steps, in order:

1. Derive strong cryptographic keys using the key in Fernet2 and 
   using password in PwFernet
2. Choose a unique IV.
3. Construct the ciphertext:
   1. Pad the message to a multiple of 16 bytes (128 bits) per [RFC
   5652, section 6.3](http://tools.ietf.org/html/rfc5652#section-6.3).
   This is the same padding technique used in [PKCS #7
   v1.5](http://tools.ietf.org/html/rfc2315#section-10.3) and all
   versions of SSL/TLS (cf. [RFC 5246, section
   6.2.3.2](http://tools.ietf.org/html/rfc5246#section-6.2.3.2) for
   TLS 1.2).
   2. Encrypt the padded message using AES 128 in CBC mode with
   the chosen IV and user-supplied encryption-key.
4. Compute the HMAC field as described above using the
user-supplied signing-key.
5. Concatenate all fields together in the format above.
6. base64url encode the entire token.

## Verifying

Given a key or password and token, to verify that the token is valid and
recover the original message, perform the following steps, in
order:

1. base64url decode the token.
2. Ensure the first byte of the token is 0x80 for Fernet, 0x81 for Fernet2 
   and 0x82 for PwFernet.
3. If token is 0x80 then handle the token as per Fernet [specs]
   (https://github.com/fernet/spec/blob/master/Spec.md)
4. Derive keys using user-supplied key for Fernet2 and password for PwFernet. 
5. Recompute the HMAC from the other fields and the signing-key.
6. Ensure the recomputed HMAC matches the HMAC field stored in the
token.
7. Decrypt the ciphertext field using AES 128 in CBC mode with the
recorded IV and the derived encryption-key.
8. Unpad the decrypted plaintext, yielding the original message.
