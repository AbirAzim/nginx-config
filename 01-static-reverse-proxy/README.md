# 01 · Static file server + reverse proxy

Config: [`nginx.conf`](nginx.conf)

Two `server` blocks on one machine: one serves files from disk, the other sits in front of it as a reverse proxy.

## What it does

| Port | Request | Result |
|---|---|---|
| 8080 | `/` | Files from `/home/ubuntu/content` |
| 8080 | `/images/a.jpg` | `/home/ubuntu/images/a.jpg` (Nginx appends the URL path to `root`) |
| 8080 | anything ending in `.png` | `404` |
| 80 | `/...` | Proxied to `http://127.0.0.1:8080/...` |
| 80 | `/img/...` | Proxied to `http://127.0.0.1:8080/images/...` |

## Key points

- **`root` vs URL path.** With `root /home/ubuntu/;` inside `location /images`, the full URI is appended, so `/images/a.jpg` becomes `/home/ubuntu/images/a.jpg`.
- **Regex locations.** `location ~ \.png$` is a case-sensitive regex. Escape the dot, because an unescaped `.` matches any character.
- **Trailing slash in `proxy_pass`.** `proxy_pass http://127.0.0.1:8080/images/;` swaps the matched prefix `/img` for `/images/`.
- **Use `127.0.0.1`, not the public IP.** The public IP changes on stop/start and sends traffic out through AWS and back in.
- `include mime.types;` makes CSS, JS and images go out with the correct `Content-Type`.

## Setup

```bash
mkdir -p ~/content ~/images
echo '<h1>Hello</h1>' > ~/content/index.html
chmod -R o+rX /home/ubuntu/content /home/ubuntu/images
chmod o+x /home/ubuntu

sudo cp nginx.conf /etc/nginx/nginx.conf
sudo nginx -t && sudo nginx -s reload
```

## Test

```bash
curl -i http://localhost:8080/          # index.html
curl -i http://localhost:8080/x.png     # 404
curl -i http://localhost/               # same page, through the proxy
curl -i http://localhost/img/a.jpg      # served from ~/images/a.jpg
```

If you get `403 Forbidden`, see [Permissions](../README.md#permissions-fixing-403-forbidden) in the main README.
