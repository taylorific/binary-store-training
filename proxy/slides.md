---
layout: section
---

# Reverse proxies: Caddy and nginx

<br>
<br>
<Link to="toc" title="Table of Contents"/>

---
hideInToc: true
level: 2
---

# Why put a proxy in front

Nexus, Pulp and Harbor all serve plain HTTP on their own ports. A reverse
proxy in front gives clients:

- **HTTPS**, with certificates managed in one place instead of in each store
- **Names instead of ports:** `docker-proxy.example.com` instead of
  `localhost:10080`, one name per Nexus Docker connector
- **One way in:** the stores can listen on `localhost` only, leaving the proxy
  the only thing on ports 80 and 443

Some clients insist on HTTPS. Docker treats any plain-HTTP registry other than
`localhost` as "insecure" and needs `daemon.json` changes on each client, and
Apple's `container` refuses Pulp's token auth over plain HTTP.

This section puts Nexus behind Caddy, then nginx, each **installed on the host**
and **in a container**.

---
hideInToc: true
level: 2
class: text-sm
---

# What a binary store needs from a proxy

A web page is small. A container image layer or an ISO isn't.

| Need | Why | Get it wrong and |
| --- | --- | --- |
| **No upload size limit** | Image layers and artifacts run to gigabytes | `413 Request Entity Too Large` on `docker push` or an upload |
| **Streaming, not buffering** | Clients send layers in chunks; the proxy shouldn't hold them first | Stalled or failed pushes, full temp disks |
| **Patience** | A store can go quiet for minutes: fetching a big file upstream before it answers, or storing an upload | `504 Gateway Timeout` after 60 seconds of silence |
| **Forwarded headers** (`Host`, `X-Forwarded-Proto`) | The store writes its own URL into links and redirects | Clients sent to `http://nexus:8081/...` |
| **Certificates clients trust** | Internal names can't get public certificates over HTTP | `x509: certificate signed by unknown authority` |

Caddy does the first four by default. nginx does none of them by default.

---
hideInToc: true
level: 2
class: text-xs
---

# Caddy and nginx

| | Caddy | nginx |
| --- | --- | --- |
| **Upload size** | No limit | 1 MB unless you set `client_max_body_size` |
| **Streaming** | Streams both ways | Buffers both ways unless you turn it off |
| **A silent store** | Waits for it | `504` after 60 seconds unless you raise the timeouts |
| **Forwarded headers** | Sent for you | You add each one |
| **HTTP to HTTPS redirect** | Automatic | A `server` block you write |
| **Certificates** | Issues and renews its own: a local CA (`tls internal`) or Let's Encrypt | You bring them and renew them |
| **You'll meet it in** | Newer projects | Every vendor's docs. Harbor and the `pulp/pulp` image run it inside |

Two others you'll see: **Traefik** finds containers by their labels, which
suits container platforms more than a few fixed hosts. **HAProxy** is a
load balancer first, with nothing built in for registries.

---
hideInToc: true
level: 2
---

# The lab

The Nexus from the earlier sections, with its `docker-proxy` connector on
port 10080, plus a Docker hosted repository on 10081 to push to:

```bash
NEXUS=http://localhost:8081/service/rest
curl --include --fail \
  --user admin:$newpass \
  -X POST \
  -H 'Content-Type: application/json' \
  -d '{
    "name": "docker-hosted",
    "online": true,
    "storage": {"blobStoreName": "default", "strictContentTypeValidation": true, "writePolicy": "ALLOW"},
    "docker": {"v1Enabled": false, "forceBasicAuth": true, "httpPort": 10081}
  }' \
  $NEXUS/v1/repositories/docker/hosted
```

---
hideInToc: true
level: 2
---

# Point the names at the proxy

| Name | Goes to |
| --- | --- |
| `nexus.example.com` | Nexus UI, API and every non-Docker repository, port 8081 |
| `docker-proxy.example.com` | The `docker-proxy` connector, port 10080 |
| `docker-hosted.example.com` | The `docker-hosted` connector, port 10081 |

In a real network these are DNS records for the proxy's address. On a
one-machine lab, `/etc/hosts` will do:

```bash
echo "127.0.0.1 nexus.example.com docker-proxy.example.com docker-hosted.example.com" | \
  sudo tee -a /etc/hosts
```

Do the same on each client, with the proxy host's address instead of
`127.0.0.1`.

`example.com` names can never get a public certificate, so both proxies
use certificates from a CA of their own. The clients have to trust that CA.

---
layout: section
level: 2
---

# Caddy

<br>
<br>
<Link to="toc" title="Table of Contents"/>

---
hideInToc: true
level: 2
---

# Install Caddy on the host

Ubuntu's own `caddy` package lags behind. Caddy publishes its own apt
repository:

```bash
sudo apt-get install -y debian-keyring debian-archive-keyring apt-transport-https curl gpg
curl -1sLf 'https://dl.cloudsmith.io/public/caddy/stable/gpg.key' | \
  sudo gpg --dearmor -o /usr/share/keyrings/caddy-stable-archive-keyring.gpg
curl -1sLf 'https://dl.cloudsmith.io/public/caddy/stable/debian.deb.txt' | \
  sudo tee /etc/apt/sources.list.d/caddy-stable.list
sudo chmod o+r /usr/share/keyrings/caddy-stable-archive-keyring.gpg \
  /etc/apt/sources.list.d/caddy-stable.list
sudo apt-get update && sudo apt-get install -y caddy

$ caddy version
v2.11.9 h1:bhZtcRKQOq0fuLvEgCdPYvUxXyGn9kzjG4lh2332DTE=
```

The package adds a `caddy` user and a systemd unit that runs
`caddy run --config /etc/caddy/Caddyfile`. The unit grants
`CAP_NET_BIND_SERVICE`, so Caddy binds ports 80 and 443 without running as
root.

---
hideInToc: true
level: 2
class: text-sm
---

# The Caddyfile

```bash
sudo tee /etc/caddy/Caddyfile > /dev/null <<'EOF'
nexus.example.com {
	tls internal
	reverse_proxy localhost:8081
}

docker-proxy.example.com {
	tls internal
	reverse_proxy localhost:10080
}

docker-hosted.example.com {
	tls internal
	reverse_proxy localhost:10081
}
EOF
caddy validate --config /etc/caddy/Caddyfile
sudo systemctl reload caddy
```

- `tls internal`: a certificate from Caddy's own CA instead of Let's Encrypt
- Nothing else to set: no size limit or timeouts, streaming, `X-Forwarded-*`
  headers, and port 80 redirects to HTTPS

---
hideInToc: true
level: 2
---

# Caddy's certificate authority

Caddy makes a CA the first time it starts with `tls internal`:

```bash
$ sudo ls /var/lib/caddy/.local/share/caddy/pki/authorities/local/
intermediate.crt  intermediate.key  root.crt  root.key

$ sudo openssl x509 -noout -subject -enddate \
    -in /var/lib/caddy/.local/share/caddy/pki/authorities/local/root.crt
subject=CN = Caddy Local Authority - 2026 ECC Root
notAfter=Sep 26 18:11:50 2036 GMT

$ echo | openssl s_client -connect localhost:443 -servername nexus.example.com 2>/dev/null | \
    openssl x509 -noout -issuer -enddate
issuer=CN = Caddy Local Authority - ECC Intermediate
notAfter=Nov 19 06:11:50 2026 GMT
```

- The **root** lasts ten years. That's what clients trust
- Site certificates last **12 hours**, and Caddy renews them on its own
- `root.key` signs everything. Back up the directory, and keep it private

---
hideInToc: true
level: 2
---

# Trust the root on clients

Copy `root.crt` from the proxy host to each client, then:

```bash
# Ubuntu and Debian: curl, apt, pip and most tools
sudo cp root.crt /usr/local/share/ca-certificates/caddy-local-root.crt
sudo update-ca-certificates
1 added, 0 removed; done.

# Docker Engine: one directory per registry name, no restart needed
for h in docker-proxy.example.com docker-hosted.example.com; do
  sudo mkdir -p /etc/docker/certs.d/$h
  sudo cp root.crt /etc/docker/certs.d/$h/ca.crt
done

# macOS: the System keychain, used by Safari, Docker Desktop
# (after a restart) and Apple's container
sudo security add-trusted-cert -d -r trustRoot \
  -k /Library/Keychains/System.keychain root.crt
```

Java tools such as Maven and Gradle use the JDK's own trust store (`changeit`
is its default password):
`sudo keytool -importcert -cacerts -storepass changeit -noprompt -alias caddy-local-root -file root.crt`

<!--
The macOS step hasn't been run in the lab yet; the Ubuntu, Docker Engine and Java ones have.
-->

---
hideInToc: true
level: 2
---

# Try it

```bash
# A 200 MB upload: past nginx's default 1 MB limit, fine through Caddy
head -c 200000000 /dev/urandom > big.bin
$ curl --silent --user admin:$newpass --upload-file big.bin \
    --output /dev/null --write-out '%{http_code}\n' \
    https://nexus.example.com/repository/testy-hosted/big.bin
201

# Nexus builds its URLs from the forwarded headers
$ curl --silent --user admin:$newpass \
    https://nexus.example.com/service/rest/v1/repositories | jq -r '.[0].url'
https://nexus.example.com/repository/testy-hosted

# Pull through the proxy repository, push to the hosted one
docker pull docker-proxy.example.com/library/python:3.13-slim
docker tag docker-proxy.example.com/library/python:3.13-slim \
  docker-hosted.example.com/python:3.13-slim
echo $newpass | docker login -u admin --password-stdin docker-hosted.example.com
docker push docker-hosted.example.com/python:3.13-slim
```

---
hideInToc: true
level: 2
class: text-sm
---

# Caddy in a container

This replaces the host install: both need ports 80 and 443. Put the proxy and
Nexus on one Docker network, so Caddy reaches Nexus by container name. In the
Caddyfile, `localhost:8081` becomes `nexus:8081`:

```bash
docker network create binary-stores
docker network connect binary-stores nexus

mkdir -p caddy
sed 's/localhost:/nexus:/' /etc/caddy/Caddyfile > caddy/Caddyfile

docker container run --detach \
  --name caddy \
  --network binary-stores \
  --publish 80:80 --publish 443:443 \
  --volume "$(pwd)/caddy/Caddyfile":/etc/caddy/Caddyfile:ro \
  --volume caddy_data:/data \
  --volume caddy_config:/config \
  docker.io/library/caddy:2.11.9

# The root clients need to trust
docker container cp caddy:/data/caddy/pki/authorities/local/root.crt .
```

After editing the Caddyfile:
`docker container exec caddy caddy reload --config /etc/caddy/Caddyfile`

---
hideInToc: true
level: 2
---

# Keep /data on a volume

`/data` holds the CA. Without a volume, every new container makes a new root,
and every client that trusted the old one fails:

```bash
# Two containers without a /data volume
sha256 Fingerprint=10:E6:90:EA:F2:A7:0E:...
sha256 Fingerprint=5E:6F:B7:C4:DD:D7:C5:...

# Two containers sharing the caddy_data volume
sha256 Fingerprint=56:F7:EB:B2:F4:74:71:...
sha256 Fingerprint=56:F7:EB:B2:F4:74:71:...
```

After the root changes, Docker reports:

```text
tls: failed to verify certificate: x509: certificate signed by unknown authority
```

Back up the `caddy_data` volume like any other data.

---
hideInToc: true
level: 2
---

# Real certificates

When the names are in a public DNS zone you control, drop `tls internal`.
Caddy then gets certificates from Let's Encrypt and renews them:

```text
nexus.example.org {
	reverse_proxy localhost:8081
}
```

- The default challenges need Let's Encrypt to reach port 80 or 443 on the
  proxy from the internet
- For a proxy only reachable inside your network, use the **DNS challenge**.
  It needs a DNS provider plugin, which the packages and the image leave out:
  build a binary or image with `xcaddy build --with github.com/caddy-dns/<provider>`
- Then clients need no extra trust at all

---
layout: section
level: 2
---

# nginx

<br>
<br>
<Link to="toc" title="Table of Contents"/>

---
hideInToc: true
level: 2
---

# Make certificates for nginx

nginx doesn't issue certificates. For the lab, make a CA and one certificate
for all three names:

```bash
mkdir -p certs && cd certs

# A CA for the lab, valid ten years. Clients trust ca.crt
openssl req -x509 -newkey ec -pkeyopt ec_paramgen_curve:prime256v1 -nodes \
  -days 3650 -subj "/CN=Example Lab CA" -keyout ca.key -out ca.crt

# The proxy's certificate, valid a year, for all three names
cat > server.ext <<'EOF'
subjectAltName = DNS:nexus.example.com, DNS:docker-proxy.example.com, DNS:docker-hosted.example.com
extendedKeyUsage = serverAuth
EOF
openssl req -newkey ec -pkeyopt ec_paramgen_curve:prime256v1 -nodes \
  -subj "/CN=nexus.example.com" -keyout server.key -out server.csr
openssl x509 -req -in server.csr -CA ca.crt -CAkey ca.key -CAcreateserial \
  -days 365 -extfile server.ext -out server.crt
```

Trust `ca.crt` on clients the same way as Caddy's `root.crt`. Renewing
`server.crt` each year is up to you: in production use certbot, or a CA
such as step-ca with an ACME client.

---
hideInToc: true
level: 2
---

# A minimal config, and what breaks

```nginx
server {
    listen 443 ssl;
    server_name docker-hosted.example.com;
    ssl_certificate     /etc/nginx/certs/server.crt;
    ssl_certificate_key /etc/nginx/certs/server.key;
    location / { proxy_pass http://127.0.0.1:10081; }
}
# ...and the same for nexus.example.com (8081) and docker-proxy.example.com (10080)
```

Every plain web page works. Then:

```text
$ docker push docker-hosted.example.com/ubuntu:24.04
unknown: failed to copy: unexpected status from PUT request to
https://docker-hosted.example.com/v2/ubuntu/blobs/uploads/...: 413 Request Entity Too Large

# /var/log/nginx/error.log
client intended to send too large body: 28965258 bytes

$ curl --silent --user admin:$newpass https://nexus.example.com/service/rest/v1/repositories | jq -r '.[0].url'
http://127.0.0.1:8081/repository/testy-hosted
```

The 1 MB default stops the first layer. Without `Host` and
`X-Forwarded-Proto`, Nexus hands out its internal URL.

---
hideInToc: true
level: 2
class: text-sm
---

# The settings binary stores need

```nginx
# Read inside http {}, so every server below gets these
client_max_body_size    0;      # no limit: image layers and artifacts are big
proxy_request_buffering off;    # stream uploads instead of spooling them first
proxy_buffering         off;    # stream downloads too
proxy_read_timeout      900s;   # the store may go quiet for minutes
proxy_send_timeout      900s;
proxy_set_header Host              $host;
proxy_set_header X-Real-IP         $remote_addr;
proxy_set_header X-Forwarded-For   $proxy_add_x_forwarded_for;
proxy_set_header X-Forwarded-Proto $scheme;

ssl_certificate     /etc/nginx/certs/server.crt;
ssl_certificate_key /etc/nginx/certs/server.key;

server {
    listen 80;
    server_name nexus.example.com docker-proxy.example.com docker-hosted.example.com;
    return 301 https://$host$request_uri;
}
server {
    listen 443 ssl;
    server_name nexus.example.com;
    location / { proxy_pass http://127.0.0.1:8081; }
}
# ...and the same for docker-proxy (10080) and docker-hosted (10081)
```

A `location` with any `proxy_set_header` of its own drops all of these
headers, not just the one it sets. Keep them at this level.

---
hideInToc: true
level: 2
---

# nginx on the host

Save the previous slide's config, all three `server` blocks, as `binary-stores.conf`:

```bash
sudo apt-get install -y nginx

$ nginx -v
nginx version: nginx/1.24.0 (Ubuntu)

sudo install -d -m 755 /etc/nginx/certs
sudo install -m 644 certs/server.crt /etc/nginx/certs/
sudo install -m 600 certs/server.key /etc/nginx/certs/

sudo cp binary-stores.conf /etc/nginx/sites-available/binary-stores
sudo ln -s /etc/nginx/sites-available/binary-stores /etc/nginx/sites-enabled/
sudo rm /etc/nginx/sites-enabled/default

sudo nginx -t
sudo systemctl reload nginx
```

Ubuntu's nginx reads `sites-enabled/` inside `http {}`, so the settings at the
top of the file apply to every site.

---
hideInToc: true
level: 2
---

# nginx in a container

The same file, with `127.0.0.1` changed to the Nexus container's name, as
the image's `conf.d/default.conf`:

```bash
mkdir -p nginx
sed 's/127\.0\.0\.1:/nexus:/' binary-stores.conf > nginx/default.conf

docker container run --detach \
  --name nginx \
  --network binary-stores \
  --publish 80:80 --publish 443:443 \
  --volume "$(pwd)/nginx/default.conf":/etc/nginx/conf.d/default.conf:ro \
  --volume "$(pwd)/certs":/etc/nginx/certs:ro \
  docker.io/library/nginx:1.30.7

docker container exec nginx nginx -t
```

After editing: `docker container exec nginx nginx -s reload`.
Run the "Try it" commands from the Caddy slides again: the upload gets `201`,
the URLs are `https://nexus.example.com/...`, and the push goes through.

---
hideInToc: true
level: 2
---

# Pulp and Harbor behind the proxy

Each store has to know the URL clients use, or it writes its internal one
into redirects and tokens.

**Pulp** (`pulp/pulp` on port 8080): in `settings.py`, change the URLs to
`https://pulp.example.com` and restart the container:

```python
CONTENT_ORIGIN='https://pulp.example.com'
TOKEN_SERVER='https://pulp.example.com/token/'
PYPI_API_HOSTNAME='https://pulp.example.com'
```

Then `reverse_proxy localhost:8080` in Caddy, or a `server` block in nginx.

**Harbor** runs its own nginx. In `harbor.yml`, set `external_url` and keep
the `https` block commented out, then run `./install.sh` again:

```yaml
external_url: https://harbor.example.com
```

Harbor's nginx then listens on plain HTTP behind yours.

<!--
Tested behind Caddy in a container: Pulp 3.123.1 (file upload and download, docker login, pull-through pull and push, with token auth) and Harbor 2.15.4 under amd64 emulation (docker login, push and pull; the token realm comes out as https://harbor.example.com/service/token).
-->

---
hideInToc: true
level: 2
---

# Which to use

**Caddy** when you run the proxy yourself and want it to just work:

- Defaults that suit binary stores, so there's little to get wrong
- Certificates issued and renewed for you, from its own CA or Let's Encrypt
- The same Caddyfile on the host and in a container

**nginx** when it's already your standard, or you need what it has built in:

- Caching (`proxy_cache`) in front of a store, rate limits, fine-grained tuning
- What vendors document, and what's inside Harbor and the Pulp image
- You'll set the size, buffering, timeout and header settings yourself, every time

Either way, the store needs to know its public URL, and the clients need to
trust the CA.
