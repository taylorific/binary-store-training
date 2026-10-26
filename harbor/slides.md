---
layout: section
---

# Harbor

<br>
<br>
<Link to="toc" title="Table of Contents"/>

---
hideInToc: true
level: 2
---

# What is Harbor?

An open source registry for container images and other OCI artifacts, and a
CNCF graduated project (Apache 2.0). You run it yourself, on your own hosts.

- **Only OCI:** container images, Helm charts, cosign signatures, anything
  `oras` can push. No apt, PyPI, npm or raw files
- **Projects** group repositories, each with its own members, quotas and policies
- **Proxy cache** projects pull through to Docker Hub, ghcr.io, Quay, ECR and others
- **Replication** pushes or pulls between Harbor and other registries
- **Scanning** with Trivy, and policies that block pulling vulnerable or unsigned images
- **Robot accounts** for CI, with scoped permissions and an expiry date
- Users in its own database, or from LDAP or OIDC
- A web UI, and a REST API at `/api/v2.0`

<br>

https://goharbor.io/docs/

---
layout: section
level: 2
---

# Install Harbor

<br>
<br>
<Link to="toc" title="Table of Contents"/>

---
hideInToc: true
level: 2
---

# What you need

- A Linux **x86_64** host: 2 CPUs, 4 GB of memory and 40 GB of disk at least
- Docker Engine with the Compose plugin
- A DNS name for the host. Not `localhost`: clients use the name in `harbor.yml`
- Ports 80 and 443 (or what you choose in `harbor.yml`)

The installer runs nine containers with Docker Compose: nginx, the portal, core,
the job service, the registry and registryctl, PostgreSQL, Valkey (a Redis fork)
and a log collector. Add `--with-trivy` for a tenth, the scanner.

Some limits on where it runs:

- Harbor publishes amd64 images only. An arm64 host can run them under emulation
  (I tested on an Apple silicon Mac), but that's slow and unsupported
- No Apple container instructions: Apple's `container` has no Compose
- For Kubernetes there's a Helm chart instead: https://github.com/goharbor/harbor-helm

---
hideInToc: true
level: 2
---

# Download and verify the installer

```bash
# https://github.com/goharbor/harbor/releases
# The online installer pulls the images from Docker Hub when you install.
# The offline installer (750 MB) carries them, for hosts without internet access.
curl -LO https://github.com/goharbor/harbor/releases/download/v2.15.4/harbor-online-installer-v2.15.4.tgz
curl -LO https://github.com/goharbor/harbor/releases/download/v2.15.4/harbor-online-installer-v2.15.4.tgz.sigstore.json

# Check it was built by Harbor's release workflow
cosign verify-blob \
  --bundle harbor-online-installer-v2.15.4.tgz.sigstore.json \
  --certificate-identity https://github.com/goharbor/harbor/.github/workflows/publish_release.yml@refs/tags/v2.15.4 \
  --certificate-oidc-issuer https://token.actions.githubusercontent.com \
  harbor-online-installer-v2.15.4.tgz
Verified OK

sudo tar -xzf harbor-online-installer-v2.15.4.tgz -C /opt
```

Without cosign, the release's `md5sum` file at least catches a bad download:
`curl -sL .../v2.15.4/md5sum | grep online | md5sum -c`

---
hideInToc: true
level: 2
---

# Configure harbor.yml

```bash
cd /opt/harbor
sudo cp harbor.yml.tmpl harbor.yml

# This example serves plain HTTP, so it comments out the https block.
# For HTTPS, set its certificate and private_key paths instead.
sudo sed -i \
  -e 's/^hostname: .*/hostname: harbor.example.com/' \
  -e '/^https:/,/^$/s/^/#/' \
  -e 's/^harbor_admin_password: .*/harbor_admin_password: superseekret/' \
  -e "s/^  password: root123/  password: $(openssl rand -hex 16)/" \
  harbor.yml

$ grep -E '^(hostname|harbor_admin_password|data_volume)' harbor.yml
hostname: harbor.example.com
harbor_admin_password: superseekret
data_volume: /data
```

- `harbor_admin_password` only applies the first time. Change it later in the UI or the API
- `root123` is the database password. The template ships with it, so change it
- `data_volume` holds the database, the images and the job logs. Back it up

---
hideInToc: true
level: 2
---

# Install and start

```bash
cd /opt/harbor
sudo ./install.sh --with-trivy

✔ ----Harbor has been installed and started successfully.----

$ curl --silent http://harbor.example.com/api/v2.0/health | jq --raw-output .status
healthy
```

- `install.sh` runs `prepare`, which writes `docker-compose.yml` and the
  `common/config` directory from `harbor.yml`, then runs `docker compose up -d`
- Every service has `restart: always`, so Docker brings Harbor back after a
  reboot. There's no systemd unit to write
- After editing `harbor.yml`, run `install.sh` again. It keeps what's in `data_volume`
- To stop and start: `sudo docker compose down` and `sudo docker compose up -d`
  in `/opt/harbor`

Then log in to the web UI at http://harbor.example.com as `admin`.

---
hideInToc: true
level: 2
---

# Let Docker clients use plain HTTP

Docker only talks plain HTTP to registries it's told are insecure. On each
client (skip this if Harbor has a certificate):

```bash
sudo tee /etc/docker/daemon.json <<'EOF'
{
  "insecure-registries": ["harbor.example.com"]
}
EOF
sudo systemctl restart docker
```

Harbor issues its own tokens at `http://<hostname>/service/token`. That's why
`hostname` must be the name clients use.

The API calls in the next section use these:

```bash
HARBOR_HOST=harbor.example.com
HARBOR=http://$HARBOR_HOST/api/v2.0
newpass="superseekret"
```

---
layout: section
level: 2
---

# The boxcutter_sonatype cookbook, in Harbor

<br>
<br>
<Link to="toc" title="Table of Contents"/>

---
hideInToc: true
level: 2
class: text-xs
---

# What carries over

Harbor covers only the cookbook's Docker setup:

| Cookbook (Nexus) | Harbor |
| --- | --- |
| Realms (`DockerToken`) | Nothing to do: token auth is built in |
| Blob stores | One storage backend for everything, set in `harbor.yml` (`storage_service`): the filesystem, S3, Azure, GCS or Swift |
| `docker-proxy` repository on port 10080 | A registry endpoint plus a proxy cache project, on Harbor's own port |
| A Docker hosted repository | A project |
| Raw, apt, PyPI and npm repositories | No equivalent |
| `engineering-read-only` role | Project member roles: Project Admin, Maintainer, Developer, Guest, Limited Guest |
| `chef` user with `nx-admin` | A user with the system administrator flag, or a robot account |

---
hideInToc: true
level: 2
---

# Docker Hub proxy cache

First a registry endpoint for Docker Hub, then a project that caches through it:

```bash
curl --fail --user admin:$newpass \
  -X POST -H 'Content-Type: application/json' \
  -d '{"name": "docker-hub", "type": "docker-hub", "url": "https://hub.docker.com"}' \
  $HARBOR/registries

registry_id=$(curl --silent --fail --user admin:$newpass \
  "$HARBOR/registries?q=name%3Ddocker-hub" | jq '.[0].id')

curl --fail --user admin:$newpass \
  -X POST -H 'Content-Type: application/json' \
  -d "{
    \"project_name\": \"docker-proxy\",
    \"registry_id\": ${registry_id},
    \"metadata\": {\"public\": \"true\"}
  }" \
  $HARBOR/projects
```

`public` lets anyone pull without logging in, as Nexus's anonymous access does.
A proxy cache project can't hold images you push.

---
hideInToc: true
level: 2
---

# Pull through the cache

```bash
# The project name goes in front. Official images keep their library/ prefix.
docker pull harbor.example.com/docker-proxy/library/alpine:3.20
docker pull harbor.example.com/docker-proxy/library/busybox:1.36
```

```bash
# Harbor records what it cached in the background, so give it a few seconds
$ curl --silent --user admin:$newpass $HARBOR/projects/docker-proxy/repositories | \
    jq --raw-output '.[].name'
docker-proxy/library/busybox
docker-proxy/library/alpine
```

Nexus serves Docker Hub's paths as they are (`localhost:10080/library/alpine`),
from its own port. Harbor serves every project on one host and port, with
the project name added to the path.

---
hideInToc: true
level: 2
---

# A hosted project and a read-only user

```bash
curl --fail --user admin:$newpass \
  -X POST -H 'Content-Type: application/json' \
  -d '{"project_name": "testy", "metadata": {"public": "false"}}' \
  $HARBOR/projects

curl --fail --user admin:$newpass \
  -X POST -H 'Content-Type: application/json' \
  -d '{
    "username": "reader",
    "email": "reader@nowhere.com",
    "realname": "Read Only",
    "password": "Readseekret63"
  }' \
  $HARBOR/users

# role_id: 1 Project Admin, 2 Developer, 3 Guest, 4 Maintainer, 5 Limited Guest
curl --fail --user admin:$newpass \
  -X POST -H 'Content-Type: application/json' \
  -d '{"role_id": 3, "member_user": {"username": "reader"}}' \
  $HARBOR/projects/testy/members
```

A Guest can pull from `testy` but not push.

---
hideInToc: true
level: 2
---

# An administrator user

The cookbook gives its `chef` user the `nx-admin` role. In Harbor, set the
system administrator flag:

```bash
curl --fail --user admin:$newpass \
  -X POST -H 'Content-Type: application/json' \
  -d '{
    "username": "chef",
    "email": "nobody@nowhere.com",
    "realname": "Chef User",
    "password": "Superseekret1"
  }' \
  $HARBOR/users

chef_id=$(curl --silent --fail --user admin:$newpass \
  "$HARBOR/users/search?username=chef" | jq '.[0].user_id')

curl --fail --user admin:$newpass \
  -X PUT -H 'Content-Type: application/json' \
  -d '{"sysadmin_flag": true}' \
  $HARBOR/users/${chef_id}/sysadmin
```

Passwords need 8 or more characters with an uppercase letter, a lowercase
letter and a digit. For automation, a robot account (next) is usually the better choice.

---
hideInToc: true
level: 2
---

# A robot account for CI

```bash
curl --silent --fail --user admin:$newpass \
  -X POST -H 'Content-Type: application/json' \
  -d '{
    "name": "ci",
    "duration": 90,
    "level": "project",
    "permissions": [{
      "kind": "project",
      "namespace": "testy",
      "access": [
        {"resource": "repository", "action": "pull"},
        {"resource": "repository", "action": "push"}
      ]
    }]
  }' \
  $HARBOR/robots > robot.json

$ jq --raw-output .name robot.json
robot$testy+ci
```

`duration` is in days (`-1` never expires). The response is the only time
Harbor shows the secret, so keep it somewhere safe.

---
hideInToc: true
level: 2
---

# Push and pull

```bash
docker tag harbor.example.com/docker-proxy/library/alpine:3.20 \
  harbor.example.com/testy/alpine:3.20

# The reader can't push
echo Readseekret63 | docker login -u reader --password-stdin harbor.example.com
$ docker push harbor.example.com/testy/alpine:3.20
unauthorized: unauthorized to access repository: testy/alpine, action: push

# The robot can
jq --raw-output .secret robot.json | \
  docker login -u 'robot$testy+ci' --password-stdin harbor.example.com
docker push harbor.example.com/testy/alpine:3.20

# The reader can pull. Without logging in, nobody can: testy isn't public.
echo Readseekret63 | docker login -u reader --password-stdin harbor.example.com
docker pull harbor.example.com/testy/alpine:3.20
```

Quote `robot$testy+ci` in single quotes, or the shell expands `$testy`.

---
hideInToc: true
level: 2
---

# Helm charts and other OCI artifacts

Harbor stores Helm charts as OCI artifacts, next to the images:

```bash
helm create demo && helm package demo

echo Superseekret1 | \
  helm registry login harbor.example.com --plain-http -u chef --password-stdin
helm push demo-0.1.0.tgz oci://harbor.example.com/testy --plain-http
Pushed: harbor.example.com/testy/demo:0.1.0

$ curl --silent --user admin:$newpass \
    $HARBOR/projects/testy/repositories/demo/artifacts | jq --raw-output '.[].type'
CHART
```

`--plain-http` is only for a Harbor without HTTPS. Nexus keeps Helm charts in
a separate `helm` repository format.

---
hideInToc: true
level: 2
---

# Scan images with Trivy

With `install.sh --with-trivy`, scan an image on request:

```bash
curl --fail --user admin:$newpass -X POST \
  $HARBOR/projects/testy/repositories/alpine/artifacts/3.20/scan

$ curl --silent --user admin:$newpass \
    "$HARBOR/projects/testy/repositories/alpine/artifacts/3.20?with_scan_overview=true" | \
    jq -c '[.scan_overview[] | .scan_status, .summary.total]'
["Success",2]
```

- Set `auto_scan` in a project's metadata to scan every image pushed to it
- Set `prevent_vul` and `severity` to stop clients pulling images with
  vulnerabilities at or above that severity
- Nexus Community Edition doesn't scan. Sonatype sells that separately

---
hideInToc: true
level: 2
class: text-xs
---

# Harbor and Nexus for containers

| | Harbor | Nexus |
| --- | --- | --- |
| **Formats** | OCI only: images, Helm charts, signatures | Docker and about 20 other formats |
| **Ports** | All projects on one host and port | One port (or a path, with a reverse proxy) per Docker repository |
| **Proxying Docker Hub** | Proxy cache project, path gets the project name | Proxy repository, Docker Hub's paths unchanged |
| **Combine hosted and proxy** | No: pull from each project by name | Docker group repository |
| **Accounts** | Users, LDAP or OIDC; robot accounts with an expiry date | Users, LDAP; user tokens (Pro) |
| **Scanning and signing** | Trivy built in; can require signed images | Not in Community Edition |
| **Replication** | Built in, to and from other registries | Pro only |
| **Runs as** | Nine or ten containers with Docker Compose, amd64 only | One JVM |
| **Limits** | None (Apache 2.0) | Community Edition limits on components and requests |

If you only need containers and Helm charts, Harbor does more for free. If
you also need apt, PyPI or npm, you'll run Nexus, Artifactory or Pulp anyway.
