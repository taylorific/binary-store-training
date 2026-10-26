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

---
hideInToc: true
level: 2
class: text-sm
---

# Other options

| | What it is | Consider it when |
| --- | --- | --- |
| **Harbor** | Open source registry for containers and Helm charts, with scanning and replication. It has its own section | You only need containers and Helm charts |
| **Your forge's registry** | GitLab, GitHub Packages, Gitea and Forgejo serve packages and images next to the code | Packages are built and used by the same projects, and you don't need to proxy upstream |
| **Cloud registries** | AWS CodeArtifact and ECR, Google Artifact Registry, Azure Artifacts and ACR | You already run in that cloud. Nothing to install, but you pay per GB stored and transferred |
| **One-format tools** | aptly or reprepro (apt), Verdaccio (npm), devpi (PyPI), Zot (OCI) | You need one format, and want something small |
| **Commercial services** | Cloudsmith (SaaS), ProGet (Windows or Linux, free tier) | You want one place for many formats, without running Nexus or Artifactory |

Nexus, Artifactory, Pulp and Harbor all run on your own hosts. Of the rest,
the forges, the one-format tools and ProGet do too.
