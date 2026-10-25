---
name: docker-volumes-storage-and-configuration
description: Manages stateful data in Docker containers - named volumes versus bind mounts (when to use each), creating/sharing/inspecting/pruning volumes, read-only mounts, declaring expected mount points in a Dockerfile with VOLUME, and runtime configuration via environment variables and build-time ARG. Use when the user asks "docker volume create," "bind mount vs named volume," "how do I persist data in a container," "share data between containers," "mount a host folder read-only," "VOLUME instruction in a Dockerfile," "configure a container with environment variables," or "my container loses its data on restart." Distinct from the storage/volume-driver questions covered for Kubernetes in container-orchestration-deployment-strategies (PersistentVolumes at cluster scale) - this is single-host Docker Engine volume and bind-mount mechanics.
---

# Docker Volumes, Storage, and Runtime Configuration

Containers are meant to be ephemeral, but the data they produce or depend on usually isn't.
Docker gives you two mechanisms to persist and share data outside a container's writable
layer — **volumes** and **bind mounts** — plus environment-variable/build-arg mechanisms for
configuring behavior without baking values into the image.

## Named volumes vs. bind mounts: use volumes unless you need host-path awareness

Both attach host storage into a container's filesystem via `-v`/`--mount`, but they differ in who
owns the lifecycle and portability:

| | Named volume | Bind mount |
|---|---|---|
| Location | Managed by Docker (`/var/lib/docker/volumes/...`) | An exact path you specify on the host |
| Portability | Fully portable — same `docker run -v myvol:/data` works on any Docker host | Tied to that host's specific filesystem layout |
| Lifecycle | Survives container removal; created/reused automatically | Just a host path; Docker doesn't manage its lifecycle at all |
| Typical use | Production persistent state (databases, uploaded files) | Local development (mounting live source code into a container) |

```bash
# Named volume - Docker owns and manages the storage location
docker volume create sample
docker container run -v sample:/data alpine /bin/sh

# Bind mount - you own the host path explicitly
docker container run -v $(pwd)/src:/app/src my-dev-image
```
**Volumes are the recommended default for persistence** precisely because they're portable and
Docker-managed; bind mounts remain the right tool specifically for the development workflow of
live-mounting your working source tree into a running container (see the dev-workflow skill for
hot-reload patterns built on this).

## Sharing data between containers

Multiple containers can mount the same named volume simultaneously:
```bash
docker volume create shared-data
docker container run -d --name writer -v shared-data:/data alpine ...
docker container run -d --name reader -v shared-data:/app/data:ro alpine ...
```
The `:ro` suffix mounts the volume **read-only** in that container — useful for a consumer that
should never be able to corrupt data a producer container owns. Any write attempt inside the
read-only mount fails at the filesystem level, not just by convention.

## Declaring expected mount points in the image: `VOLUME`

```dockerfile
VOLUME /app/data
VOLUME /app/data, /app/profiles, /app/config
VOLUME ["/app/data", "/app/profiles", "/app/config"]
```
This tells Docker "this path is expected to hold data that should survive a container replacement."
If you run a container from such an image *without* specifying a mount for that path, Docker
auto-creates an anonymous volume for it — better than losing the data outright, but anonymous
volumes are easy to forget about and accumulate as clutter. Prefer explicitly naming volumes at
`run`/Compose time over relying on the anonymous fallback.

## Configuring containers: environment variables and build args

```dockerfile
ENV LOG_DIR=/var/log/my-log
ENV MAX_LOG_FILES=5
ENV MAX_LOG_SIZE=1G

ARG BASE_IMAGE_VERSION=12.7-stretch
FROM debian:${BASE_IMAGE_VERSION}
```
- **`ENV`** sets a default baked into the image and visible to every container run from it (and
  overridable per-container with `docker run -e KEY=value` or Compose's `environment:`).
- **`ARG`** exists only during the *build* — a container run from the finished image doesn't see it
  unless you also assign it to an `ENV`. Use `ARG` to parameterize the build itself (e.g., which
  base image tag to build from via `--build-arg`), and `ENV` for values the running application
  needs to read.
- Never put secrets in either — both persist in inspectable image metadata/layers. See
  docker-container-security-hardening for the actual secrets-management mechanisms
  (BuildKit secret mounts for build time, Docker/Compose secrets or an external vault for runtime).

## Housekeeping: volumes accumulate silently

```bash
docker volume prune          # WARNING: removes every volume not referenced by any container
docker volume rm <name>      # remove one specific volume
```
Deliberately **do not** default to `-f`/`--force` on `volume prune` — unlike image or container
pruning, volumes are the one Docker resource type where an accidental deletion is often
unrecoverable data loss, not just wasted disk space. In CI or throwaway dev environments this
caution matters less; on anything touching production-like data, delete volumes one at a time by
name after confirming they're truly unused.

To find out why a bind mount or volume mapping isn't behaving as expected, remove the
container along with any anonymous volumes it created rather than leaving orphans behind:
```bash
docker container rm -v -f <container>
```

## Common pitfalls

- **Reaching for a bind mount in production "because it's simpler"** — ties the deployment to
  that exact host's filesystem layout and loses Docker's volume lifecycle management; use a
  named volume for anything meant to persist and move between hosts.
- **Forgetting `:ro` on a mount that should never be written to** — a consumer container with
  accidental write access to shared data is a correctness and security risk, not just tidiness.
  Read/write access should be an explicit choice, not the unexamined default.
- **Relying on anonymous volumes created implicitly by `VOLUME`** — they work, but they're easy
  to lose track of and prune accidentally; name your volumes explicitly at run/Compose time.
- **Putting credentials in `ENV`** — visible via `docker inspect`, in process listings, and often in
  logs; use a real secrets mechanism instead.
- **Running `docker volume prune` (especially with `-f`) as a reflex disk-cleanup habit** — it is
  the one prune command that destroys data, not just reclaimed disk space from unused images.
- **Assuming `ARG` values are readable inside the running container** — they're build-time only
  unless explicitly re-exposed via `ENV`.

## Checklist

- [ ] Persistent application state uses a named volume, not a bind mount, unless the workflow
      specifically needs host-path awareness (e.g., live-mounted dev source)
- [ ] Mounts that should never be written to by a given container are explicitly marked `:ro`
- [ ] `Dockerfile`/`compose.yml` volumes are named explicitly rather than left to anonymous-volume
      fallback
- [ ] No secret lives in `ENV`, `ARG`, or a file baked into an image layer
- [ ] `docker volume prune` is a deliberate, confirmed action — never scripted with `-f` against
      anything that might hold real data
- [ ] `ARG` is used for build-time parameterization, `ENV` for values the running app actually reads
