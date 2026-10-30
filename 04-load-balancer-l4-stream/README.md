# 04 · Layer 4 (TCP) load balancer: `stream`

Config: [`nginx.conf`](nginx.conf)

The same three backends as [02](../02-load-balancer-l7/), but proxied at the **TCP level** with the `stream` module instead of `http`.

## What it does

- `stream { }` sits next to `http { }` at the top level, not inside it.
- Nginx forwards raw bytes. It never parses HTTP.
- Balancing happens **per TCP connection**, not per request. A browser reuses one keep-alive connection, so it keeps hitting the same backend. `curl` opens a new connection each time, so you will see round robin there.

## Layer 7 vs layer 4

| | `http` (layer 7) | `stream` (layer 4) |
|---|---|---|
| Sees | URL, headers, method | Raw TCP bytes |
| `location` routing | Yes | No |
| Add headers (`X-Forwarded-For`) | Yes | No |
| Balances per | Request | TCP connection |
| Works for | HTTP(S) | Any TCP/UDP: databases, SSH, MQTT, … |

## Setup

On Ubuntu the stream module is a separate package, which the `include` line at the top loads:

```bash
sudo apt install -y libnginx-mod-stream
sudo cp nginx.conf /etc/nginx/nginx.conf
sudo nginx -t && sudo nginx -s reload
```

If you see `unknown directive "stream"`, the module is not installed or the `include` line is missing.

## Test

```bash
for i in 1 2 3; do curl -s http://localhost:8080 | grep -o '"appId":"[0-9]*"'; done
```
