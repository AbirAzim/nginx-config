# 03 · Path-based load splitting

Config: [`nginx.conf`](nginx.conf)

Builds on [02](../02-load-balancer-l7/). It sends traffic to **different upstream pools depending on the URL path**, which is something only a layer 7 proxy can do.

## What it does

| Path | Upstream | Backends |
|---|---|---|
| `/` | `backends` | 1111, 2222, 3333 |
| `/node1` | `load1` | 1111, 2222 |
| `/node2` | `load2` | 3333 |

Each pool load-balances on its own, using round robin.

## Key points

- The trailing slash in `proxy_pass http://load1/;` strips the matched prefix, so `/node1/foo` reaches the backend as `/foo`.
- Typical real-world use: `/api` goes to API servers, `/static` to file servers, or a canary pool gets a specific path.

## Test

```bash
sudo cp nginx.conf /etc/nginx/nginx.conf
sudo nginx -t && sudo nginx -s reload

for i in 1 2 3 4; do curl -s http://localhost:8080/node1 | grep -o '"appId":"[0-9]*"'; done   # 1111 / 2222
for i in 1 2 3 4; do curl -s http://localhost:8080/node2 | grep -o '"appId":"[0-9]*"'; done   # always 3333
```
