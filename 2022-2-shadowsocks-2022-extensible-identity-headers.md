# Shadowsocks 2022 Extensible Identity Headers

Identity headers are one or more additional layers of headers, each consisting of the next layer's PSK hash. The next layer of an identity header is the next identity header, or the protocol header if it's the last identity header. Identity headers are encrypted with the current layer's identity PSK using an AES block cipher.

Identity headers are implemented in such a way that's fully backward compatible with current Shadowsocks 2022 implementations. Each identity processor is fully transparent to the next.

- iPSKn: The nth identity PSK that identifies the current layer.
- uPSKn: The nth user PSK that identifies a user on the server.

## TCP

In TCP requests, identity headers are located between salt and AEAD chunks.

```
identity_subkey := blake3::derive_key(context: "shadowsocks 2022 identity subkey", key_material: iPSKn + salt)
plaintext := blake3::hash(iPSKn+1)[0..16] // Take the first 16 bytes of the next iPSK's hash.
identity_header := aes_encrypt(key: identity_subkey, plaintext: plaintext)
```

## UDP

In UDP request packets, identity headers are located between the separate header (session ID, packet ID) and AEAD ciphertext.

Response packets do not have identity headers.

```
plaintext := blake3::hash(iPSKn+1)[0..16] ^ session_id_packet_id // XOR to make it different for each packet.
identity_header := aes_encrypt(key: iPSKn, plaintext: plaintext)
```

For request packets, when iPSKs are used, the separate header MUST be encrypted with the first iPSK. Each identity processor MUST decrypt and re-encrypt the separate header with the next layer's PSK.

For response packets, no special handling is needed for the separate header. The server encrypts the separate header with the user's PSK, per the base spec. Identity processors MUST forward response packets directly to clients without parsing or modifying them.

## Scenarios

```
      client0       >---+
(iPSK0:iPSK1:uPSK0)      \
                          \
      client1       >------\                        +--->    server0 [iPSK1]
(iPSK0:iPSK1:uPSK1)         \                      /      [uPSK0, uPSK1, uPSK2]
                             >-> relay0 [iPSK0] >-<
      client2               /    [iPSK1, uPSK3]    \
(iPSK0:iPSK1:uPSK2) >------/                        +--->    server1 [uPSK3]
                          /
      client3            /
   (iPSK0:uPSK3)    >---+
```

A set of PSKs, delimited by `:`, are assigned to each client. To send a request, a client MUST generate one identity header for each iPSK.

A relay decrypts the first identity header with its identity key, looks up the PSK hash table to find the target server, and relays the remainder of the request.

A single-port-multi-user-capable server decrypts the identity header with its identity key, looks up the user PSK hash table to find the cipher for the user PSK, and processes the remainder of the request.

In the above graph, `client0`, `client1`, `client2` are users of `server0`, which is relayed through `relay0`. `server1` is a simple server without identity header support. `client3` connects to `server1` via `relay0`.

To start a TCP session, `client0` generates a random salt, encrypts iPSK1's hash with iPSK0-derived subkey as the 1st identity header, encrypts uPSK0's hash with iPSK1-derived subkey as the 2nd identity header, and finishes the remainder of the request following the original spec.

To process the TCP request, `relay0` decrypts the 1st identity header with iPSK0-derived subkey, looks up the PSK hash table, and writes the salt and remainder of the request (without the processed identity header) to `server0`.

To send a UDP packet, `client0` encrypts the separate header with iPSK0, encrypts (iPSK1's hash XOR session_id_packet_id) with iPSK0 as the 1st identity header, encrypts (uPSK0's hash XOR session_id_packet_id) with iPSK1 as the 2nd identity header, and finishes off following the original spec.

To process the UDP packet, `relay0` decrypts the separate header in-place with iPSK0, decrypts the 1st identity header with iPSK0, looks up the PSK hash table, re-encrypt the separate header into the place of the first identity header, and sends the packet (starting at the re-encrypted separate header) to `server0`.
