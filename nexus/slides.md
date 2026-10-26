---
layout: section
---

# Sonatype Nexus

<br>
<br>
<Link to="toc" title="Table of Contents"/>

---
layout: section
level: 2
---

# Install self-hosted Nexus Repository

<br>
<br>
<Link to="toc" title="Table of Contents"/>

---
hideInToc: true
level: 2
---

# Download and check the install archive

The archive bundles its own Java (Temurin 21), so there's nothing else to install.

```bash
# https://help.sonatype.com/en/download.html
## x86_64
curl -o /tmp/nexus-3.97.2-02-linux-x86_64.tar.gz \
  -L https://download.sonatype.com/nexus/3/nexus-3.97.2-02-linux-x86_64.tar.gz
echo "cda1837b8fde687989af423dabf00be471695679e736e3c527e35b6fcd88c2f5  /tmp/nexus-3.97.2-02-linux-x86_64.tar.gz" | \
  sha256sum --check
```

```bash
## aarch64
curl -o /tmp/nexus-3.97.2-02-linux-aarch_64.tar.gz \
  -L https://download.sonatype.com/nexus/3/nexus-3.97.2-02-linux-aarch_64.tar.gz
echo "5d7bf1a6c5beb855060052272f4aec6bc661fe00f88abd1c657d8971d66022b5  /tmp/nexus-3.97.2-02-linux-aarch_64.tar.gz" | \
  sha256sum --check
```

Sonatype publishes each checksum next to the archive: add `.sha256` to the URL.

---
hideInToc: true
level: 2
---

# Unpack into /opt/sonatype

```bash
sudo mkdir -p /opt/sonatype
sudo tar -xzf /tmp/nexus-3.97.2-02-linux-$(uname -m | sed 's/aarch64/aarch_64/').tar.gz \
  --directory=/opt/sonatype \
  --exclude='sonatype-work*' \
  --keep-directory-symlink

# Point /opt/sonatype/nexus at this version, so an upgrade is a new symlink
sudo ln -sfn /opt/sonatype/nexus-3.97.2-02 /opt/sonatype/nexus

$ readlink /opt/sonatype/nexus
/opt/sonatype/nexus-3.97.2-02
```

- The archive also holds an empty `sonatype-work` skeleton. Skip it: the data
  directory goes in `/var/lib/nexus/nexus-data`, outside the versioned install,
  so an upgrade doesn't touch it
- The program files stay owned by root. Only the data directory belongs to `nexus`

---
hideInToc: true
level: 2
---

# Create the nexus user and data directory

```bash
sudo useradd \
  --system \
  --shell /usr/sbin/nologin \
  --comment "Nexus Repository Manager user" \
  --home-dir /var/lib/nexus \
  --create-home \
  nexus

sudo install -d -o nexus -g nexus \
  /var/lib/nexus/nexus-data \
  /var/lib/nexus/nexus-data/etc

# The start script switches to this user. Since 3.80 the archive
# doesn't ship a nexus.rc, so create it
echo 'run_as_user="nexus"' | sudo tee /opt/sonatype/nexus/bin/nexus.rc
```

Create `nexus-data` first, as its own argument. `install -d` makes any
missing parent directories owned by root, and Nexus then can't create `tmp`
in it (`AccessDeniedException: /var/lib/nexus/nexus-data/tmp`).

---
hideInToc: true
level: 2
---

# Move the data directory

`bin/nexus.vmoptions` points the data, log and temporary directories at
`../sonatype-work/nexus3`. Point them at `/var/lib/nexus/nexus-data`:

```bash
# https://help.sonatype.com/en/configuring-the-runtime-environment.html
sudo sed -i \
  -e 's|\.\./sonatype-work/nexus3|/var/lib/nexus/nexus-data|' \
  /opt/sonatype/nexus/bin/nexus.vmoptions

# Keep java.util.prefs from writing warnings to the log
echo '-Djava.util.prefs.userRoot=/var/lib/nexus/nexus-data/javaprefs' | \
  sudo tee -a /opt/sonatype/nexus/bin/nexus.vmoptions

$ grep nexus-data /opt/sonatype/nexus/bin/nexus.vmoptions
-XX:LogFile=/var/lib/nexus/nexus-data/log/jvm.log
-Dkaraf.data=/var/lib/nexus/nexus-data
-Dkaraf.log=/var/lib/nexus/nexus-data/log
-Djava.io.tmpdir=/var/lib/nexus/nexus-data/tmp
-Djava.util.prefs.userRoot=/var/lib/nexus/nexus-data/javaprefs
```

`nexus.vmoptions` is in the versioned directory, so redo this after each upgrade.

---
hideInToc: true
level: 2
---

# Start with no repositories

A new Nexus creates `maven-central`, `nuget-group` and others. To start with
only the repositories you create, set this before the first start:

```bash
sudo -u nexus tee /var/lib/nexus/nexus-data/etc/nexus.properties <<'EOF'
nexus.skipDefaultRepositories=true
EOF
```

---
hideInToc: true
level: 2
---

# Run Nexus as a systemd service

```bash
sudo tee /etc/systemd/system/nexus.service <<'EOF'
[Unit]
Description=Nexus Repository Manager
After=network.target

[Service]
Type=forking
LimitNOFILE=65536
ExecStart=/opt/sonatype/nexus/bin/nexus start
ExecStop=/opt/sonatype/nexus/bin/nexus stop
User=nexus
Restart=on-abort
TimeoutSec=600

[Install]
WantedBy=multi-user.target
EOF

sudo systemctl daemon-reload
sudo systemctl enable --now nexus

# The first start takes a minute or two
until curl --silent --fail http://localhost:8081/service/rest/v1/status/writable; do sleep 5; done
```

To watch it start in the foreground instead: `sudo -u nexus /opt/sonatype/nexus/bin/nexus run`

---
hideInToc: true
level: 2
---

# Accept the EULA

Nexus Community Edition won't serve anything until the EULA is accepted.
Do this before changing the password, as both use the initial one:

```bash
# https://help.sonatype.com/en/eula-rest-api.html
oldpass=$(sudo cat /var/lib/nexus/nexus-data/admin.password)

disclaimer=$(curl --silent --show-error \
  --user admin:$oldpass \
  http://localhost:8081/service/rest/v1/system/eula | \
    jq --raw-output .disclaimer)

curl --include --fail \
  --user admin:$oldpass \
  -X POST \
  -H 'Content-Type: application/json' \
  -d "{\"accepted\": true, \"disclaimer\": \"${disclaimer}\"}" \
  http://localhost:8081/service/rest/v1/system/eula
```

---
hideInToc: true
level: 2
---

# Choose a password for the admin user

```bash
newpass="superseekret"

curl --include --fail \
  --user admin:$oldpass \
  -X PUT \
  -H 'Content-Type: text/plain' \
  --data "${newpass}" \
  http://localhost:8081/service/rest/v1/security/users/admin/change-password
```

Nexus deletes `admin.password` once the password changes. Nexus 3.71
re-created it on restart, holding the stale initial password
([nexus-public#458](https://github.com/sonatype/nexus-public/issues/458)).
3.97.2 doesn't, so its absence means the password has been set.

---
hideInToc: true
level: 2
---

# Configure anonymous access

```bash
curl --include --fail \
  --user admin:$newpass \
  -X PUT \
  -H 'Content-Type: application/json' \
  -d '{"enabled": true, "userId": "anonymous", "realmName": "NexusAuthorizingRealm"}' \
  http://localhost:8081/service/rest/v1/security/anonymous
```

---
layout: section
level: 2
---

# Docker install

<br>
<br>
<Link to="toc" title="Table of Contents"/>

---
hideInToc: true
level: 2
---

# Run the container

```bash
docker container run -d \
  -p 8081:8081 \
  -p 10080:10080 \
  -e INSTALL4J_ADD_VM_PARAMS="-Djava.util.prefs.userRoot=/nexus-data/javaprefs -Dnexus.skipDefaultRepositories=true" \
  --name nexus \
  docker.io/sonatype/nexus3:3.97.2
```

- `10080` is the port the `docker-proxy` repository listens on (see the cookbook slides)
- `nexus.skipDefaultRepositories=true` starts Nexus with no repositories, so only
  the ones you create exist. It only matters on the first start.
- `INSTALL4J_ADD_VM_PARAMS` replaces the image's default, so keep its `javaprefs` setting

---
hideInToc: true
level: 2
---

# Apple container install (macOS)

```bash
container system start   # once per boot

# Nexus defaults to a 2.7 GB Java heap, and containers get 1 GB unless you ask
container run --detach \
  --cpus 4 --memory 4g \
  --publish 8081:8081 \
  --publish 10080:10080 \
  --env INSTALL4J_ADD_VM_PARAMS="-Djava.util.prefs.userRoot=/nexus-data/javaprefs -Dnexus.skipDefaultRepositories=true" \
  --name nexus \
  docker.io/sonatype/nexus3:3.97.2
```

The REST calls on the following slides are the same. Read the initial
password with `container exec` in place of `docker exec -t`:

```bash
oldpass=$(container exec nexus cat /nexus-data/admin.password)
```

---
hideInToc: true
level: 2
---

# Accept EULA

https://help.sonatype.com/en/eula-rest-api.html

```bash
disclaimer=$(curl \
  --silent --show-error \
  -u admin:$(docker exec -t nexus cat /nexus-data/admin.password) \
  -X GET 'http://localhost:8081/service/rest/v1/system/eula' | \
    jq --raw-output .disclaimer)

curl --include --fail \
  -X POST \
  -u admin:$(docker exec -t nexus cat /nexus-data/admin.password) \
  -H 'Content-Type: application/json' \
  -d "{\"accepted\": true, \"disclaimer\": \"${disclaimer}\"}" \
  http://localhost:8081/service/rest/v1/system/eula
```

---
hideInToc: true
level: 2
---

# Choose a password for the admin user

```bash
newpass="superseekret"
oldpass=$(docker exec -t nexus cat /nexus-data/admin.password)

curl --include --fail \
  --user admin:$oldpass \
  -X PUT \
  -H 'Content-Type: text/plain' \
  --data "${newpass}" \
  http://localhost:8081/service/rest/v1/security/users/admin/change-password
```

---
hideInToc: true
level: 2
---

# Configure anonymous access

```bash
curl --include --fail \
  --user admin:$newpass \
  -X PUT \
  -H 'Content-Type: application/json' \
  -d '{"enabled": true, "userId": "anonymous"}' \
  http://localhost:8081/service/rest/v1/security/anonymous
```

---
hideInToc: true
level: 2
---

# Official Image source

- https://github.com/sonatype/docker-nexus3/blob/main/Dockerfile

---
layout: section
level: 2
---

# The boxcutter_sonatype cookbook, in Nexus

<br>
<br>
<Link to="toc" title="Table of Contents"/>

---
hideInToc: true
level: 2
class: text-xs
---

# What the cookbook configures

https://github.com/boxcutter/boxcutter-chef-cookbooks/tree/main/cookbooks/boxcutter_sonatype

Sonatype doesn't ship a CLI for configuring Nexus. The cookbook calls the REST API,
and so do these slides, with `curl`. The full API is at
`http://localhost:8081/service/rest/swagger.json` (280 paths in 3.97.2).

| Cookbook setting | REST API |
| --- | --- |
| `nexus.skipDefaultRepositories` | `nexus.properties` (or a `-D` system property) |
| Realms: `NexusAuthenticatingRealm`, `DockerToken` | `PUT /v1/security/realms/active` |
| `blobstores` | `POST /v1/blobstores/file` |
| `repositories` | `POST /v1/repositories/<format>/<type>` |
| `roles` | `POST /v1/security/roles` |
| `users` | `POST /v1/security/users` |

```bash
NEXUS=http://localhost:8081/service/rest
newpass="superseekret"
```

---
hideInToc: true
level: 2
---

# Skip the default repositories

A new Nexus creates `maven-central`, `maven-releases`, `nuget-group` and others.
The cookbook turns that off so that only the repositories it manages exist.
It only matters before the first start.

```bash
# Tarball install: the data directory's etc/nexus.properties
sudo -u nexus tee /var/lib/nexus/nexus-data/etc/nexus.properties <<'EOF'
nexus.skipDefaultRepositories=true
EOF
```

```bash
# Container: as a system property (see "Run the container")
-e INSTALL4J_ADD_VM_PARAMS="-Djava.util.prefs.userRoot=/nexus-data/javaprefs -Dnexus.skipDefaultRepositories=true"
```

```bash
curl --silent --fail --user admin:$newpass $NEXUS/v1/repositories | jq -c '[.[].name]'
[]
```

---
hideInToc: true
level: 2
---

# Realms

Realms are the ways Nexus can authenticate a request. The cookbook turns on
the Docker Bearer Token realm, without which `docker login` fails with
`401 Unauthorized`. Add `NpmToken` too if people will use `npm login`.

```bash
curl --silent --fail --user admin:$newpass $NEXUS/v1/security/realms/available \
  | jq -c '[.[].id]'

curl --include --fail \
  --user admin:$newpass \
  -X PUT \
  -H 'Content-Type: application/json' \
  -d '["NexusAuthenticatingRealm", "DockerToken", "NpmToken"]' \
  $NEXUS/v1/security/realms/active
```

The list you `PUT` replaces the active realms. Leave out
`NexusAuthenticatingRealm` and the local users, admin included, can't log in.

---
hideInToc: true
level: 2
---

# Blob stores

Where repositories keep their files. Nexus starts with one file blob store,
`default`. Each repository names its blob store when you create it. The API
won't change it later ("You cannot change a repository's blob store through the API").

```bash
curl --silent --fail --user admin:$newpass $NEXUS/v1/blobstores \
  | jq -c '[.[] | {name, type}]'
[{"name":"default","type":"File"}]

# A second file blob store, for the proxy caches. The path is relative to the
# data directory's blobs/ unless it's absolute.
curl --include --fail \
  --user admin:$newpass \
  -X POST \
  -H 'Content-Type: application/json' \
  -d '{"name": "proxies", "path": "proxies"}' \
  $NEXUS/v1/blobstores/file
```

S3, Azure and Google Cloud blob stores use `POST /v1/blobstores/s3` (or `azure`, `google`).

---
hideInToc: true
level: 2
---

# Raw hosted

```bash
curl --include --fail \
  --user admin:$newpass \
  -X POST \
  -H 'Content-Type: application/json' \
  -d '{
    "name": "testy-hosted",
    "online": true,
    "storage": {"blobStoreName": "default", "strictContentTypeValidation": false, "writePolicy": "ALLOW"}
  }' \
  $NEXUS/v1/repositories/raw/hosted
```

```bash
# Upload with a PUT, download anonymously
echo "hello from nexus" > hello.txt
curl --fail --user admin:$newpass --upload-file hello.txt \
  http://localhost:8081/repository/testy-hosted/hello.txt
curl http://localhost:8081/repository/testy-hosted/hello.txt
```

`writePolicy`: `ALLOW` (overwrite), `ALLOW_ONCE` (no redeploy) or `DENY` (read-only)

---
hideInToc: true
level: 2
---

# Raw proxy

Fetches any path from the upstream the first time a client asks for it

```bash
curl --include --fail \
  --user admin:$newpass \
  -X POST \
  -H 'Content-Type: application/json' \
  -d '{
    "name": "ubuntu-releases-proxy",
    "online": true,
    "storage": {"blobStoreName": "proxies", "strictContentTypeValidation": false},
    "proxy": {"remoteUrl": "https://releases.ubuntu.com", "contentMaxAge": 1440, "metadataMaxAge": 1440},
    "negativeCache": {"enabled": true, "timeToLive": 1440},
    "httpClient": {"blocked": false, "autoBlock": true}
  }' \
  $NEXUS/v1/repositories/raw/proxy
```

```bash
curl http://localhost:8081/repository/ubuntu-releases-proxy/noble/SHA256SUMS
```

The cookbook's `ubuntu-cdimage-proxy`, `cinc-omnitruck-proxy` and
`onepassword-proxy` are the same with a different `remoteUrl`.

---
hideInToc: true
level: 2
---

# APT proxy

```bash
curl --include --fail \
  --user admin:$newpass \
  -X POST \
  -H 'Content-Type: application/json' \
  -d '{
    "name": "gazebo-apt-proxy",
    "online": true,
    "storage": {"blobStoreName": "proxies", "strictContentTypeValidation": false},
    "proxy": {
      "remoteUrl": "http://packages.osrfoundation.org/gazebo/ubuntu-stable",
      "contentMaxAge": 1440, "metadataMaxAge": 1440
    },
    "negativeCache": {"enabled": true, "timeToLive": 1440},
    "httpClient": {"blocked": false, "autoBlock": true},
    "apt": {"distribution": "jammy", "flat": false}
  }' \
  $NEXUS/v1/repositories/apt/proxy
```

---
hideInToc: true
level: 2
---

# APT client

```bash
# On an Ubuntu 22.04 client. Nexus passes the upstream InRelease through,
# so the upstream key still verifies it: no trusted=yes.
sudo curl -fsSL https://packages.osrfoundation.org/gazebo.gpg \
  -o /usr/share/keyrings/pkgs-osrf-archive-keyring.gpg
echo "deb [signed-by=/usr/share/keyrings/pkgs-osrf-archive-keyring.gpg] \
http://localhost:8081/repository/gazebo-apt-proxy jammy main" \
  | sudo tee /etc/apt/sources.list.d/gazebo.list
sudo apt-get update && sudo apt-get install sdformat14-sdf
```

Pulp, by contrast, writes its own `Release` file, so clients need Pulp's
signing key or `trusted=yes`.

---
hideInToc: true
level: 2
---

# Docker proxy

```bash
curl --include --fail \
  --user admin:$newpass \
  -X POST \
  -H 'Content-Type: application/json' \
  -d '{
    "name": "docker-proxy",
    "online": true,
    "storage": {"blobStoreName": "proxies", "strictContentTypeValidation": true},
    "proxy": {"remoteUrl": "https://registry-1.docker.io", "contentMaxAge": 1440, "metadataMaxAge": 1440},
    "negativeCache": {"enabled": true, "timeToLive": 1440},
    "httpClient": {"blocked": false, "autoBlock": true},
    "docker": {"v1Enabled": false, "forceBasicAuth": false, "httpPort": 10080},
    "dockerProxy": {"indexType": "HUB"}
  }' \
  $NEXUS/v1/repositories/docker/proxy
```

```bash
# Anonymous, including images nobody has pulled yet
docker pull localhost:10080/library/alpine:3.22
```

Each Docker repository gets its own port (`httpPort`), which the container
must publish. `forceBasicAuth: false` is what lets anonymous clients pull.

---
hideInToc: true
level: 2
---

# PyPI proxy and hosted

```bash
curl --include --fail --user admin:$newpass -X POST -H 'Content-Type: application/json' \
  -d '{
    "name": "pypi-proxy", "online": true,
    "storage": {"blobStoreName": "proxies", "strictContentTypeValidation": true},
    "proxy": {"remoteUrl": "https://pypi.org/", "contentMaxAge": 1440, "metadataMaxAge": 1440},
    "negativeCache": {"enabled": true, "timeToLive": 1440},
    "httpClient": {"blocked": false, "autoBlock": true}
  }' \
  $NEXUS/v1/repositories/pypi/proxy

curl --include --fail --user admin:$newpass -X POST -H 'Content-Type: application/json' \
  -d '{
    "name": "pypi-hosted", "online": true,
    "storage": {"blobStoreName": "default", "strictContentTypeValidation": true, "writePolicy": "ALLOW_ONCE"}
  }' \
  $NEXUS/v1/repositories/pypi/hosted
```

---
hideInToc: true
level: 2
---

# PyPI group and clients

```bash
# A group serves both from one URL, members searched in order
curl --include --fail --user admin:$newpass -X POST -H 'Content-Type: application/json' \
  -d '{
    "name": "pypi-group", "online": true,
    "storage": {"blobStoreName": "default", "strictContentTypeValidation": true},
    "group": {"memberNames": ["pypi-hosted", "pypi-proxy"]}
  }' \
  $NEXUS/v1/repositories/pypi/group
```

```bash
# Upload to the hosted repository
twine upload \
  --repository-url http://localhost:8081/repository/pypi-hosted/ \
  --username admin --password "${newpass}" \
  dist/*

# Install from the group: your package and PyPI's, one index
pip install --index-url http://localhost:8081/repository/pypi-group/simple/ \
  hello-boxcutter requests
```

Pulp has no group repository; with Pulp, pip needs `--extra-index-url` for
the second index.

---
hideInToc: true
level: 2
---

# npm proxy and hosted

```bash
curl --include --fail --user admin:$newpass -X POST -H 'Content-Type: application/json' \
  -d '{
    "name": "npm-proxy", "online": true,
    "storage": {"blobStoreName": "proxies", "strictContentTypeValidation": true},
    "proxy": {"remoteUrl": "https://registry.npmjs.org", "contentMaxAge": 1440, "metadataMaxAge": 1440},
    "negativeCache": {"enabled": true, "timeToLive": 1440},
    "httpClient": {"blocked": false, "autoBlock": true}
  }' \
  $NEXUS/v1/repositories/npm/proxy

curl --include --fail --user admin:$newpass -X POST -H 'Content-Type: application/json' \
  -d '{
    "name": "npm-hosted", "online": true,
    "storage": {"blobStoreName": "default", "strictContentTypeValidation": true, "writePolicy": "ALLOW_ONCE"}
  }' \
  $NEXUS/v1/repositories/npm/hosted
```

---
hideInToc: true
level: 2
---

# npm group and clients

```bash
curl --include --fail --user admin:$newpass -X POST -H 'Content-Type: application/json' \
  -d '{
    "name": "npm-group", "online": true,
    "storage": {"blobStoreName": "default", "strictContentTypeValidation": true},
    "group": {"memberNames": ["npm-hosted", "npm-proxy"]}
  }' \
  $NEXUS/v1/repositories/npm/group
```

```bash
# Publish to the hosted repository
echo "//localhost:8081/repository/npm-hosted/:_auth=$(printf 'admin:%s' "$newpass" | base64)" >> ~/.npmrc
npm publish --registry http://localhost:8081/repository/npm-hosted/

# Install from the group: your packages and npmjs.org's
npm install --registry http://localhost:8081/repository/npm-group/ @boxcutter/hello left-pad
```

---
hideInToc: true
level: 2
---

# Roles

Privileges are named `nx-repository-view-<format>-<repository>-<action>`,
with `*` as a wildcard

```bash
curl --include --fail \
  --user admin:$newpass \
  -X POST \
  -H 'Content-Type: application/json' \
  -d '{
    "id": "engineering-read-only",
    "name": "engineering-read-only",
    "description": "Read-only access to engineering repositories",
    "privileges": [
      "nx-healthcheck-read",
      "nx-search-read",
      "nx-repository-view-*-*-read",
      "nx-repository-view-*-*-browse"
    ],
    "roles": []
  }' \
  $NEXUS/v1/security/roles
```

`nx-admin` and `nx-anonymous` are built in

---
hideInToc: true
level: 2
---

# Users

```bash
curl --include --fail --user admin:$newpass -X POST -H 'Content-Type: application/json' \
  -d '{
    "userId": "chef", "firstName": "Chef", "lastName": "User",
    "emailAddress": "nobody@nowhere.com", "password": "superseekret",
    "status": "active", "roles": ["nx-admin"]
  }' \
  $NEXUS/v1/security/users

curl --include --fail --user admin:$newpass -X POST -H 'Content-Type: application/json' \
  -d '{
    "userId": "reader", "firstName": "Read", "lastName": "Only",
    "emailAddress": "reader@nowhere.com", "password": "Readseekret63",
    "status": "active", "roles": ["engineering-read-only"]
  }' \
  $NEXUS/v1/security/users
```

```bash
curl -o /dev/null -w "%{http_code}\n" --user reader:Readseekret63 \
  --upload-file hello.txt http://localhost:8081/repository/testy-hosted/x.txt
403
```

---
hideInToc: true
level: 2
---

# Check the result

```bash
curl --silent --fail --user admin:$newpass $NEXUS/v1/repositories \
  | jq -r '.[] | "\(.name) \(.format) \(.type)"'
testy-hosted raw hosted
ubuntu-releases-proxy raw proxy
gazebo-apt-proxy apt proxy
docker-proxy docker proxy
pypi-proxy pypi proxy
pypi-hosted pypi hosted
pypi-group pypi group
npm-proxy npm proxy
npm-hosted npm hosted
npm-group npm group
```

`POST` fails if the repository exists. To change one, `PUT` the same body to
`/v1/repositories/<format>/<type>/<name>`. The cookbook checks what exists
first, which is what makes it safe to run again.
