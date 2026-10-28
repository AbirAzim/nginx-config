# nginx-config


# EC2 Nginx + Docker Practice

Notes from setting up Nginx and Docker on an Ubuntu EC2 instance (t3.micro, ap-south-1).

## Server

| Item | Value |
|---|---|
| OS | Ubuntu (user `ubuntu`) |
| Key pair | `nginx-ec2.pem` (not committed) |
| Nginx config | `/etc/nginx/nginx.conf` |
| Web content | `/home/ubuntu/content` |
| Images | `/home/ubuntu/images` |

Connect:

```bash
ssh -i "nginx-ec2.pem" ubuntu@<public-ip>
```

## Nginx config

`/etc/nginx/nginx.conf`

```nginx
http {
	server {
		listen 8080;
		root /home/ubuntu/content;

		location /images {
			root /home/ubuntu/;
		}

		location ~ .png$ {
			return 404;
		}
	}

	server {
		listen 80;

		location / {
			proxy_pass http://<public-ip>:8080/;
		}

		location /img {
			proxy_pass http://<public-ip>:8080/images/;
		}
	}
}

events {}
```

### What it does

- **Port 8080** serves static files from `/home/ubuntu/content`.
  - `/images/...` is served from `/home/ubuntu/images/...` (Nginx appends the URL path to `root`).
  - Any URL ending in `png` returns 404.
- **Port 80** is a reverse proxy in front of port 8080.
  - `/` forwards to the 8080 server.
  - `/img/...` forwards to `/images/...` on the 8080 server.

### Improvements to make

- Use `proxy_pass http://127.0.0.1:8080/;` instead of the public IP. The public IP changes on stop/start and sends traffic out through AWS and back in.
- Escape the dot in the regex: `location ~ \.png$` (an unescaped `.` matches any character).
- Add `include mime.types;` inside `http { }` so CSS/JS/images get the right `Content-Type`.

## Nginx commands

```bash
sudo apt update && sudo apt install -y nginx
sudo systemctl enable --now nginx
sudo nginx -t                     # test config
sudo nginx -s reload              # apply config
sudo tail -5 /var/log/nginx/error.log
```

Always edit files in `/etc` with `sudo`.

## Permissions (fix for 403 Forbidden)

Nginx workers run as `www-data`, which counts as "others" for files in `/home/ubuntu`. Directories need `x` (pass through); files need `r` (read).

```bash
chmod o+x /home/ubuntu
chmod o+x /home/ubuntu/content
chmod o+r /home/ubuntu/content/index.html
# or for everything inside a folder:
chmod -R o+rX /home/ubuntu/content
```

## Security group inbound rules

| Type | Port | Purpose |
|---|---|---|
| SSH | 22 | Remote access (restrict source to My IP) |
| HTTP | 80 | Nginx reverse proxy |
| Custom TCP | 8080 | Nginx static server |
| Custom TCP | 1111 / 2222 / 3333 | Node app containers |

## Docker

Install (official repo):

```bash
sudo apt install -y ca-certificates curl
sudo install -m 0755 -d /etc/apt/keyrings
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc
echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/ubuntu $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
sudo apt update
sudo apt install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
sudo systemctl enable --now docker
sudo usermod -aG docker ubuntu    # then log out and back in
```

Build and run the Node app:

```bash
cd ~/nodecicd
docker build -t nodeapp:latest .
docker run -d --name nodeapp1 -p 1111:8080 nodeapp:latest
docker run -d --name nodeapp2 -p 2222:8080 nodeapp:latest
docker run -d --name nodeapp3 -p 3333:8080 nodeapp:latest
docker ps
curl -i http://localhost:1111
```

`-p HOST:CONTAINER` — the right side must match the port the app listens on inside the container.

## Copying files from Mac to EC2

Run these on the Mac, not the EC2.

```bash
scp -i "nginx-ec2.pem" photo.jpg ubuntu@<public-ip>:/home/ubuntu/content/
scp -i "nginx-ec2.pem" -r ./folder ubuntu@<public-ip>:/home/ubuntu/

# with excludes
rsync -avz -e "ssh -i nginx-ec2.pem" --exclude node_modules --exclude .git \
  ./project/ ubuntu@<public-ip>:~/project/
```
