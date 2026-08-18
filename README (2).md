<div align="center">

```
r o o t @ s n i f f e r : ~ / w r i t e u p s / h t t p - c y c l e #
```

# 🌐 The HTTP Request/Response Cycle

</div>

---

You load a page in under a second. Behind that second is a full round trip — a request built, sent, processed, and answered — happening in plain text, sitting right there in a Wireshark capture if you know where to look.

## 🔁 The Four Steps

```
1. Client sends a request   → method, URL, headers
2. Server processes it      → checks the route, queries data if needed
3. Server sends a response  → status code + headers + body
4. Client renders what it got
```

## 📤 Anatomy of a Request

```http
GET /products/42 HTTP/1.1
Host: example.com
User-Agent: Mozilla/5.0
Accept: text/html
Cookie: session_id=abc123
```

| Part | Meaning |
|---|---|
| `GET` | The HTTP method — what kind of action this is |
| `/products/42` | The path being requested |
| `HTTP/1.1` | Protocol version |
| `Host` | Which site, if the server hosts multiple domains |
| Headers | Metadata — what the client accepts, who it is, session info |

## 📥 Anatomy of a Response

```http
HTTP/1.1 200 OK
Content-Type: text/html
Content-Length: 4821

<html>...</html>
```

| Part | Meaning |
|---|---|
| `200 OK` | Status line — code + short description |
| `Content-Type` | What kind of data is in the body |
| `Content-Length` | Size of the body, in bytes |
| Body | The actual content — HTML, JSON, an image, whatever was requested |

## 🚦 Status Codes That Actually Matter

```
2xx — Success
  200 OK                    request succeeded
  201 Created                new resource created
  204 No Content              succeeded, nothing to return

3xx — Redirection
  301 Moved Permanently      resource has a new home, update your links
  304 Not Modified            cached version is still valid

4xx — Client Error
  400 Bad Request              malformed request
  401 Unauthorized             auth required, and missing/invalid
  403 Forbidden                 authenticated, but not allowed
  404 Not Found                 resource doesn't exist

5xx — Server Error
  500 Internal Server Error    the server broke, not you
  502 Bad Gateway                upstream server sent an invalid response
  503 Service Unavailable       server's overloaded or down for maintenance
```

A 4xx means check what you sent. A 5xx means it's not your fault — something broke on the other end.

## 🦈 Watching It Happen in Wireshark

```
Filter: http

Capture traffic while loading a page, and you'll see the full exchange
as plain text (for unencrypted HTTP) — method, headers, status code,
and body, all readable directly in the packet details pane.
```

For HTTPS, Wireshark shows you the TLS handshake wrapping around this same exchange — the request/response structure is identical underneath, just encrypted. See the [ARP Spoofing](../arp-spoofing) and [TCP Handshake](../tcp-handshake) writeups for what happens one and two layers below this.

## 🧩 Why This Matters Beyond "Making Websites"

- **Debugging** — a blank page with no errors almost always has an answer sitting in the Network tab (or a `http` Wireshark filter) as a status code you didn't expect.
- **API testing** — every API call is this exact same cycle, just usually returning JSON instead of HTML.
- **Threat hunting** — a spike of 401s from one IP is a brute-force login attempt. A spike of 404s scanning sequential paths is reconnaissance. The status codes tell the story before you read a single log line in detail.

---

<div align="center">

```
TYPE      FIELD WRITEUP
STATUS    ACTIVE
```

Reference version of this topic lives in the [tool repos](https://github.com/thedetectlab/Featured-Work) — this is the explanation.

</div>
