# Shadowsocks Server Configuration URL

This document defines the URL format for Shadowsocks server configuration sharing.

Most programming languages provide utilities for parsing and assembling URLs in their standard libraries. Feed the following information into a URL builder to produce a URL.

- Scheme: `ss`
- Userinfo
    - Username: method
    - Password: password
- Fragment: server name

Examples:

```
ss://2022-blake3-aes-128-gcm:5mOQSa20Kt6ay2LXruBoHQ%3D%3D@example.com:443/#name
ss://2022-blake3-aes-256-gcm:t7XRzLCvgsH4r4r669cyqPnVNFG2c%2FHC5Tt%2BMjINJB0%3D@[2001:db8:1f74:3c86:aef9:a75:5d2a:425e]:20220/#name
```
