# Shadowsocks 2022 Edition: Secure L4 Tunnel with Symmetric Encryption

## Abstract

This document defines the 2022 Edition of the Shadowsocks protocol. Improving upon Shadowsocks AEAD (2017), Shadowsocks 2022 addresses well-known issues of the previous editions, drops usage of obsolete cryptography, optimizes for security and performance, and leaves room for future extensions.

## 1. Overview

Shadowsocks 2022 is a secure proxy protocol for TCP and UDP traffic. The protocol uses [AEAD](https://en.wikipedia.org/wiki/Authenticated_encryption) with a pre-shared symmetric key to protect payload integrity and confidentiality. The proxy traffic is indistinguishable from a random byte stream, and therefore can circumvent firewalls and Internet censors that rely on [DPI (Deep Packet Inspection)](https://en.wikipedia.org/wiki/Deep_packet_inspection).

Compared to [previous editions](https://github.com/shadowsocks/shadowsocks-org/blob/master/whitepaper/whitepaper.md) of the protocol family, Shadowsocks 2022 allows and mandates full replay protection. Each message has its unique type and cannot be used for unintended purposes. The session-based UDP proxying significantly reduces protocol overhead and improves reliability and efficiency. Obsolete cryptographic functions have been replaced by their modern counterparts.

As with previous editions, Shadowsocks 2022 does not provide forward secrecy. It is believed that using a pre-shared key without performing handshakes is best for its use cases.

A Shadowsocks 2022 implementation consists of a server, a client, and optionally a relay. This document specifies requirements that implementations must follow.

### 1.1. Document Structure

This document describes the Shadowsocks 2022 Edition and is structured as follows:

- Section 2 describes requirements on the encryption key and how to derive session subkeys.
- Section 3 defines the encoding details of the required AES-GCM methods and the process for handling requests and responses.
- Section 4 defines the encoding details of the optional ChaCha-Poly1305 methods.

### 1.2. Terms and Definitions

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED", "NOT RECOMMENDED", "MAY", and "OPTIONAL" in this document are to be interpreted as described in BCP 14 [RFC2119](https://www.rfc-editor.org/info/rfc2119) [RFC8174](https://www.rfc-editor.org/info/rfc8174) when, and only when, they appear in all capitals, as shown here.

Commonly used terms in this document are described below.

- Shadowsocks AEAD: The original AEAD construction of Shadowsocks, standardized in 2017.

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

TCP connections over a Shadowsocks 2022 tunnel maps 1:1 to proxy connections. Each proxy connection carries 2 proxy streams: request stream and response stream. A client initiates a proxy connection by starting a request stream, and the server sends back response over the response stream. These streams carry chunks of data encrypted by the session subkey.

For payload transfer, Shadowsocks 2022 inherits the length-chunk-payload-chunk model from Shadowsocks AEAD, with some minor tweaks to improve performance. Standalone header chunks are added to both request and response streams to improve security and protect against replay attacks.

#### 3.1.1. Encryption and Decryption

Each proxy stream derives its own session subkey with a random salt for encryption and decryption. A 12-byte little-endian integer is used as nonce, and is incremented after each encryption or decryption operation.

```
u96le counter
aead := aead_new(key: session_subkey)
ciphertext := aead.seal(nonce: counter, plaintext)
plaintext := aead.open(nonce: counter, ciphertext)
```

#### 3.1.2. Format

A request stream starts with one random salt and two standalone header chunks, followed repeatedly by one length chunk and one payload chunk. Each chunk is independently encrypted/decrypted using the AEAD cipher.

A response stream also starts with a random salt, but only has one fixed-length header chunk, which also acts as the first length chunk.

A length chunk is a 16-bit big-endian unsigned integer that describes the payload length in the next payload chunk. Servers and clients rely on length chunks to know how many bytes to read for the next payload chunk.

A payload chunk can have up to 0xFFFF (65535) bytes of unencrypted payload. The 0x3FFF (16383) length cap in Shadowsocks AEAD does not apply to this edition.

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

Request stream:
+--------+------------------------+---------------------------+------------------------+---------------------------+---+
|  salt  | encrypted header chunk |  encrypted header chunk   | encrypted length chunk |  encrypted payload chunk  |...|
+--------+------------------------+---------------------------+------------------------+---------------------------+---+
| 16/32B |     11B + 16B tag      | variable length + 16B tag |  2B length + 16B tag   | variable length + 16B tag |...|
+--------+------------------------+---------------------------+------------------------+---------------------------+---+

Response stream:
+--------+------------------------+---------------------------+------------------------+---------------------------+---+
|  salt  | encrypted header chunk |  encrypted payload chunk  | encrypted length chunk |  encrypted payload chunk  |...|
+--------+------------------------+---------------------------+------------------------+---------------------------+---+
| 16/32B |    27/43B + 16B tag    | variable length + 16B tag |  2B length + 16B tag   | variable length + 16B tag |...|
+--------+------------------------+---------------------------+------------------------+---------------------------+---+
```

#### 3.1.3. Header

```
Request fixed-length header:
+------+---------------+--------+
| type |   timestamp   | length |
+------+---------------+--------+
|  1B  | 8B unix epoch |  u16be |
+------+---------------+--------+

Request variable-length header:
+------+----------+-------+----------------+----------+-----------------+
| ATYP |  address |  port | padding length |  padding | initial payload |
+------+----------+-------+----------------+----------+-----------------+
|  1B  | variable | u16be |     u16be      | variable |    variable     |
+------+----------+-------+----------------+----------+-----------------+

Response fixed-length header:
+------+---------------+----------------+--------+
| type |   timestamp   |  request salt  | length |
+------+---------------+----------------+--------+
|  1B  | 8B unix epoch |     16/32B     |  u16be |
+------+---------------+----------------+--------+

HeaderTypeClientStream = 0
HeaderTypeServerStream = 1
MinPaddingLength = 0
MaxPaddingLength = 900
```

- 1-byte type: Differentiates between client and server messages. A request stream has type `0`. A response stream has type `1`.
- 8-byte Unix epoch timestamp: Messages with over 30 seconds of time difference MUST be treated as replay.
- Length: Indicates the next chunk's plaintext length (not including the authentication tag).
- ATYP + address + port: Target address in [SOCKS5 address format](https://datatracker.ietf.org/doc/html/rfc1928#section-5).
- Request salt in response header: This maps a response stream to a request stream. The client MUST check this field in response header against the request salt.

#### 3.1.3. Detection Prevention

The random salt and header chunks MUST be buffered and sent in one write call to the underlying socket. Separate writes can result in predictable packet sizes, which could reveal the protocol in use.

To process the salt and the fixed-length header, servers and clients MUST make exactly one read call. If the amount of data received is not enough for decryption, or decryption fails, or header validation fails, the server MUST act in a way that does not exhibit the amount of bytes consumed by the server. This defends against probes that send one byte at a time to detect how many bytes the server consumes before closing the connection.

In such circumstances, do not immediately close the socket. Closing the socket with unread data causes RST to be sent. This reveals the exact number of bytes consumed by the server. Implementations MAY choose to employ one of the following strategies:

1. To consistently send RST even when the receive buffer is empty, set `SO_LINGER` to true with a zero timeout, then close the socket.
2. To consistently send FIN even when the receive buffer has unread data, shut down the write half of the connection by calling `shutdown(SHUT_WR)`, then drain the connection for any further received data.
3. To consistently send FIN even when the receive buffer has unread data, but disallow unlimited writes, shut down the write half of the connection by calling `shutdown(SHUT_WR)`, then `epoll` for `EPOLLRDHUP`. Read until EOF then close the connection. This limits the amount of data the other party can send to the size of your socket receive buffer.

In a request header, either initial payload or padding MUST be present. When making a request header, if payload is not available, add non-zero random length padding.

For client implementations, a simple approach is to always send random length padding. To accommodate TCP Fast Open (TFO), clients MAY wait a short amount of time (typically less than one second) for client-first protocols to write the first payload, before carrying on to establish a proxy connection and write the header.

Servers MUST reject the request if the variable-length header chunk does not contain payload and the padding length is 0.
Servers MUST enforce that the request header (including padding) does not extend beyond the header chunks.

For response streams, the header is always sent along with payload. No padding is needed.

#### 3.1.4. Replay Protection

Servers and clients MUST store all incoming salts for 60 seconds. When a new TCP session is established, the first received message is decrypted and its timestamp MUST be checked against system time. If the time difference is within 30 seconds, then the salt is checked against all stored salts. If no repeated salt is discovered, then the salt is added to the pool and the session is successfully established.

For salt storage, implementations MUST NOT use Bloom filters or anything that could return a false positive result.

### 3.2. UDP

Shadowsocks 2022 completely overhauled UDP relay. Each UDP relay session has a unique session ID, which is also used as salt to derive the session subkey. A packet ID acts as packet counter for the session. The session ID and packet ID are combined and encrypted in a separate header.

Clients create UDP relay sessions based on source address and port. When a client receives a packet from a new source address and port, it opens a new relay session, and subsequent packets from that source are sent over the same session.

Servers manage UDP relay sessions by session ID. Each client session corresponds to one outgoing UDP socket on the server.

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
Packet:
+---------------------------+---------------------------+
| encrypted separate header |       encrypted body      |
+---------------------------+---------------------------+
|            16B            | variable length + 16B tag |
+---------------------------+---------------------------+

Separate header:
+------------+-----------+
| session ID | packet ID |
+------------+-----------+
|     8B     |   u64be   |
+------------+-----------+
```

UDP sessions are initiated by clients. To start a UDP session, the client generates a new random session ID and maintains a counter starting at zero as packet ID. These are used in client-to-server messages and are usually referred to as client session ID and client packet ID.

Servers use client session IDs to identify UDP sessions. For server-to-client messages, a different set of session ID and packet ID is used, and may be referred to as server session ID and server packet ID. Servers MUST not use client session IDs as server session IDs.

#### 3.2.3. Main Header

The main header, or message header, is the header at the start of the body. The client-to-server message header consists of type, timestamp, padding and SOCKS address. The server-to-client message header has an additional client session ID field, which maps the server session to a client session.

```
Client-to-server message header:
+------+---------------+----------------+----------+------+----------+-------+
| type |   timestamp   | padding length |  padding | ATYP |  address |  port |
+------+---------------+----------------+----------+------+----------+-------+
|  1B  | 8B unix epoch |     u16be      | variable |  1B  | variable | u16be |
+------+---------------+----------------+----------+------+----------+-------+

Server-to-client message header:
+------+---------------+-------------------+----------------+----------+------+----------+-------+
| type |   timestamp   | client session ID | padding length |  padding | ATYP |  address |  port |
+------+---------------+-------------------+----------------+----------+------+----------+-------+
|  1B  | 8B unix epoch |         8B        |     u16be      | variable |  1B  | variable | u16be |
+------+---------------+-------------------+----------------+----------+------+----------+-------+

HeaderTypeClientPacket = 0
HeaderTypeServerPacket = 1
```

- 1-byte type: Differentiates between client and server messages. A client message has type `0`. A server message has type `1`.
- 8-byte Unix epoch timestamp: Messages with over 30 seconds of time difference MUST be treated as replay.
- Padding length: Specifies the length of the optional padding. Implementations MAY allow users to select from a list of predefined padding policies. Care SHOULD be taken to not exceed the network path's MTU when padding packets.

#### 3.2.4. Session ID based Routing and Sliding Window Replay Protection

Servers MUST route packets based on client session ID, not packet source address. When a server receives a packet with a new client session ID, a new relay session is created, and subsequent packets from that client session are sent over this relay session.

A relay session MUST keep track of the last seen client address. When a packet is received from the client and is successfully validated, the last seen client address MUST be updated. Return packets MUST be sent to this address. This allows UDP sessions to survive client network changes.

Each relay session MUST be remembered for at least 60 seconds. A shorter NAT timeout may allow attackers to successfully replay packets from an already forgotten client session.

To handle server restarts, clients MUST allow each client session to be associated with more than one server session. Each association MUST be remembered for no less than the NAT timeout, which is at least 60 seconds. Alternatively, clients MAY choose to keep track of one old server session and one current server session, and reject newer server sessions when the last packet received from the old session is less than 1 minute old.

Clients and servers MUST employ a sliding window filter for each relay session to check incoming packets for duplicate or out-of-window packet IDs. Existing implementations from WireGuard MAY be used. The check MUST be performed after header validation, which filters out packets that are semantically invalid or have a bad timestamp.

## 4. Optional Methods

Implementations MAY choose to implement `2022-blake3-chacha20-poly1305`, `2022-blake3-chacha12-poly1305` and `2022-blake3-chacha8-poly1305` when support for CPUs without AES instructions is a priority. The use of reduced-round ChaCha20 variants is justified by [this paper](https://eprint.iacr.org/2019/1492.pdf).

For TCP, AES-GCM is simply replaced by ChaCha-Poly1305. For UDP, a slightly different construction is used.

### 4.1. UDP Construction

`2022-blake3-chacha20-poly1305` uses XChaCha20-Poly1305 with the pre-shared key directly and a random nonce for each message.

A UDP packet starts with the random nonce, followed by an encrypted body. The session ID and packet ID are merged into the main header.

The same sliding window filter is used for replay protection. It is not necessary to check for repeated nonce.

```
Packet:
+-------+---------------------------+
| nonce |       encrypted body      |
+-------+---------------------------+
|  24B  | variable length + 16B tag |
+-------+---------------------------+

Client-to-server message header:
+-------------------+------------------+------+---------------+----------------+----------+------+----------+-------+
| client session ID | client packet ID | type |   timestamp   | padding length |  padding | ATYP |  address |  port |
+-------------------+------------------+------+---------------+----------------+----------+------+----------+-------+
|         8B        |       u64be      |  1B  | 8B unix epoch |     u16be      | variable |  1B  | variable | u16be |
+-------------------+------------------+------+---------------+----------------+----------+------+----------+-------+

Server-to-client message header:
+-------------------+------------------+------+---------------+-------------------+----------------+----------+------+----------+-------+
| server session ID | server packet ID | type |   timestamp   | client session ID | padding length |  padding | ATYP |  address |  port |
+-------------------+------------------+------+---------------+-------------------+----------------+----------+------+----------+-------+
|         8B        |       u64be      |  1B  | 8B unix epoch |         8B        |     u16be      | variable |  1B  | variable | u16be |
+-------------------+------------------+------+---------------+-------------------+----------------+----------+------+----------+-------+
```

## Acknowledgement

I would like to thank @zonyitoo, @xiaokangwang, and @nekohasekai for their input on the design of the protocol.
