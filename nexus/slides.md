---
layout: section
---

# Sonatype Nexus

<br>
<br>
<Link to="toc" title="Table of Contents"/>

---
layout: section
---

# Install self-hosted Nexus Repository

<br>
<br>
<Link to="toc" title="Table of Contents"/>

---
hideInToc: true
---

# Download and unpack the install archive

```bash
# https://help.sonatype.com/en/download.html
## x86_64
$ curl -o /tmp/nexus-linux-x86_64.tar.gz \
    -L https://download.sonatype.com/nexus/3/nexus-3.87.1-01-linux-x86_64.tar.gz
$ shasum -a 256 /tmp/nexus-linux-x86_64.tar.gz
9403cc4a78e11af09fc65e217e381dfcf435755dea31cdba9c947d6e1d439cd7  nexus-3.87.1-01-linux-x86_64.tar.gz

mkdir -p /opt/sonatype
tar -xvzf /tmp/nexus-linux-x86_64.tar.gz \
  --directory=/opt/sonatype \
  --keep-directory-symlink
```  

```bash
# https://help.sonatype.com/en/download.html
## aarch64
$ curl -o /tmp/nexus-linux-aarch_64.tar.gz \
    -L https://download.sonatype.com/nexus/3/nexus-3.87.1-01-linux-aarch_64.tar.gz
$ shasum -a 256 /tmp/nexus-linux-aarch_64.tar.gz
35847fc66895d3bd5cf8582b4d6c22161a00ce36924aed19f9b38107334b2ebb  /tmp/nexus-linux-aarch_64.tar.gz

sudo mkdir -p /opt/sonatype
sudo tar -xvzf /tmp/nexus-linux-aarch_64.tar.gz \
  --directory=/opt/sonatype \
  --keep-directory-symlink
```

```bash
# Make a symlink to the latest version as /opt/sonatype/nexus to make it easier to upgrade
# Pick the latest version explicitly
sudo ln -s "$(ls -d /opt/sonatype/nexus-* | sort -V | tail -n 1)" /opt/sonatype/nexus
```

---
hideInToc: true
---

# Create data directory

```bash
# Create user for nexus
sudo useradd \
  --system \
  --shell /bin/false \
  --comment "Nexus Repository Manager user" \
  --home-dir /var/lib/nexus \
  nexus

sudo mkdir -p /var/lib/nexus
sudo chown -R nexus:nexus /var/lib/nexus

# Define run_as_user as nexus
sudo tee /opt/sonatype/nexus/bin/nexus.rc > /dev/null <<'EOF'
run_as_user="nexus"
EOF
````

```bash
sudo chown -R nexus:nexus /opt/sonatype/sonatype-work
```

---
hideInToc: true
---

# Verify install starts up correctly

```bash
runuser -u nexus -- /bin/bash
cd /opt/sonatype/nexus/bin
./nexus run

# Visit http://localhost:8081 in a web browser
```

---
hideInToc: true
---

```bash
# https://help.sonatype.com/en/configuring-the-runtime-environment.html

mkdir -p /var/lib/nexus/javaprefs
chown -R nexus:nexus /var/lib/nexus/javaprefs
# Add the following to /opt/sonatype/nexus/bin/nexus.vmoptions
# This will avoid 'java.util.prefs' spamming the log 
-Djava.util.prefs.userRoot=/opt/sonatype/sonatype-work/nexus3/javaprefs
```

---
layout: section
---

# Docker install

<br>
<br>
<Link to="toc" title="Table of Contents"/>

---
hideInToc: true
---

```bash
docker container run -d \
  -p 8081:8081 \
  --name nexus \
  docker.io/sonatype/nexus3
```

---
hideInToc: true
---

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
---

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
---

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
---

```bash
  -H 'Content-Type: application/json' \
  
  http://localhost:8081/service/rest/v1/security/anonymous
```

---
hideInToc: true
---

# Official Image source

- https://github.com/sonatype/docker-nexus3/blob/main/Dockerfile
