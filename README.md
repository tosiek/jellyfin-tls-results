# TLS / client certificate compatibility of the Jellyfin Android apps

Measured TLS and client certificate (mTLS) support of jellyfin-android and jellyfin-androidtv with the custom CA /
client certificate changes, on emulators. Measured with the automated tests in
[tosiek/jellyfin-tls-test-matrix](https://github.com/tosiek/jellyfin-tls-test-matrix).

Tested: Android 6, 7.0, 8, 9, 10, 11, 13, 14, 16, 17 and Android TV 11 and 14.
The app offers several options in every handshake, the server chooses. Security therefore depends on
the certificates AND on the server configuration.

**Short version:** Caddy defaults + client certificates with EC P-384 keys, exported as 3DES .p12 with a long password, is the strongest setup that works on every Android version from 6 to 17 (Android 14+: AES-256 .p12). The weak point is the .p12 file during the transfer, not the connection itself.

## Recommended setup per Android version

| | Android 6 | Android 7.0 | Android 8-9 | Android 10-13 | Android 14-16 | Android 17 |
|---|---|---|---|---|---|---|
| TLS version | TLS 1.2 | TLS 1.2 | TLS 1.2 | TLS 1.3 | TLS 1.3 | TLS 1.3 |
| Key exchange | P-384 (or P-256) | P-256 (no P-384) | X25519 | X25519 | X25519 | X25519MLKEM768 |
| Cipher | ECDHE + AES-GCM | ECDHE + AES-GCM or ChaCha20 | ECDHE + AES-GCM or ChaCha20 | AES-256-GCM or ChaCha20 | AES-256-GCM or ChaCha20 | AES-256-GCM or ChaCha20 |
| Client key | EC P-384 | EC P-384 | EC P-384 | EC P-384 | EC P-384 | EC P-384 |
| .p12 file format | 3DES / SHA-1 | 3DES / SHA-1 | 3DES / SHA-1 | 3DES / SHA-1 | AES-256 / PBKDF2 | AES-256 / PBKDF2 |
| Post-quantum | – | – | – | – | – | X25519MLKEM768 |

**Android 7.0 vs 7.1:** Android 7.0 offers only P-256 for the key exchange (no P-384, P-521 or X25519), a known
limitation of that release, fixed in Android 7.1.1. Only Android 7.0 (API 24) was tested, Android 7.1 (API 25) was
not.

**Client key**
EC P-384 is the strongest key type that works on all versions. P-256 is equally safe in practice
and faster. RSA 4096 works too but is slow on TV boxes. Ed25519 and P-521 do not work on any version.

**.p12 file format**
The encryption only protects the FILE while it is transferred. On import the key is moved into the
Android Keystore (hardware protected on most real devices) and the file is no longer needed.
- Android 6 - 13 require 3DES / SHA-1 (or OpenSSL -legacy). That is weak for the file itself,
  therefore: use a LONG password, transfer the file over a safe channel and DELETE it after importing.
- Only for Android 14+: OpenSSL 3 default (AES-256-CBC / PBKDF2 / HMAC-SHA256).
- Android 17 also reads a .p12 with an unencrypted key; do not use it, the file is then unprotected.

**Post-quantum key exchange**
Only Android 17 offers the hybrid X25519MLKEM768 (ML-KEM), already in the first ClientHello.
The mTLS handshake with it works with EC and RSA client keys (test Caddy 2.11.7). Older versions
fall back to X25519 / P-256, so enabling it on the server costs nothing.

**ChaCha20 vs AES-GCM**
Both are secure. ChaCha20 is faster on cheap TV boxes without AES hardware acceleration.
Allow both.

**Server certificate**
ECDSA P-256 / P-384 (or RSA 2048+). Ed25519 and ML-DSA certificates do not work on any version.

## Server settings (Caddy / nginx)

- Minimum TLS 1.2, TLS 1.3 enabled.
  Android 6 - 9 have no TLS 1.3, a TLS 1.3-only server locks them out.

- TLS 1.2 ciphers: only ECDHE with AEAD (AES-GCM, ChaCha20-Poly1305).
  No CBC, no plain RSA key exchange.
  The app still OFFERS older options (RSA_AES_*_CBC, SHA-1 signatures, and on Android 10 - 14 the
  HttpsURLConnection path used by the mobile ExoPlayer even offers TLS 1.0 / 1.1), so the server
  must not accept them.
  Caddy defaults already do exactly this.
  nginx: ssl_protocols TLSv1.2 TLSv1.3; plus an ECDHE + AEAD only ssl_ciphers list.

- Key exchange groups: X25519MLKEM768, X25519, P-256, P-384.
  Keep P-256, otherwise Android 7.0 cannot connect (it offers only P-256).
  X25519MLKEM768 needs a server that supports it (tested with the Caddy 2.11.7 release build;
  the Ubuntu package of Caddy did not offer it).

- Client certificates: require_and_verify against your own client CA.

## Overview

✅ works · ❌ does not work · ? not measured

| What | Android 6 | Android 7.0 | Android 8–9 | Android 10–13 | Android 14–16 | Android 17 |
| --- | --- | --- | --- | --- | --- | --- |
| All device tests pass | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| Client certificate (mTLS) handshake over TLS 1.2 | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| Client certificate (mTLS) handshake over TLS 1.3 | ❌ | ❌ | ❌ | ✅ | ✅ | ✅ |
| TLS 1.3 offered | ❌ | ❌ | ❌ | ✅ | ✅ | ✅ |
| ChaCha20-Poly1305 cipher suites | ❌ | ✅ | ✅ | ✅ | ✅ | ✅ |
| X25519 key exchange (ECDHE) | ❌ | ❌ | ✅ | ✅ | ✅ | ✅ |
| P-384 (secp384r1) key exchange | ✅ | ❌ | ✅ | ✅ | ✅ | ✅ |
| Post-quantum hybrid key exchange (X25519MLKEM768 / ML-KEM) | ❌ | ❌ | ❌ | ❌ | ❌ | ✅ |
| Post-quantum key share in the first ClientHello (no extra round trip) | ❌ | ❌ | ❌ | ❌ | ❌ | ✅ |
| Post-quantum mTLS handshake with the test Caddy, EC client key | ? | ? | ? | ? | ? | ✅ |
| Post-quantum mTLS handshake with the test Caddy, RSA client key | ? | ? | ? | ? | ? | ✅ |
| Ed25519 or ML-DSA signed server certificates | ❌ | ❌ | ❌ | ❌ | ❌ | ❌ |
| .p12 encrypted with 3DES / SHA-1 (key and certificates) | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| .p12 OpenSSL -legacy (3DES key, RC2-40 certificates) | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| .p12 OpenSSL 3 default (AES-256-CBC / PBKDF2 / HMAC-SHA256) | ❌ | ❌ | ❌ | ❌ | ✅ | ✅ |
| .p12 with unencrypted private key | ❌ | ❌ | ❌ | ❌ | ❌ | ✅ |
| RSA 2048 / 4096 client keys | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| RSA 1024 client keys (accepted with warning) | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| EC P-256 (secp256r1) client keys | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| EC P-384 (secp384r1) client keys | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| EC P-521 (secp521r1) client keys | ❌ | ❌ | ❌ | ❌ | ❌ | ❌ |
| Ed25519 client keys | ❌ | ❌ | ❌ | ❌ | ❌ | ❌ |

Tested emulators: Android 6 = API_23, Android 7.0 = API_24, Android 8–9 = API_26, API_28, Android 10–13 = API_29, API_30, TV_30, API_33, Android 14–16 = API_34, TV_34, API_36, Android 17 = API_37

## Test runs

| Suite | App | API_23 (23) | API_24 (24) | API_26 (26) | API_28 (28) | API_29 (29) | API_30 (30) | TV_30 (30) | API_33 (33) | API_34 (34) | TV_34 (34) | API_36 (36) | API_37 (37) |
| --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- |
| device | tv | ✅ 20/20 | ✅ 20/20 | ✅ 20/20 | ✅ 20/20 | ✅ 20/20 | ✅ 20/20 | ✅ 20/20 | ✅ 20/20 | ✅ 20/20 | ✅ 20/20 | ✅ 20/20 | ✅ 20/20 |
| device | mobile | ✅ 20/20 | ✅ 20/20 | ✅ 20/20 | ✅ 20/20 | ✅ 20/20 | ✅ 20/20 | ✅ 20/20 | ✅ 20/20 | ✅ 20/20 | ✅ 20/20 | ✅ 20/20 | ✅ 20/20 |
| GUI | tv | ✅ 14/14 | ✅ 14/14 | ✅ 14/14 | ✅ 14/14 | ✅ 14/14 | ✅ 14/14 | ✅ 14/14 | ✅ 14/14 | ✅ 14/14 | ✅ 14/14 | ✅ 14/14 | ✅ 14/14 |
| GUI | mobile | ✅ 15/15 | ✅ 15/15 | ✅ 15/15 | ✅ 15/15 | ✅ 15/15 | ✅ 15/15 | ✅ 15/15 | ✅ 15/15 | ✅ 15/15 | ✅ 15/15 | ✅ 15/15 | ✅ 15/15 |
| end-to-end | tv | ✅ 6/6 | ✅ 6/6 | ✅ 6/6 | ✅ 6/6 | ✅ 6/6 | ✅ 6/6 | ✅ 6/6 | ✅ 6/6 | ✅ 6/6 | ✅ 6/6 | ✅ 6/6 | ✅ 6/6 |
| end-to-end | mobile | ✅ 7/7 | ✅ 7/7 | ✅ 7/7 | ✅ 7/7 | ✅ 7/7 | ✅ 7/7 | ✅ 7/7 | ✅ 7/7 | ✅ 7/7 | ✅ 7/7 | ✅ 7/7 | ✅ 7/7 |

## PKCS#12 import (client certificate)

`✅ KS` stored in AndroidKeyStore · `✅ SW` encrypted software key fallback · `❌ enc` encryption not readable · `❌ type` key type rejected · `❌ key` key not decodable

### OpenSSL 3 default (AES-256 / PBKDF2)

| Key | API_23 (23) | API_24 (24) | API_26 (26) | API_28 (28) | API_29 (29) | API_30 (30) | TV_30 (30) | API_33 (33) | API_34 (34) | TV_34 (34) | API_36 (36) | API_37 (37) |
| --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- |
| rsa2048 | ❌ enc | ❌ enc | ❌ enc | ❌ enc | ❌ enc | ❌ enc | ❌ enc | ❌ enc | ✅ KS | ✅ KS | ✅ KS | ✅ KS |
| rsa4096 | ❌ enc | ❌ enc | ❌ enc | ❌ enc | ❌ enc | ❌ enc | ❌ enc | ❌ enc | ✅ KS | ✅ KS | ✅ KS | ✅ KS |
| rsa1024 | ❌ enc | ❌ enc | ❌ enc | ❌ enc | ❌ enc | ❌ enc | ❌ enc | ❌ enc | ✅ KS | ✅ KS | ✅ KS | ✅ KS |
| ec256 | ❌ enc | ❌ enc | ❌ enc | ❌ enc | ❌ enc | ❌ enc | ❌ enc | ❌ enc | ✅ KS | ✅ KS | ✅ KS | ✅ KS |
| ec384 | ❌ enc | ❌ enc | ❌ enc | ❌ enc | ❌ enc | ❌ enc | ❌ enc | ❌ enc | ✅ KS | ✅ KS | ✅ KS | ✅ KS |
| ec521 | ❌ enc | ❌ enc | ❌ enc | ❌ enc | ❌ enc | ❌ enc | ❌ enc | ❌ enc | ❌ type | ❌ type | ❌ type | ❌ type |
| ed25519 | ❌ enc | ❌ enc | ❌ enc | ❌ enc | ❌ enc | ❌ enc | ❌ enc | ❌ enc | ❌ key | ❌ key | ❌ key | ❌ type |

### OpenSSL -legacy (3DES key, RC2-40 certs)

| Key | API_23 (23) | API_24 (24) | API_26 (26) | API_28 (28) | API_29 (29) | API_30 (30) | TV_30 (30) | API_33 (33) | API_34 (34) | TV_34 (34) | API_36 (36) | API_37 (37) |
| --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- |
| rsa2048 | ✅ KS | ✅ KS | ✅ KS | ✅ KS | ✅ KS | ✅ KS | ✅ KS | ✅ KS | ✅ KS | ✅ KS | ✅ KS | ✅ KS |
| rsa4096 | ✅ KS | ✅ KS | ✅ KS | ✅ KS | ✅ KS | ✅ KS | ✅ KS | ✅ KS | ✅ KS | ✅ KS | ✅ KS | ✅ KS |
| rsa1024 | ✅ KS | ✅ KS | ✅ KS | ✅ KS | ✅ KS | ✅ KS | ✅ KS | ✅ KS | ✅ KS | ✅ KS | ✅ KS | ✅ KS |
| ec256 | ✅ KS | ✅ KS | ✅ KS | ✅ KS | ✅ KS | ✅ KS | ✅ KS | ✅ KS | ✅ KS | ✅ KS | ✅ KS | ✅ KS |
| ec384 | ✅ KS | ✅ KS | ✅ KS | ✅ KS | ✅ KS | ✅ KS | ✅ KS | ✅ KS | ✅ KS | ✅ KS | ✅ KS | ✅ KS |
| ec521 | ❌ type | ❌ type | ❌ type | ❌ type | ❌ type | ❌ type | ❌ type | ❌ type | ❌ type | ❌ type | ❌ type | ❌ type |
| ed25519 | ❌ key | ❌ key | ❌ key | ❌ key | ❌ key | ❌ key | ❌ key | ❌ key | ❌ key | ❌ key | ❌ key | ❌ type |

### 3DES / SHA-1

| Key | API_23 (23) | API_24 (24) | API_26 (26) | API_28 (28) | API_29 (29) | API_30 (30) | TV_30 (30) | API_33 (33) | API_34 (34) | TV_34 (34) | API_36 (36) | API_37 (37) |
| --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- |
| rsa2048 | ✅ KS | ✅ KS | ✅ KS | ✅ KS | ✅ KS | ✅ KS | ✅ KS | ✅ KS | ✅ KS | ✅ KS | ✅ KS | ✅ KS |
| rsa4096 | ✅ KS | ✅ KS | ✅ KS | ✅ KS | ✅ KS | ✅ KS | ✅ KS | ✅ KS | ✅ KS | ✅ KS | ✅ KS | ✅ KS |
| rsa1024 | ✅ KS | ✅ KS | ✅ KS | ✅ KS | ✅ KS | ✅ KS | ✅ KS | ✅ KS | ✅ KS | ✅ KS | ✅ KS | ✅ KS |
| ec256 | ✅ KS | ✅ KS | ✅ KS | ✅ KS | ✅ KS | ✅ KS | ✅ KS | ✅ KS | ✅ KS | ✅ KS | ✅ KS | ✅ KS |
| ec384 | ✅ KS | ✅ KS | ✅ KS | ✅ KS | ✅ KS | ✅ KS | ✅ KS | ✅ KS | ✅ KS | ✅ KS | ✅ KS | ✅ KS |
| ec521 | ❌ type | ❌ type | ❌ type | ❌ type | ❌ type | ❌ type | ❌ type | ❌ type | ❌ type | ❌ type | ❌ type | ❌ type |
| ed25519 | ❌ key | ❌ key | ❌ key | ❌ key | ❌ key | ❌ key | ❌ key | ❌ key | ❌ key | ❌ key | ❌ key | ❌ type |

### Unencrypted key

| Key | API_23 (23) | API_24 (24) | API_26 (26) | API_28 (28) | API_29 (29) | API_30 (30) | TV_30 (30) | API_33 (33) | API_34 (34) | TV_34 (34) | API_36 (36) | API_37 (37) |
| --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- |
| rsa2048 | ❌ enc | ❌ enc | ❌ enc | ❌ enc | ❌ enc | ❌ enc | ❌ enc | ❌ enc | ❌ enc | ❌ enc | ❌ enc | ✅ KS |
| rsa4096 | ❌ enc | ❌ enc | ❌ enc | ❌ enc | ❌ enc | ❌ enc | ❌ enc | ❌ enc | ❌ enc | ❌ enc | ❌ enc | ✅ KS |
| rsa1024 | ❌ enc | ❌ enc | ❌ enc | ❌ enc | ❌ enc | ❌ enc | ❌ enc | ❌ enc | ❌ enc | ❌ enc | ❌ enc | ✅ KS |
| ec256 | ❌ enc | ❌ enc | ❌ enc | ❌ enc | ❌ enc | ❌ enc | ❌ enc | ❌ enc | ❌ enc | ❌ enc | ❌ enc | ✅ KS |
| ec384 | ❌ enc | ❌ enc | ❌ enc | ❌ enc | ❌ enc | ❌ enc | ❌ enc | ❌ enc | ❌ enc | ❌ enc | ❌ enc | ✅ KS |
| ec521 | ❌ enc | ❌ enc | ❌ enc | ❌ enc | ❌ enc | ❌ enc | ❌ enc | ❌ enc | ❌ enc | ❌ enc | ❌ enc | ❌ type |
| ed25519 | ❌ enc | ❌ enc | ❌ enc | ❌ type | ❌ type | ❌ type | ❌ type | ❌ enc | ❌ type | ❌ type | ❌ type | ❌ type |

### Additional fixtures

| File | API_23 (23) | API_24 (24) | API_26 (26) | API_28 (28) | API_29 (29) | API_30 (30) | TV_30 (30) | API_33 (33) | API_34 (34) | TV_34 (34) | API_36 (36) | API_37 (37) |
| --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- |
| extra-brainpool256-3des | ❌ key | ❌ key | ❌ key | ❌ key | ❌ key | ❌ key | ❌ key | ❌ key | ❌ key | ❌ key | ❌ key | ❌ key |
| extra-dsa2048-3des | ❌ type | ❌ type | ❌ type | ❌ type | ❌ type | ❌ type | ❌ type | ❌ type | ❌ type | ❌ type | ❌ type | ❌ type |
| extra-ec256-aes128 | ❌ enc | ❌ enc | ❌ enc | ❌ enc | ❌ enc | ❌ enc | ❌ enc | ❌ enc | ✅ KS | ✅ KS | ✅ KS | ✅ KS |
| extra-ec256-mac-sha384 | ❌ enc | ❌ enc | ❌ enc | ❌ enc | ❌ enc | ❌ enc | ❌ enc | ❌ enc | ✅ KS | ✅ KS | ✅ KS | ✅ KS |
| extra-ec256-mac-sha512 | ❌ enc | ❌ enc | ❌ enc | ❌ enc | ❌ enc | ❌ enc | ❌ enc | ❌ enc | ✅ KS | ✅ KS | ✅ KS | ✅ KS |
| extra-ec256-pbes1-md5 | ❌ enc | ❌ enc | ❌ enc | ❌ enc | ❌ enc | ❌ enc | ❌ enc | ❌ enc | ❌ enc | ❌ enc | ❌ enc | ❌ enc |
| extra-ec256-utf8-password | ✅ KS | ✅ KS | ✅ KS | ✅ KS | ✅ KS | ✅ KS | ✅ KS | ✅ KS | ✅ KS | ✅ KS | ✅ KS | ✅ KS |
| extra-expired-client-3des | ❌ expired | ❌ expired | ❌ expired | ❌ expired | ❌ expired | ❌ expired | ❌ expired | ❌ expired | ❌ expired | ❌ expired | ❌ expired | ❌ expired |
| extra-mldsa65-3des | ❌ key | ❌ key | ❌ key | ❌ key | ❌ key | ❌ key | ❌ key | ❌ key | ❌ key | ❌ key | ❌ key | ❌ key |
| extra-no-cn-3des | ✅ KS | ✅ KS | ✅ KS | ✅ KS | ✅ KS | ✅ KS | ✅ KS | ✅ KS | ✅ KS | ✅ KS | ✅ KS | ✅ KS |
| extra-rsa8192-3des | ✅ KS | ✅ KS | ✅ KS | ✅ KS | ✅ KS | ✅ KS | ✅ KS | ✅ KS | ✅ KS | ✅ KS | ✅ KS | ✅ KS |
| extra-utf8-subject-3des | ✅ KS | ✅ KS | ✅ KS | ✅ KS | ✅ KS | ✅ KS | ✅ KS | ✅ KS | ✅ KS | ✅ KS | ✅ KS | ✅ KS |

## Summary per Android version

| Emulator | Readable .p12 encryption | Usable key types |
| --- | --- | --- |
| API_23 (23) | OpenSSL -legacy (3DES key, RC2-40 certs), 3DES / SHA-1 | rsa2048, rsa4096, rsa1024, ec256, ec384 |
| API_24 (24) | OpenSSL -legacy (3DES key, RC2-40 certs), 3DES / SHA-1 | rsa2048, rsa4096, rsa1024, ec256, ec384 |
| API_26 (26) | OpenSSL -legacy (3DES key, RC2-40 certs), 3DES / SHA-1 | rsa2048, rsa4096, rsa1024, ec256, ec384 |
| API_28 (28) | OpenSSL -legacy (3DES key, RC2-40 certs), 3DES / SHA-1 | rsa2048, rsa4096, rsa1024, ec256, ec384 |
| API_29 (29) | OpenSSL -legacy (3DES key, RC2-40 certs), 3DES / SHA-1 | rsa2048, rsa4096, rsa1024, ec256, ec384 |
| API_30 (30) | OpenSSL -legacy (3DES key, RC2-40 certs), 3DES / SHA-1 | rsa2048, rsa4096, rsa1024, ec256, ec384 |
| TV_30 (30) | OpenSSL -legacy (3DES key, RC2-40 certs), 3DES / SHA-1 | rsa2048, rsa4096, rsa1024, ec256, ec384 |
| API_33 (33) | OpenSSL -legacy (3DES key, RC2-40 certs), 3DES / SHA-1 | rsa2048, rsa4096, rsa1024, ec256, ec384 |
| API_34 (34) | OpenSSL 3 default (AES-256 / PBKDF2), OpenSSL -legacy (3DES key, RC2-40 certs), 3DES / SHA-1 | rsa2048, rsa4096, rsa1024, ec256, ec384 |
| TV_34 (34) | OpenSSL 3 default (AES-256 / PBKDF2), OpenSSL -legacy (3DES key, RC2-40 certs), 3DES / SHA-1 | rsa2048, rsa4096, rsa1024, ec256, ec384 |
| API_36 (36) | OpenSSL 3 default (AES-256 / PBKDF2), OpenSSL -legacy (3DES key, RC2-40 certs), 3DES / SHA-1 | rsa2048, rsa4096, rsa1024, ec256, ec384 |
| API_37 (37) | OpenSSL 3 default (AES-256 / PBKDF2), OpenSSL -legacy (3DES key, RC2-40 certs), 3DES / SHA-1, Unencrypted key | rsa2048, rsa4096, rsa1024, ec256, ec384 |

## mTLS handshakes (imported client key)

| Key (storage) / client | API_23 (23) | API_24 (24) | API_26 (26) | API_28 (28) | API_29 (29) | API_30 (30) | TV_30 (30) | API_33 (33) | API_34 (34) | TV_34 (34) | API_36 (36) | API_37 (37) |
| --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- |
| rsa2048 HttpsURLConnection | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| rsa2048 TLS_1_2 | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| rsa2048 TLS_1_3 | – | – | – | – | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| rsa4096 HttpsURLConnection | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| rsa4096 TLS_1_2 | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| rsa4096 TLS_1_3 | – | – | – | – | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| ec256 HttpsURLConnection | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| ec256 TLS_1_2 | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| ec256 TLS_1_3 | – | – | – | – | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| ec384 HttpsURLConnection | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| ec384 TLS_1_2 | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| ec384 TLS_1_3 | – | – | – | – | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| pqc client-ec.p12 | – | – | – | – | – | – | – | – | – | – | – | ✅ |
| pqc client-rsa.p12 | – | – | – | – | – | – | – | – | – | – | – | ✅ |

## TLS capabilities offered by the app

Measured from the ClientHello. `✅` offered by OkHttp (API, WebSocket, images, TV ExoPlayer) and HttpsURLConnection (mobile ExoPlayer, artwork, crash reports) · `OkHttp` / `URLConn` only by that client · `–` not offered

### TLS versions

| versions | API_23 (23) | API_24 (24) | API_26 (26) | API_28 (28) | API_29 (29) | API_30 (30) | TV_30 (30) | API_33 (33) | API_34 (34) | TV_34 (34) | API_36 (36) | API_37 (37) |
| --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- |
| TLS 1.2 | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| TLS 1.3 | – | – | – | – | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| TLS 1.1 | – | – | – | – | URLConn | URLConn | URLConn | URLConn | URLConn | URLConn | – | – |
| TLS 1.0 | – | – | – | – | URLConn | URLConn | URLConn | URLConn | URLConn | URLConn | – | – |

### Key exchange groups (incl. post-quantum)

| groups | API_23 (23) | API_24 (24) | API_26 (26) | API_28 (28) | API_29 (29) | API_30 (30) | TV_30 (30) | API_33 (33) | API_34 (34) | TV_34 (34) | API_36 (36) | API_37 (37) |
| --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- |
| secp256r1 | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| secp384r1 | ✅ | – | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| secp521r1 | ✅ | – | – | – | – | – | – | – | – | – | – | – |
| x25519 | – | – | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| X25519MLKEM768 | – | – | – | – | – | – | – | – | – | – | – | ✅ |

### Key shares (groups the first ClientHello is ready for)

| keyshares | API_23 (23) | API_24 (24) | API_26 (26) | API_28 (28) | API_29 (29) | API_30 (30) | TV_30 (30) | API_33 (33) | API_34 (34) | TV_34 (34) | API_36 (36) | API_37 (37) |
| --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- |
| x25519 | – | – | – | – | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| X25519MLKEM768 | – | – | – | – | – | – | – | – | – | – | – | ✅ |

### Cipher suites

| ciphers | API_23 (23) | API_24 (24) | API_26 (26) | API_28 (28) | API_29 (29) | API_30 (30) | TV_30 (30) | API_33 (33) | API_34 (34) | TV_34 (34) | API_36 (36) | API_37 (37) |
| --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- |
| ECDHE_ECDSA_AES_128_GCM | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| ECDHE_RSA_AES_128_GCM | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| ECDHE_ECDSA_AES_256_GCM | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| ECDHE_RSA_AES_256_GCM | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| ECDHE_RSA_AES_128_CBC | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| ECDHE_RSA_AES_256_CBC | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| RSA_AES_128_GCM | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| RSA_AES_256_GCM | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| RSA_AES_128_CBC | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| RSA_AES_256_CBC | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| EMPTY_RENEGOTIATION_INFO | ✅ | – | – | – | – | – | – | – | – | – | – | – |
| 0x009e | URLConn | URLConn | – | – | – | – | – | – | – | – | – | – |
| 0x009f | URLConn | URLConn | – | – | – | – | – | – | – | – | – | – |
| ECDHE_ECDSA_AES_128_CBC | URLConn | URLConn | URLConn | URLConn | URLConn | URLConn | URLConn | URLConn | URLConn | URLConn | URLConn | URLConn |
| ECDHE_ECDSA_AES_256_CBC | URLConn | URLConn | URLConn | URLConn | URLConn | URLConn | URLConn | URLConn | URLConn | URLConn | URLConn | URLConn |
| 0x0033 | URLConn | URLConn | – | – | – | – | – | – | – | – | – | – |
| 0x0039 | URLConn | URLConn | – | – | – | – | – | – | – | – | – | – |
| 0xc007 | URLConn | – | – | – | – | – | – | – | – | – | – | – |
| 0xc011 | URLConn | – | – | – | – | – | – | – | – | – | – | – |
| 0x0005 | URLConn | – | – | – | – | – | – | – | – | – | – | – |
| ECDHE_ECDSA_CHACHA20_POLY1305 | – | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| ECDHE_RSA_CHACHA20_POLY1305 | – | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| TLS_AES_128_GCM_SHA256 | – | – | – | – | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| TLS_AES_256_GCM_SHA384 | – | – | – | – | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| TLS_CHACHA20_POLY1305_SHA256 | – | – | – | – | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |

### Signature algorithms (server and client certificates)

| sigalgs | API_23 (23) | API_24 (24) | API_26 (26) | API_28 (28) | API_29 (29) | API_30 (30) | TV_30 (30) | API_33 (33) | API_34 (34) | TV_34 (34) | API_36 (36) | API_37 (37) |
| --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- |
| rsa_pkcs1_sha512 | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| ecdsa_secp521r1_sha512 | ✅ | ✅ | ✅ | – | – | – | – | – | – | – | – | – |
| rsa_pkcs1_sha384 | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| ecdsa_secp384r1_sha384 | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| rsa_pkcs1_sha256 | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| ecdsa_secp256r1_sha256 | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| 0x0301 | ✅ | ✅ | – | – | – | – | – | – | – | – | – | – |
| 0x0303 | ✅ | ✅ | – | – | – | – | – | – | – | – | – | – |
| rsa_pkcs1_sha1 | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| ecdsa_sha1 | ✅ | ✅ | – | – | – | – | – | – | – | – | – | – |
| rsa_pss_rsae_sha256 | – | – | – | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| rsa_pss_rsae_sha384 | – | – | – | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| rsa_pss_rsae_sha512 | – | – | – | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |

## WebView per emulator

| AVD | API | WebView in use | Login test |
| --- | --- | --- | --- |
| TLS_API_23 | 23 | com.android.webview 88.0.4324.152 | – |
| TLS_API_24 | 24 | com.android.chrome 119.0.6045.194 | – |
| TLS_API_26 | 26 | com.android.chrome 126.0.6478.71 | – |
| TLS_API_28 | 28 | com.android.chrome 126.0.6478.71 | – |
| TLS_API_29 | 29 | com.google.android.webview 146.0.7680.164 | – |
| TLS_API_30 | 30 | com.android.webview 83.0.4103.120 | – |
| TLS_API_33 | 33 | com.google.android.webview 109.0.5414.123 | – |
| TLS_API_34 | 34 | com.google.android.webview 113.0.5672.136 | – |
| TLS_API_36 | 36 | com.google.android.webview 133.0.6943.137 | – |
| TLS_TV_30 | 30 | com.google.android.webview 90.0.4430.91 | – |
| TLS_TV_34 | 34 | com.google.android.webview 113.0.5672.136 | – |
