# Shadowsocks 2022 Implementations

This document tracks and compares implementations of Shadowsocks 2022 by feature and spec compliance, and showcases some benchmark and speedtest results.

## 1. Implementations

### 1.1. Reference Implementations

Reference implementations of Shadowsocks 2022 are fully spec-compliant, performance-optimized, and ready for production.

|     | shadowsocks-go | shadowsocks-rust | sing-box |
| --- | -------------: | ---------------: | -------: |
| Description | A versatile and efficient proxy platform for secure communications. | CLIs and libraries written in Rust. | The universal proxy platform. |
| URL | https://github.com/database64128/shadowsocks-go | https://github.com/shadowsocks/shadowsocks-rust | https://github.com/SagerNet/sing-shadowsocks, https://github.com/SagerNet/sing-box |
| License | AGPLv3 | MIT | GPLv3 |
| Server  | ✅ | ✅ | ✅ |
| Relay   | ❌ | ❌ | ✅ |
| Client  | ✅ | ✅ | ✅ |
| [EIH (Extensible Identity Headers)](2022-2-shadowsocks-2022-extensible-identity-headers.md) | ✅ | ✅ | ✅ |

### 1.2. Derived Implementations

Derived implementations are based on one of the reference implementations but developed separately by downstream developers.

|     | v2ray-core (SagerNet Fork) | Xray-core |
| --- | -------------------------: | --------: |
| URL | https://github.com/SagerNet/v2ray-core | https://github.com/XTLS/Xray-core |
| Protocol Implementation | sing-shadowsocks | sing-shadowsocks |
| License | GPLv3 | MPLv2 |
| Server  | ✅ | ✅ |
| Relay   | ✅ | ✅ |
| Client  | ✅ | ✅ |
| [EIH (Extensible Identity Headers)](2022-2-shadowsocks-2022-extensible-identity-headers.md) | ✅ | ✅ |

### 1.3. GUI Clients

### 1.4. Other Implementations

## 2. Benchmarks

### 2.1. Header Benchmarks

https://github.com/database64128/cubic-go-playground/blob/main/shadowsocks/udpheader/udpheader_test.go

```
BenchmarkGenSaltHkdfSha1-8          	  372712	      3283 ns/op	    1264 B/op	      18 allocs/op
BenchmarkGenSaltBlake3-8            	  770426	      1554 ns/op	       0 B/op	       0 allocs/op
BenchmarkAesEcbHeaderEncryption-8   	99224395	        12.84 ns/op	       0 B/op	       0 allocs/op
BenchmarkAesEcbHeaderDecryption-8   	100000000	        11.79 ns/op	       0 B/op	       0 allocs/op
```

### 2.2. Full UDP Packet Construction Benchmarks

https://github.com/database64128/cubic-go-playground/blob/main/shadowsocks/udp_test.go

```
BenchmarkShadowsocksAEADAes256GcmEncryption-8             	  252819	      4520 ns/op	    2160 B/op	      23 allocs/op
BenchmarkShadowsocksAEADAes256GcmWithBlake3Encryption-8   	  320718	      3367 ns/op	     896 B/op	       5 allocs/op
BenchmarkDraftSeparateHeaderAes256GcmEncryption-8         	 2059383	       590.5 ns/op	       0 B/op	       0 allocs/op
BenchmarkDraftXChaCha20Poly1305Encryption-8               	  993336	      1266 ns/op	       0 B/op	       0 allocs/op
```

### 2.3. shadowsocks-rust Speed Tests

|       shadowsocks-rust        |   TCP    |   UDP    |
| ----------------------------- | -------: | -------: |
| 2022-blake3-aes-128-gcm       | 12.2Gbps | 14.2Gbps |
| 2022-blake3-aes-256-gcm       | 10.9Gbps | 12.5Gbps |
| 2022-blake3-chacha20-poly1305 | 8.05Gbps | 2.35Gbps |
| 2022-blake3-chacha8-poly1305  | 8.36Gbps | 2.60Gbps |
| aes-128-gcm                   | 8.99Gbps | 13.5Gbps |
| aes-256-gcm                   | 8.21Gbps | 11.9Gbps |
| chacha20-poly1305             | 6.55Gbps | 8.66Gbps |
