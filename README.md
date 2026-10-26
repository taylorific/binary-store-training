# Binary Store Training

To start the slide show, with either Docker or Apple's
[container](https://github.com/apple/container) CLI:

## Docker

```bash
docker run -it --rm \
  --mount type=bind,source="$(pwd)",target="/slidev" \
  --publish 3030:3030 \
  docker.io/boxcutter/slidev

# Visit <http://localhost:3030/>
```

## Apple container (macOS)

```bash
container system start   # once per boot

container run -it --rm \
  --mount type=bind,source="$(pwd)",target="/slidev" \
  --publish 3030:3030 \
  docker.io/boxcutter/slidev

# Visit <http://localhost:3030/>
```

Edit the [slides.md](./slides.md) to see the changes.

Learn more about Slidev at the [documentation](https://sli.dev/).

# Sections

Each section is its own file, pulled into [slides.md](./slides.md) with `src:`:

| Directory | Covers |
| --- | --- |
| [overview](./overview/slides.md) | Choosing a binary store: Nexus, Artifactory and Pulp pros and cons, and the other options |
| [nexus](./nexus/slides.md) | Sonatype Nexus Repository: install from the tarball as a systemd service, with Docker or with Apple container, then the admin password, EULA and anonymous access, and the [boxcutter_sonatype](https://github.com/boxcutter/boxcutter-chef-cookbooks/tree/main/cookbooks/boxcutter_sonatype) setup as `curl` commands |
| [artifactory](./artifactory/slides.md) | JFrog Artifactory |
| [pulp](./pulp/slides.md) | Pulp: install with Docker, Apple container or by hand, then the [boxcutter_sonatype](https://github.com/boxcutter/boxcutter-chef-cookbooks/tree/main/cookbooks/boxcutter_sonatype) setup redone in Pulp, compared with Nexus |
| [harbor](./harbor/slides.md) | Harbor: install on premises with the installer and Docker Compose, then the cookbook's Docker setup redone in Harbor (proxy cache, projects, users, robot accounts, Helm charts, Trivy scanning), compared with Nexus |

# Update dependencies

## Docker

To see what is out of date:

```bash
docker run -it --rm \
  --mount type=bind,source="$(pwd)",target="/slidev" \
  --entrypoint npm \
  docker.io/boxcutter/slidev outdated
```

To upgrade Slidev and the themes to their latest versions:

```bash
docker run -it --rm \
  --mount type=bind,source="$(pwd)",target="/slidev" \
  --entrypoint npm \
  docker.io/boxcutter/slidev \
    install @slidev/cli@latest @slidev/theme-default@latest @slidev/theme-seriph@latest
```

## Apple container (macOS)

To see what is out of date:

```bash
container run -it --rm \
  --mount type=bind,source="$(pwd)",target="/slidev" \
  --entrypoint npm \
  docker.io/boxcutter/slidev outdated
```

To upgrade Slidev and the themes to their latest versions:

```bash
container run -it --rm \
  --mount type=bind,source="$(pwd)",target="/slidev" \
  --entrypoint npm \
  docker.io/boxcutter/slidev \
    install @slidev/cli@latest @slidev/theme-default@latest @slidev/theme-seriph@latest
```

Then, on the host:

```bash
git add package.json package-lock.json
git commit -m "Sync npm lockfile"
git push
```

# Attributions

The following projects were used for inspiration and some code was
reused for the slideware:

- https://sli.dev/ - Slidev slideware
- https://github.com/alexanderdavide/slidev-theme-academic  Pagination  - Alexander Eble
- https://github.com/jpetazzo/container.training - Idea of using markdown slideware for training - Jérôme Petazzoni
