---
name: docker-dockerfile-build-optimization
description: Writes and optimizes Dockerfiles and the Docker image build process - layered filesystem and copy-on-write mechanics, FROM/RUN/COPY/ADD/WORKDIR/CMD/ENTRYPOINT semantics, ordering instructions for layer-cache reuse, multi-stage builds to shrink final image size, .dockerignore, BuildKit build secrets, and docker buildx for multi-architecture (amd64/arm64) images. Use when the user asks "write a Dockerfile," "why is my Docker image so big," "why is my Docker build slow / why does it reinstall dependencies every time," "multi-stage build," "reduce Docker image size," "docker buildx," "build for arm64 and amd64," "BuildKit secret mount," "layer caching," or "what's the difference between CMD and ENTRYPOINT." This is Docker's own image-building pipeline (Dockerfile syntax, BuildKit, buildx) - distinct from podman-docker-migration-compatibility (which covers CLI/API compatibility once an image already exists) and distinct from docker-container-security-hardening (runtime hardening of a built image, not the build itself).
---

# Docker: Dockerfile and Build Optimization

A Docker image is a tarball of a **layered filesystem**: each layer stores only the delta versus
the layers below it, layers are immutable, and a running container adds one thin writable layer
on top (copy-on-write — a layer that modifies a file from below copies it up first, then edits the
copy). Because layers are immutable, Docker can **cache** them indefinitely and reuse a cached
layer verbatim on the next build — that caching behavior is the single biggest lever you have
over build speed and image size.

## The core mental model: one instruction = one layer, cache stops at the first change

Almost every Dockerfile instruction (`FROM`, `RUN`, `COPY`, `ADD`, ...) produces a new layer.
`CMD`/`ENTRYPOINT` do not — they're metadata, not filesystem changes. On a rebuild, Docker
reuses cached layers **up to the first one whose inputs changed** — every layer after that point
must be rebuilt, even if its own content wouldn't have changed. This is why instruction order is
the most impactful thing you control in a Dockerfile.

**Wrong order — busts the cache on every source change:**
```dockerfile
FROM node:23-bookworm
WORKDIR /app
COPY . /app          # any file change here invalidates everything below
RUN npm install      # reinstalls ALL dependencies on every code change, every time
CMD ["npm", "start"]
```

**Right order — dependency install only re-runs when the manifest changes:**
```dockerfile
FROM node:23-bookworm
WORKDIR /app
COPY package.json /app/     # rarely changes -> layer stays cached
RUN npm install              # only reruns when package.json changes
COPY . /app                  # frequent source changes only invalidate this cheap COPY
CMD ["npm", "start"]
```
The same principle applies to any ecosystem: copy `requirements.txt`/`go.mod`/`pom.xml` and
install/restore dependencies *before* copying the rest of the source tree.

## Dockerfile keyword reference (the parts people get wrong)

- **`FROM`** — the base image. `FROM scratch` is a no-op that starts from a genuinely empty
  filesystem (no layer generated) — useful for a single statically-linked binary. Prefer official,
  actively maintained base images (and `-slim`/`-alpine` variants) over full distro images.
- **`RUN`** — executes at *build* time. Chain related commands with `&&` and line continuations
  to avoid creating one layer per command (fewer layers = less work for the storage driver to
  merge at container start, and no leftover "delete this file" layer bloating the image):
  ```dockerfile
  RUN apt-get update \
      && apt-get install -y --no-install-recommends ca-certificates \
      && rm -rf /var/lib/apt/lists/*
  ```
  Doing the `apt-get update && install && cleanup` in **one** `RUN` matters — if you split it
  across three `RUN` lines, the cleanup layer doesn't shrink the image; the package files were
  already committed to an earlier, immutable layer.
- **`COPY` vs `ADD`** — functionally similar; `ADD` additionally auto-extracts local tar archives
  and can fetch a URL. Prefer `COPY` unless you specifically need one of those two `ADD`
  behaviors — it's more predictable. Both support `--chown=uid:gid` to set file ownership in the
  same layer (avoids a separate `RUN chown` layer that doubles the size cost of large files).
- **`WORKDIR`** — sets the working directory for every subsequent instruction *and persists
  across layers*. `RUN cd /app` does **not** persist — each `RUN` is its own shell invocation, so a
  bare `cd` is silently thrown away by the next instruction. Always use `WORKDIR`, not `RUN cd`.
- **`CMD` vs `ENTRYPOINT`** — think of a shell command as `<command> <args>`. `ENTRYPOINT`
  defines the command; `CMD` defines its default arguments. Use the JSON-array "exec form"
  (`["ping", "-c", "3"]`) over the shell form (`CMD ping -c 3`) — exec form runs the process
  directly as PID 1 (proper signal handling), shell form wraps it in `/bin/sh -c "..."`.
  `docker run <image> <new-args>` overrides `CMD`; `docker run --entrypoint <cmd> <image>`
  overrides `ENTRYPOINT`. If `ENTRYPOINT` is unset, Docker defaults it to `/bin/sh -c`, and
  `CMD`'s value is passed to that shell as a string.
- **`ARG` vs `ENV`** — `ARG` is a build-time-only variable (`--build-arg`), not present in the
  running container unless you also assign it to an `ENV`. `ENV` is baked into the image and
  visible to every container run from it (and every downstream image that uses it as a base) —
  never put secrets in `ENV`.

## Multi-stage builds: the single biggest size win

A naive Dockerfile for a compiled language bakes the entire toolchain (compiler, SDK, headers)
into the final image. Example: a "Hello, World" C binary built this way weighs in around 260 MB
because the image carries the whole Alpine SDK it needed only to compile.

Fix it by splitting the Dockerfile into a **build stage** and a minimal **runtime stage**, copying
only the compiled artifact across the boundary with `COPY --from=<stage>`:

```dockerfile
FROM alpine:3.21 AS build
RUN apk update && apk add --update alpine-sdk
WORKDIR /app
COPY . /app
RUN mkdir bin && gcc -static -O2 hello.c -o bin/hello   # static link: no runtime deps needed

FROM scratch
COPY --from=build /app/bin/hello /app/hello
ENTRYPOINT ["/app/hello"]
```
That takes the same app from ~260 MB down to a couple hundred KB — the SDK, headers, and
intermediate object files never appear in the final image at all. The same pattern applies to any
compiled or bundled language: `FROM node:23 AS build` → `npm ci && npm run build` → final
stage `FROM node:23-slim` (or `nginx:alpine` for a static SPA) copying only `dist/`.

Smaller images matter beyond disk space: less to pull/push over the network, faster container
start (less for the storage driver's union filesystem to assemble), and a smaller attack surface for
vulnerability scanners to flag.

## `.dockerignore`: mandatory, not optional

Without one, `COPY . .` sends everything in the build context — `.git/`, local virtualenvs,
`node_modules/`, `.env` files with live credentials — to the builder, which both bloats the image
and risks baking secrets into a layer that's cached forever and shared with anyone who pulls the
image.

```
# Ignore everything by default, then allow-list what the build actually needs
**
!my-app/
!scripts/
my-app/*.log
scripts/temp/
.git
.env
node_modules
__pycache__
```
A `.dockerignore` also speeds up the build itself: the Docker daemon has to transfer the entire
build context before the first instruction even runs, so excluding large irrelevant directories
(`node_modules`, `.git`, build output) cuts that transfer time.

## BuildKit build secrets: credentials the build needs but the image must not keep

Passing a private-registry token or Git credential through `ARG`/`ENV` leaves it recoverable in
`docker history` or an intermediate layer forever, even if a later `RUN rm` "deletes" it (the earlier
layer is immutable — the secret is still on disk in it). BuildKit's secret mount exposes a secret
only to the one `RUN` step that needs it, as a file that never gets committed to any layer:

```dockerfile
# syntax=docker/dockerfile:1
FROM alpine
RUN --mount=type=secret,id=repo_token \
    TOKEN=$(cat /run/secrets/repo_token) && \
    wget --header="Authorization: Bearer $TOKEN" https://private.example.com/artifact.tar
```
```bash
docker build --secret id=repo_token,src=./repo_token.txt -t myapp .
```
Use this for anything the build needs to *fetch* (private package registries, internal artifact
repos) — never bake the credential itself into the final image via `ARG`/`ENV`.

## Multi-architecture builds with buildx

A single-platform image built on an Apple Silicon laptop won't run on an x86_64 cloud node (and
vice versa). `docker buildx` builds and pushes one manifest covering multiple platforms in one
command:

```bash
# One-time: create a builder that supports multi-platform output
docker buildx create --use --name mbuilder
docker buildx inspect --bootstrap

# Build and push both architectures under one tag
docker buildx build \
  --platform linux/amd64,linux/arm64 \
  -t yourregistry/yourapp:1.0.1 \
  --push .
```
`--push` is required for multi-platform output (a multi-arch result can't be loaded into the local
single-arch image store the way `docker build` normally would).

## Common pitfalls

- **`COPY . .` before installing dependencies** — busts the cache on every source change; always
  copy the dependency manifest and install first.
- **Splitting a package-install-then-cleanup sequence across multiple `RUN` lines** — the cleanup
  doesn't shrink anything; the files were already committed to an earlier immutable layer. Chain
  them in one `RUN` with `&&`.
- **No `.dockerignore`** — leaks `.git`, `.env`, and dev artifacts into the image and slows every
  build by shipping unnecessary context to the daemon.
- **Skipping multi-stage builds for compiled/bundled apps** — shipping the compiler/SDK/full
  `node_modules` dev tree in production images inflates size and attack surface for no benefit.
- **Putting secrets in `ARG`/`ENV`** — both persist in `docker history`/image layers even after a
  later instruction "removes" them; use BuildKit `--mount=type=secret` for build-time credentials
  and a runtime secrets mechanism (see docker-container-security-hardening) for the running app.
- **Relying on `RUN cd /some/dir`** to set context for later instructions — it doesn't persist across
  layers; use `WORKDIR`.
- **Forgetting exec-form `CMD`/`ENTRYPOINT`** — shell form runs your process as a child of
  `/bin/sh`, breaking signal propagation (e.g., a container that ignores `SIGTERM` on `docker stop`).
- **Building single-arch images for a mixed-architecture fleet** (Apple Silicon dev machines,
  x86_64 cloud nodes) — use `docker buildx build --platform` instead of assuming the build
  host's architecture matches production.

## Checklist

- [ ] Dependency manifests are copied and installed *before* the rest of the source tree
- [ ] Related `RUN` commands (install + cleanup) are chained in a single layer with `&&`
- [ ] A `.dockerignore` excludes `.git`, secrets, local envs, and build artifacts
- [ ] Compiled/bundled applications use a multi-stage build so the final image excludes the
      toolchain
- [ ] No secret ever appears in `ARG`, `ENV`, or a `COPY`'d file — build-time secrets use
      `RUN --mount=type=secret`
- [ ] `CMD`/`ENTRYPOINT` use JSON exec-array form, not shell-string form
- [ ] Images destined for a mixed-architecture fleet are built with `docker buildx build --platform`
- [ ] Final image size and layer count were actually checked (`docker image ls`,
      `docker image history <image>`), not assumed
