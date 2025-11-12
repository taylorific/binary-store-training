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
