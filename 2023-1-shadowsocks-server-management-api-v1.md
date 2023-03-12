# Shadowsocks Server Management API v1

This document defines a RESTful API for managing Shadowsocks servers.

## 1. Security

The API server SHOULD use TLS client certificates to authenticate API users, or host the endpoints behind a secret path.

## 2. Standard Error Response

API errors MUST return a JSON object with at least an error message in the `error` field:

```json
{
    "error": "server not found"
}
```

## 3. Server Info

Get information about the server.

#### Request

```
GET /
```

#### Response: `200 OK`

```json
{
    "server": "shadowsocks-go 1.13.0",
    "apiVersion": "v1"
}
```

## 4. Users

A Shadowsocks server with EIH support can serve multiple users on a single port.

When a server has EIH configured, this API endpoint can be used to manage users. If EIH is not used, respond with 404 and a friendly error message.

### 4.1. List Users

#### Request

```
GET /users
```

#### Response: `200 OK`

```json
{
    "users": [
        {
            "username": "database64128",
            "uPSK": "oE/s2z9Q8EWORAB8B3UCxw=="
        }
    ]
}
```

### 4.2. Add User

#### Request

```
POST /users
```

```json
{
    "username": "database64128",
    "uPSK": "oE/s2z9Q8EWORAB8B3UCxw=="
}
```

#### Response: `201 Created`

Successfully added the new user.

#### Response: `400 Bad Request`

- A user with the same username already exists.
- Malformed request.

### 4.3. Get User Details

#### Request

```
GET /users/:username
```

#### Response: `200 OK`

```json
{
    "username": "database64128",
    "uPSK": "oE/s2z9Q8EWORAB8B3UCxw==",
    "downlinkPackets": 64,
    "downlinkBytes": 4096,
    "uplinkPackets": 32,
    "uplinkBytes": 2048,
    "tcpSessions": 16,
    "udpSessions": 8
}
```

#### Response: `404 Not Found`

User not found.

### 4.4. Update User

#### Request

```
PATCH /users/:username
```

```json
{
    "uPSK": "DL7Vrnv7sOdPPDghtxyxgg=="
}
```

#### Response: `204 No Content`

Successfully updated the user's uPSK.

#### Response: `400 Bad Request`

- The provided uPSK is the same as the existing one.
- Malformed request.

#### Response: `404 Not Found`

User not found.

### 4.5. Delete User

#### Request

```
DELETE /users/:username
```

#### Response: `204 No Content`

Successfully deleted the user.

#### Response: `404 Not Found`

User not found.

## 5. Stats

Get data usage stats.

#### Request

```
GET /stats
```

`?clear=true`: also clear stats

#### Response: `200 OK`

```json
{
    "downlinkPackets": 64,
    "downlinkBytes": 4096,
    "uplinkPackets": 32,
    "uplinkBytes": 2048,
    "tcpSessions": 16,
    "udpSessions": 8,
    "users": [
        {
            "username": "database64128",
            "downlinkPackets": 64,
            "downlinkBytes": 4096,
            "uplinkPackets": 32,
            "uplinkBytes": 2048,
            "tcpSessions": 16,
            "udpSessions": 8
        }
    ]
}
```
