# nginx-config

Hands-on Nginx configs and notes from an Ubuntu EC2 instance. Covers static hosting, reverse proxying, load balancing Docker containers at layer 7 and layer 4, Let's Encrypt TLS, HTTP/2 and HTTP/3.

Each config has its own folder with an `nginx.conf` (with a header comment explaining it) and a `README.md`.

> Addresses are placeholders. `<public-ip>` is the instance's public IPv4.

## Configs

| # | Folder | What it shows | Ports |
|---|---|---|---|
| 01 | [`01-static-reverse-proxy`](01-static-reverse-proxy/) | Static files from `/home/ubuntu`, regex `location`, reverse proxy | 80, 8080 |
| 02 | [`02-load-balancer-l7`](02-load-balancer-l7/) | HTTP round-robin across 3 Docker containers, `X-Forwarded-For`, `ip_hash` / consistent hashing | 8080 |
| 03 | [`03-path-based-load-split`](03-path-based-load-split/) | Different upstream pools per URL path (`/node1`, `/node2`) | 8080 |
| 04 | [`04-load-balancer-l4-stream`](04-load-balancer-l4-stream/) | TCP load balancing with the `stream` module, L4 vs L7 | 8080 |
| 05 | [`05-https-http3`](05-https-http3/) | Let's Encrypt TLS, HTTP/2, HTTP/3 (QUIC), `Alt-Svc` | 80, 443 tcp+udp |

To try one:

```bash
sudo cp 02-load-balancer-l7/nginx.conf /etc/nginx/nginx.conf
sudo nginx -t && sudo nginx -s reload
```

```
.
├── README.md                         ← you are here (server setup, Docker, general notes)
├── 01-static-reverse-proxy/
│   ├── nginx.conf
│   └── README.md
├── 02-load-balancer-l7/
│   ├── nginx.conf
│   └── README.md
├── 03-path-based-load-split/
│   ├── nginx.conf
│   └── README.md
├── 04-load-balancer-l4-stream/
│   ├── nginx.conf
│   └── README.md
└── 05-https-http3/
    ├── nginx.conf
    └── README.md
```

---

## Server

| Item | Value |
|---|---|
| Instance | EC2 `t3.micro` (ap-south-1), Ubuntu (user `ubuntu`) |
| Key pair | `nginx-ec2.pem` (never commit it) |
| Domain | `nginx-badhon.webhop.me` (No-IP free DNS) |
| Nginx config | `/etc/nginx/nginx.conf` |
| Static content | `/home/ubuntu/content` |
| Images | `/home/ubuntu/images` |
| Node app | `/home/ubuntu/nodecicd` |

Connect (run on the Mac):

```bash
chmod 400 nginx-ec2.pem
ssh -i "nginx-ec2.pem" ubuntu@<public-ip>
```

- Amazon Linux uses `ec2-user` and `dnf`. Ubuntu uses `ubuntu` and `apt`.
- The shell prompt shows the **private** IP (`ip-172-31-x-x`). From outside, always connect with the **public** IP or the domain.
- The public IP changes on stop/start unless an **Elastic IP** is attached.

---

## Install Nginx

```bash
sudo apt update
sudo apt install -y nginx
sudo systemctl enable --now nginx     # start now + on every boot
curl -I http://localhost              # expect HTTP/1.1 200 OK
```

Useful paths on Ubuntu:

| Path | Purpose |
|---|---|
| `/etc/nginx/nginx.conf` | Main config |
| `/etc/nginx/sites-available/default` | Default site (only loaded if `nginx.conf` includes it) |
| `/etc/nginx/modules-enabled/` | One `load_module` file per installed module |
| `/var/www/html/` | Default web root |
| `/var/log/nginx/error.log` | Errors. Check this first |

Everyday commands:

```bash
sudo nano /etc/nginx/nginx.conf       # always use sudo for /etc
sudo nginx -t                         # test config
sudo nginx -s reload                  # apply (or: sudo systemctl reload nginx)
sudo nginx -T                         # print full merged config
sudo tail -5 /var/log/nginx/error.log
```

Replace the whole config in one go:

```bash
sudo tee /etc/nginx/nginx.conf > /dev/null <<'CONF'
... config ...
CONF
```

### Common config mistakes

- `listen: 8080;` is wrong. There is no colon: `listen 8080;`
- A missing `;` at the end of a directive
- `http://` written twice in `proxy_pass`
- `http://` inside an `upstream` server line (only `host:port` is allowed)
- `127.0.0.0` instead of `127.0.0.1` (`127.0.0.0` is the network address, not localhost)
- An unescaped dot in regex: `~ .png$` instead of `~ \.png$`

Quick fix without an editor:

```bash
sudo sed -i 's/listen: /listen /' /etc/nginx/nginx.conf
```

---

## Permissions: fixing `403 Forbidden`

Nginx workers run as `www-data`, which counts as **others** for files in `/home/ubuntu`. To read a file, a user needs `x` on **every directory** in the path and `r` on the file itself.

```bash
chmod o+x /home/ubuntu
chmod o+x /home/ubuntu/content
chmod o+r /home/ubuntu/content/index.html
# or everything inside a folder (capital X = dirs only):
chmod -R o+rX /home/ubuntu/content
```

| Permission | On a file | On a directory |
|---|---|---|
| `r` | read contents | list contents |
| `w` | modify | create/delete inside |
| `x` | execute | enter / pass through |

A tighter alternative is to give access to the group instead of everyone:

```bash
sudo usermod -aG ubuntu www-data
chmod g+x /home/ubuntu
sudo systemctl restart nginx          # restart, not reload, to pick up new groups
```

On real servers, keep site files in `/var/www/` instead.

---

## Copy files from the Mac to EC2

Run these **on the Mac**, not on the EC2.

```bash
# one file
scp -i "nginx-ec2.pem" photo.jpg ubuntu@<public-ip>:/home/ubuntu/content/

# a folder
scp -i "nginx-ec2.pem" -r ./folder ubuntu@<public-ip>:/home/ubuntu/

# EC2 → Mac
scp -i "nginx-ec2.pem" ubuntu@<public-ip>:~/ec2-history.txt ~/Downloads/

# folder with excludes (scp can't exclude)
rsync -avz -e "ssh -i nginx-ec2.pem" \
  --exclude node_modules --exclude .git --exclude .env \
  ./nodecicd/ ubuntu@<public-ip>:~/nodecicd/
```

A trailing slash on the rsync source copies the folder's **contents**. Add `-n` for a dry run.

---

## Docker

The backends used by configs 02–05.

### Install (official repo)

```bash
sudo apt update
sudo apt install -y ca-certificates curl
sudo install -m 0755 -d /etc/apt/keyrings
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc
echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/ubuntu $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
sudo apt update
sudo apt install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
sudo systemctl enable --now docker
sudo usermod -aG docker ubuntu        # then log out and back in (or: newgrp docker)
docker run hello-world
```

`permission denied ... docker.sock` means the docker group isn't active in this session yet.

### Build and run three copies of the same app

```bash
cd ~/nodecicd
docker build -t nodeapp:latest .

docker run -d --name nodeapp1 -e APPID=1111 -p 1111:8080 nodeapp:latest
docker run -d --name nodeapp2 -e APPID=2222 -p 2222:8080 nodeapp:latest
docker run -d --name nodeapp3 -e APPID=3333 -p 3333:8080 nodeapp:latest

docker ps
curl -i http://localhost:1111
```

- `-p HOST:CONTAINER`: the right side must match the port the app listens on inside the container.
- `-e APPID=1111` (with `=`) runs one image with different config per container.
- The app must listen on `0.0.0.0`, not `localhost`.

### Cleanup

```bash
docker ps -a                          # includes stopped containers
docker logs nodeapp1                  # why did it exit?
docker rm -f nodeapp1 nodeapp2 nodeapp3
docker container prune -f
docker rmi nodeapp:latest
```

`docker stop` only stops a container. It still exists and blocks `docker rmi`. After code changes, rebuild the image **and** recreate the containers, because `docker restart` reuses the old image.

### Identify which container answered

```js
const os = require('os');

const app = express();
app.set('trust proxy', true);   // read real client IP from X-Forwarded-For

app.get('/', (req, res) => {
    res.json({
        message: 'Hello World',
        appId: process.env.APPID ?? 'unknown',
        hostname: os.hostname(),          // container ID, or set with --hostname
        ip: req.ip?.replace('::ffff:', ''),
        method: req.method,
        path: req.path,
        time: new Date().toISOString(),
    });
});

app.listen(8080, '0.0.0.0');
```

Why did the IP show `172.17.0.1`? That is the Docker bridge gateway. Nginx and Docker each open a new connection, so the app only sees the last hop. The fix is for Nginx to send `X-Forwarded-For` (see [02](02-load-balancer-l7/)) and for Express to set `trust proxy`.

---

## Domain + DNS

Free hostname from No-IP: <https://my.noip.com/dns/records?jump_to_zone=webhop.me>
Point the A record at the EC2 public IP. Attach an Elastic IP so the address doesn't change.

```bash
ping nginx-badhon.webhop.me          # should resolve to the EC2 IP
```

Certificate setup and HTTP/3 are covered in [05-https-http3](05-https-http3/).

---

## Security group

| Type | Protocol | Port | Why |
|---|---|---|---|
| SSH | TCP | 22 | Remote access (better: source = My IP) |
| HTTP | TCP | 80 | Plain HTTP / certbot challenge |
| HTTPS | TCP | 443 | First connection, HTTP/1.1 + HTTP/2 |
| Custom UDP | UDP | 443 | HTTP/3 (QUIC) |
| Custom TCP | TCP | 8080 | Only while testing configs 01–04 from outside |

Lesson learned: HTTPS needs **both** TCP 443 and UDP 443. Replacing the TCP rule with UDP broke the site entirely, because browsers always connect over TCP first.

Ports 1111/2222/3333 don't need to be public, because Nginx reaches the containers on `127.0.0.1`.

---

## Saving work

```bash
history -a                            # flush current session to ~/.bash_history
history > ~/ec2-history.txt
history | grep certbot                # search
```

Full server backup: EC2 → Instance → Actions → Image and templates → **Create image (AMI)**.
