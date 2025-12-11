---
layout: section
---

# Sonatype Nexus

<br>
<br>
<Link to="toc" title="Table of Contents"/>

---
hideInToc: true
---

```
tar -tzf archive.tar.gz
```

```
default['nexus_repository_manager']['sonatype']['path'] = '/opt/sonatype'
default['nexus_repository_manager']['nexus_data']['path'] = '/nexus-data'
# sonatype-work
default['nexus_repository_manager']['sonatype_work']['path'] = default['nexus_repository_manager']['sonatype']['path'] + '/sonatype-work'
# nexus home
default['nexus_repository_manager']['nexus_home']['path'] = default['nexus_repository_manager']['sonatype']['path'] + '/nexus'


# Create user for ROS
sudo useradd \
  --system \
  --shell /bin/false \
  --comment "Nexus Repository Manager user" \<img width="1325" height="982" alt="2025-12-10_19-22-21" src="https://github.com/user-attachments/assets/15609a23-1134-4de9-97c4-f254676c1bb4" />

  --home-dir /opt/sonatype/nexus \
  nexus

sudo groupadd --system nexus

echo "nexus ALL=(ALL:ALL) NOPASSWD: ALL" | sudo tee "/etc/sudoers.d/dont-prompt-ros-for-sudo-password"
sudo su - nexus

sudo apt update
sudo apt install openjdk-17-jdk

# https://help.sonatype.com/en/download.html
$ curl -LO https://download.sonatype.com/nexus/3/nexus-3.87.1-01-linux-x86_64.tar.gz
$ shasum -a 256 nexus-3.87.1-01-linux-x86_64.tar.gz 
9403cc4a78e11af09fc65e217e381dfcf435755dea31cdba9c947d6e1d439cd7  nexus-3.87.1-01-linux-x86_64.tar.gz

cd /opt
tar xvz --keep-directory-symlink -f ./nexus-3.87.1-01-linux-x86_64.tar.gz 

tar -xvzf /path/to/nexus-3.87.1-01-linux-x86_64.tar.gz \
  --directory=/opt \
  --keep-directory-symlink

./nexus-3.78.0-14 # Application Directory
./sonatype-work   # Parent of the default Data Directory
```

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
