---
layout: section
---

# Choosing a binary store

<br>
<br>
<Link to="toc" title="Table of Contents"/>

---
hideInToc: true
level: 2
class: text-sm
---

# Nexus, Artifactory and Pulp

| | Sonatype Nexus Repository | JFrog Artifactory | Pulp |
| --- | --- | --- | --- |
| **Free version** | Community Edition: every format, up to 40,000 components and 100,000 requests a day | Artifactory OSS (Java and generic), JFrog Container Registry (Docker, Helm), CE for C/C++ (Conan, generic) | All of it: GPLv2, no paid edition |
| **Pros** | Proxy, hosted and group repositories for every format. Web UI. One JVM to run | The widest range of formats. Virtual repositories. JFrog CLI, Xray scanning, SaaS option | Every change is a repository version you can roll back to. Strong rpm and deb support (Red Hat Satellite is built on it). No usage limits |
| **Cons** | Past the Community Edition limits, nobody can add new components. No CLI: automate the REST API | Each free edition covers a few formats; mixing formats needs a paid license | No web UI. PostgreSQL, Redis and three services to run. No raw proxy or group repositories |
| **Pick it when** | You want one free server for many formats, and stay under the limits | You're paying anyway, or need only Docker/Helm or only Maven | You mirror Linux distributions, want versioned snapshots, or need open source with no limits |
