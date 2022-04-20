# Shadowsocks 2022 Edition: Secure L4 Tunnel with Symmetric Encryption

## Abstract

This document defines the 2022 Edition of the Shadowsocks protocol. Improving upon Shadowsocks AEAD (2017), Shadowsocks 2022 addresses well-known issues of the previous editions, drops the use of obsolete cryptography, optimizes for security and performance, and leaves room for future extensions.

## 1. Overview

Shadowsocks 2022 is the latest edition of the Shadowsocks protocol family. The protocol provides a secure TCP & UDP tunnel, encrypted with a pre-shared symmetric key. Shadowsocks 2022's traffic looks indistinguishable from a random byte stream, and therefore can circumvent firewalls and Internet censors that rely on [DPI (Deep Packet Inspection)](https://en.wikipedia.org/wiki/Deep_packet_inspection).

Compared to previous editions of the protocol family, Shadowsocks 2022 allows and mandates full replay protection. Each message has its unique type and cannot be used for unintended purposes. The session-based UDP proxying significantly reduces protocol overhead and improves reliability and efficiency. Obsolete cryptographic functions have been replaced by modern counterparts.

As with previous editions, Shadowsocks 2022 does not provide forward secrecy. It is believed that using a pre-shared key without performing handshakes is best for its use cases.

A Shadowsocks 2022 implementation consists of a server, a client, and optionally a relay. This document specifies requirements that implementations must follow.

### 1.1. Document Structure

This document describes the Shadowsocks 2022 Edition and is structured as follows:

- Section 2 describes requirements on the encryption key and how to derive session subkeys.
- Section 3 defines the encoding details of the required AES-GCM methods and the process for handling requests and responses.
- Section 4 defines the encoding details of the optional ChaCha-Poly1305 methods.
- Section 5 showcases some benchmark and speed test results.

### 1.2. Terms and Definitions

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED", "NOT RECOMMENDED", "MAY", and "OPTIONAL" in this document are to be interpreted as described in BCP 14 [RFC2119](https://www.rfc-editor.org/info/rfc2119) [RFC8174](https://www.rfc-editor.org/info/rfc8174) when, and only when, they appear in all capitals, as shown here.

Commonly used terms in this document are described below.

- Shadowsocks AEAD: The original AEAD construction of Shadowsocks, standardized in 2017.
- OOCv1: An acronym for the Open Online Config protocol version 1 defined by [Open Online Config Version 1](https://github.com/Shadowsocks-NET/OpenOnlineConfig/blob/master/docs/0001-open-online-config-v1.md).

## 2. Encryption/Decryption Keys

A pre-shared key is used to derive session subkeys, which are subsequently used to encrypt/decrypt traffic for the session. The pre-shared key is also used directly in some places.

### 2.1. PSK

Unlike previous editions, Shadowsocks 2022 requires that a cryptographically-secure fixed-length PSK to be directly provided by the user. Implementations MUST NOT use the old `EVP_BytesToKey` function or any other method to generate keys from passwords.

The PSK is encoded in base64 for convenience. Practically, it can be generated with `openssl rand -base64 <key_size>`. The key size depends on the chosen method. This change was inspired by WireGuard.

| Method                  | Key Bytes | Salt Bytes |
| ----------------------- | --------: | ---------: |
| 2022-blake3-aes-128-gcm |        16 |         16 |
| 2022-blake3-aes-256-gcm |        32 |         32 |

### 2.2. Subkey Derivation

Shadowsocks 2022's subkey derivation uses [BLAKE3](https://raw.githubusercontent.com/BLAKE3-team/BLAKE3-specs/master/blake3.pdf)'s key derivation mode, which replaces the obsolete HKDF_SHA1 function in previous editions. A randomly generated salt is appended to the PSK to be used as key material. The salt has the same length as the pre-shared key.

```
session_subkey := blake3::derive_key(context: "shadowsocks 2022 session subkey", key_material: key + salt)
```

## 3. Required Methods

Method `2022-blake3-aes-128-gcm` and `2022-blake3-aes-256-gcm` MUST be implemented by all implementations. `2022` reflects the fast-changing and flexible nature of the protocol.

### 3.1. TCP

Shadowsocks 2022's TCP wire format is the same as Shadowsocks AEAD, with some minor tweaks to improve performance. A header is included in the first payload chunk in both request and response. The header improves security and protects against replay attacks.

#### 3.1.1. Encryption and Decryption

The AEAD cipher used in encryption or decryption is constructed with the session subkey. A 12-byte little-endian integer is used as nonce.

```
u96le counter
aead := aead_new(key: session_subkey)
ciphertext := aead.seal(nonce: counter, plaintext)
plaintext := aead.open(nonce: counter, ciphertext)
```

#### 3.1.2. Format

The request stream starts with a random salt, followed repeatedly by one length chunk and one payload chunk. Each chunk is independently encrypted/decrypted using the AEAD cipher. After each encryption/decryption operation, the nonce MUST be incremented by 1.

The random salt, the first length chunk and the first payload chunk MUST be buffered and sent in one write call to the underlying socket. Separate writes can lead to predictable packet sizes, which could potentially be used to detect the protocol.

The length chunk contains a 16-bit big-endian unsigned integer that describes the payload length in the next payload chunk. Servers and clients use this information to read the next payload chunk. The payload length is up to 0xFFFF (65535) bytes. The 0x3FFF length cap in Shadowsocks AEAD has been dropped in this edition.

```
+----------------+
|  length chunk  |
+----------------+
| u16 big-endian |
+----------------+

+---------------+
| payload chunk |
+---------------+
|   variable    |
+---------------+

+--------+------------------------+---------------------------+---+
|  salt  | encrypted length chunk |  encrypted payload chunk  |...|
+--------+------------------------+---------------------------+---+
| 16/32B |  2B length + 16B tag   | variable length + 16B tag |...|
+--------+------------------------+---------------------------+---+
```

#### 3.1.3. Header

The first payload chunk of a request stream contains header and payload. If payload is not available, add non-zero random length padding.

For client implementations, a simple approach is to always send random length padding. To accommodate TCP Fast Open (TFO), clients MAY wait a short amount of time (typically less than one second) for client-first protocols to write the first payload, before carrying on to establish a proxy connection and write the header.

Servers MUST reject the request if the first payload chunk does not contain payload and the padding length is 0.
Servers MUST enforce that the request header (including padding) does not extend beyond the first payload chunk.

For response streams, the header is always sent along with payload. No padding is needed.

```
+------+---------------+------+----------+-------+----------------+----------+
| type |   timestamp   | ATYP |  address |  port | padding length |  padding |
+------+---------------+------+----------+-------+----------------+----------+
|  1B  | 8B unix epoch |  1B  | variable | u16be |     u16be      | variable |
+------+---------------+------+----------+-------+----------------+----------+

+------+---------------+----------------+
| type |   timestamp   |  request salt  |
+------+---------------+----------------+
|  1B  | 8B unix epoch |     16/32B     |
+------+---------------+----------------+

HeaderTypeClientStream = 0
HeaderTypeServerStream = 1
MinPaddingLength = 0
MaxPaddingLength = 900
```

- 1-byte type: Differentiates between client and server messages. A request stream has type `0`. A response stream has type `1`.
- 8-byte Unix epoch timestamp: Messages with over 30 seconds of time difference MUST be treated as replay.
- ATYP + address + port: Target address in [SOCKS5 address format](https://datatracker.ietf.org/doc/html/rfc1928#section-5).
- Request salt in response header: This maps a response stream to a request stream. The client MUST check this field in response header against the request salt.

#### 3.1.4. Replay Protection

Servers and clients MUST store all incoming salts for 60 seconds. When a new TCP session is established, the first received message is decrypted and its timestamp MUST be check against system time. If the time difference is within 30 seconds, then the salt is checked against all stored salts. If no repeated salt is discovered, then the salt is added to the pool and the session is successfully established.

For salt storage, implementations MUST NOT use Bloom filters or anything that could return a false positive result.

### 3.2. UDP

Shadowsocks 2022 completely overhauled UDP relay. Each UDP relay session has a unique session ID, which is also used as salt to derive the session subkey. A packet ID acts as packet counter for the session. The session ID and packet ID are combined and encrypted in a separate header.

#### 3.2.1. Encryption and Decryption

The separate header is encrypted/decrypted with the pre-shared key using an AES block cipher. The body is encrypted/decrypted with the session subkey using an AEAD cipher specific to the session.

```
block_cipher := aes_new(psk)
encrypted_separate_header := block_cipher.encrypt(separate_header)
decrypted_separate_header := block_cipher.decrypt(encrypted_separate_header)

session_subkey := blake3::derive_key(context: "shadowsocks 2022 session subkey", key_material: key + separate_header[0..8])
session_aead_cipher := aes_gcm_new(session_subkey)
encrypted_body := session_aead_cipher.seal(nonce: separate_header[4..16], body)
decrypted_body := session_aead_cipher.open(nonce: separate_header[4..16], body)
```

#### 3.2.2. Format and Separate Header

A UDP packet consists of a separate header and an AEAD-encrypted body. The separate header consists of an 8-byte session ID and an 8-byte big-endian unsigned integer as packet ID. The body is made up of the main header and payload.

```
+---------------------------+---------------------------+
| encrypted separate header |       encrypted body      |
+---------------------------+---------------------------+
|            16B            | variable length + 16B tag |
+---------------------------+---------------------------+

+------------+-----------+
| session ID | packet ID |
+------------+-----------+
|     8B     |   u64be   |
+------------+-----------+
```

UDP sessions are initiated by clients. To start a UDP session, the client generates a new random session ID and maintains a counter starting at zero as packet ID. These are used in client-to-server messages and are usually referred to as client session ID and client packet ID.

Servers use client session IDs to identify UDP sessions. For server-to-client messages, a different set of session ID and packet ID is used, and may be referred to as server session ID and server packet ID. Servers MUST not use client session IDs as server session IDs.

#### 3.2.3. Main Header

The main header, or message header, is the header at the start of the body. The client-to-server message header consists of type, timestamp, padding and SOCKS address. The server-to-client message header has an additional client session ID field.

```
+------+---------------+----------------+----------+------+----------+-------+
| type |   timestamp   | padding length |  padding | ATYP |  address |  port |
+------+---------------+----------------+----------+------+----------+-------+
|  1B  | 8B unix epoch |     u16be      | variable |  1B  | variable | u16be |
+------+---------------+----------------+----------+------+----------+-------+

+------+---------------+-------------------+----------------+----------+------+----------+-------+
| type |   timestamp   | client session ID | padding length |  padding | ATYP |  address |  port |
+------+---------------+-------------------+----------------+----------+------+----------+-------+
|  1B  | 8B unix epoch |         8B        |     u16be      | variable |  1B  | variable | u16be |
+------+---------------+-------------------+----------------+----------+------+----------+-------+

HeaderTypeClientPacket = 0
HeaderTypeServerPacket = 1
```

#### 3.2.4. Session ID based Routing and Sliding Window Replay Protection

Servers SHOULD use client session IDs as keys in the NAT table.

A server NAT entry SHOULD cache the session AEAD ciphers for encrypting and decrypting packets.

A server NAT entry SHOULD keep track of the last seen client address. When a packet is received from the client and is successfully validated, the last seen client address SHOULD be updated. Return packets SHOULD be sent to this address. This allows UDP sessions to survive client network changes.

A server NAT entry MUST implement a sliding window filter that checks the packet ID for duplicate or out-of-window packets.

Clients SHOULD use a map with a timeout no less than the NAT timeout to keep track of server sessions. Alternatively, clients MAY choose to keep track of one old server session and one current server session, and reject new server sessions, when the last packet received from the old session is less than 1 minute old.

## 4. Optional Methods

Implementations MAY choose to implement `2022-blake3-chacha20-poly1305`, `2022-blake3-chacha12-poly1305` and `2022-blake3-chacha8-poly1305` when support for CPUs without AES instructions is a priority. The use of reduced-round ChaCha20 variants is justified by [this paper](https://eprint.iacr.org/2019/1492.pdf).

For TCP, AES-GCM is simply replaced by ChaCha-Poly1305. For UDP, a slightly different construction is used.

### 4.1. UDP Construction

`2022-blake3-chacha20-poly1305` uses XChaCha20-Poly1305 with the pre-shared key directly and a random nonce for each message.

A UDP packet starts with the random nonce, followed by an encrypted body. The session ID and packet ID are merged into the main header.

The same sliding window filter is used for replay protection. It is not necessary to check for repeated nonce.

```
+-------+---------------------------+
| nonce |       encrypted body      |
+-------+---------------------------+
|  24B  | variable length + 16B tag |
+-------+---------------------------+

+-------------------+------------------+------+---------------+----------------+----------+------+----------+-------+
| client session ID | client packet ID | type |   timestamp   | padding length |  padding | ATYP |  address |  port |
+-------------------+------------------+------+---------------+----------------+----------+------+----------+-------+
|         8B        |       u64be      |  1B  | 8B unix epoch |     u16be      | variable |  1B  | variable | u16be |
+-------------------+------------------+------+---------------+----------------+----------+------+----------+-------+

+-------------------+------------------+------+---------------+-------------------+----------------+----------+------+----------+-------+
| server session ID | server packet ID | type |   timestamp   | client session ID | padding length |  padding | ATYP |  address |  port |
+-------------------+------------------+------+---------------+-------------------+----------------+----------+------+----------+-------+
|         8B        |       u64be      |  1B  | 8B unix epoch |         8B        |     u16be      | variable |  1B  | variable | u16be |
+-------------------+------------------+------+---------------+-------------------+----------------+----------+------+----------+-------+
```

## 5. Benchmarks

### 5.1. Header benchmarks

https://github.com/database64128/cubic-go-playground/blob/main/shadowsocks/udpheader/udpheader_test.go

```
BenchmarkGenSaltHkdfSha1-8          	  372712	      3283 ns/op	    1264 B/op	      18 allocs/op
BenchmarkGenSaltBlake3-8            	  770426	      1554 ns/op	       0 B/op	       0 allocs/op
BenchmarkAesEcbHeaderEncryption-8   	99224395	        12.84 ns/op	       0 B/op	       0 allocs/op
BenchmarkAesEcbHeaderDecryption-8   	100000000	        11.79 ns/op	       0 B/op	       0 allocs/op
```

### 5.2. Full UDP packet construction benchmarks

https://github.com/database64128/cubic-go-playground/blob/main/shadowsocks/udp_test.go

```
BenchmarkShadowsocksAEADAes256GcmEncryption-8             	  252819	      4520 ns/op	    2160 B/op	      23 allocs/op
BenchmarkShadowsocksAEADAes256GcmWithBlake3Encryption-8   	  320718	      3367 ns/op	     896 B/op	       5 allocs/op
BenchmarkDraftSeparateHeaderAes256GcmEncryption-8         	 2059383	       590.5 ns/op	       0 B/op	       0 allocs/op
BenchmarkDraftXChaCha20Poly1305Encryption-8               	  993336	      1266 ns/op	       0 B/op	       0 allocs/op
```

### 5.3. shadowsocks-rust Speed Tests

|       shadowsocks-rust        |   TCP    |   UDP    |
| ----------------------------- | -------: | -------: |
| 2022-blake3-aes-128-gcm       | 12.2Gbps | 14.2Gbps |
| 2022-blake3-aes-256-gcm       | 10.9Gbps | 12.5Gbps |
| 2022-blake3-chacha20-poly1305 | 8.05Gbps | 2.35Gbps |
| 2022-blake3-chacha8-poly1305  | 8.36Gbps | 2.60Gbps |
| aes-128-gcm                   | 8.99Gbps | 13.5Gbps |
| aes-256-gcm                   | 8.21Gbps | 11.9Gbps |
| chacha20-poly1305             | 6.55Gbps | 8.66Gbps |

## Acknowledgement

I would like to thank @zonyitoo, @xiaokangwang, and @nekohasekai for their input on the design of the protocol.
