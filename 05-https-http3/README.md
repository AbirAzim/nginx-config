# 05 · HTTPS + HTTP/2 + HTTP/3 (QUIC)

Config: [`nginx.conf`](nginx.conf)

The final setup. It is the load balancer from [02](../02-load-balancer-l7/) behind a real domain (`nginx-badhon.webhop.me`), with a Let's Encrypt certificate, HTTP/2 over TCP and HTTP/3 over UDP.

## What it does

| Listener | Protocol |
|---|---|
| `listen 80;` / `listen 8080;` | Plain HTTP |
| `listen 443 ssl;` + `http2 on;` | HTTPS over **TCP**: HTTP/1.1 and HTTP/2 |
| `listen 443 quic reuseport;` | HTTP/3 over **UDP** (QUIC) |
| `add_header Alt-Svc 'h3=":443"; ma=86400';` | Tells the browser that HTTP/3 is available on UDP 443 |

How the browser upgrades:

```
1st request:   TCP 443 → HTTP/2 → response has Alt-Svc: h3=":443"
next requests: UDP 443 → HTTP/3 (QUIC)
```

## Prerequisites

1. **DNS.** An A record for the domain points at the EC2 public IP. Attach an Elastic IP so the address doesn't change.
2. **Certificate.** Port 80 must be open and free, because `--standalone` runs its own web server for the http-01 challenge:
   ```bash
   sudo apt install -y certbot
   sudo systemctl stop nginx
   sudo certbot certonly --standalone -d nginx-badhon.webhop.me
   sudo systemctl start nginx
   ```
   Renewal (certs last 90 days):
   ```bash
   sudo certbot renew --dry-run --pre-hook "systemctl stop nginx" --post-hook "systemctl start nginx"
   ```
3. **Nginx with HTTP/3 support.** `nginx -V 2>&1 | grep -o http_v3_module` must print a match.
4. **Security group.** Open **both** TCP 443 and UDP 443. Browsers always connect over TCP first, so opening only UDP breaks the site.

## Gotchas

- Old tutorials use `listen 443 ssl http3;`. Current Nginx rejects that with `invalid parameter "http3"`. Use `quic` instead.
- `reuseport` may appear on only one `listen ... quic` line per port.

## Verify

```bash
curl -sI https://nginx-badhon.webhop.me | grep -i alt-svc   # server advertises h3
sudo ss -ulnp | grep 443                                    # nginx listening on UDP
sudo tcpdump -ni any udp port 443                           # QUIC packets arriving?
```

In Chrome, open DevTools → Network, right-click the column headers and enable **Protocol**. The first load shows `h2` and later loads show `h3`. If it stays on `h2`, quit Chrome fully (Cmd+Q) and reload a few times. `chrome://net-internals/#quic` lists active QUIC sessions.

TLS check: <https://www.cdn77.com/tls-test/result?domain=nginx-badhon.webhop.me>
