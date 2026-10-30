# 02 · Layer 7 (HTTP) load balancer

Config: [`nginx.conf`](nginx.conf)

Spreads HTTP requests on port `8080` across three copies of the same Node app running in Docker on ports `1111`, `2222` and `3333`.

## What it does

- The `upstream backends` block defines the pool. With no algorithm set, Nginx uses **round robin**.
- `proxy_pass http://backends/;` sends each request to the next server in the pool.
- The `proxy_set_header` lines pass along the original host, client IP and scheme. Without them, the app only sees the Docker bridge gateway (`172.17.0.1`) as the client.

## Balancing algorithms

| Directive (inside `upstream`) | Behaviour |
|---|---|
| *(none)* | Round robin |
| `ip_hash;` | Same client goes to the same backend. Hashes the first 3 IPv4 octets modulo N, which is **not** consistent hashing, so adding or removing a server reshuffles most clients |
| `hash $remote_addr consistent;` | Real consistent hashing (ketama ring). Only about 1/N of clients move when the pool changes |
| `least_conn;` | Goes to the backend with the fewest active connections |

## Prerequisites

Run the three containers (see [Docker](../README.md#docker) in the main README):

```bash
docker run -d --name nodeapp1 -e APPID=1111 -p 1111:8080 nodeapp:latest
docker run -d --name nodeapp2 -e APPID=2222 -p 2222:8080 nodeapp:latest
docker run -d --name nodeapp3 -e APPID=3333 -p 3333:8080 nodeapp:latest
```

## Test

```bash
sudo cp nginx.conf /etc/nginx/nginx.conf
sudo nginx -t && sudo nginx -s reload

for i in 1 2 3 4 5 6; do curl -s http://localhost:8080 | grep -o '"appId":"[0-9]*"'; done
# "appId":"1111" "appId":"2222" "appId":"3333" "appId":"1111" ...
```

Uncomment `ip_hash;`, reload, and run the loop again. Every request should now return the same `appId`.
