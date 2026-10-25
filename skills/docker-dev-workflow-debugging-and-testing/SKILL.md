---
name: docker-dev-workflow-debugging-and-testing
description: Makes day-to-day Docker development productive - inspecting and debugging running containers (docker exec, logs, inspect, container diff), bind-mounting source code for hot-reload (nodemon and language equivalents), attaching a remote debugger (VS Code launch.json) to code running inside a container, running unit/integration/functional tests in containers for CI parity, and Docker Engine housekeeping (docker system/image/container prune, running one-off admin tasks in disposable containers, bind-mounting the Docker socket to run Docker CLI commands from within a container). Use when the user asks "how do I debug code running in a container," "hot reload in Docker," "attach VS Code debugger to a container," "run tests in Docker for CI," "docker exec into a container," "my Docker disk usage is huge, how do I clean it up," "run a one-off script in a container without installing anything," or "docker-in-docker." Distinct from docker-container-security-hardening (the docker.sock-mounting pattern's security implications) and docker-compose-multi-service-development (multi-service orchestration, not single-container debugging).
---

# Docker Development Workflow: Debugging, Hot Reload, and Testing

The productivity case for containerized development is that the container-based inner loop
should feel as fast and interactive as running the app directly on the host. That requires three
things working together: bind-mounting live source code, a hot-reload mechanism, and a way to
attach a real debugger — plus knowing the basic inspection commands well enough to not guess.

## Basic container inspection (the commands to reach for before guessing)

```bash
docker container logs -f <name>          # follow stdout/stderr
docker container exec -it <name> /bin/sh # run a shell (or any command) inside a running container
docker container inspect <name>          # full JSON: mounts, network, env, health status
docker container diff <name>             # filesystem changes vs. the image (A=added, C=changed, D=deleted)
docker container top <name>              # processes running inside the container
docker stats                             # live CPU/memory/IO across all running containers
```
`docker container diff` is underused — it's the fastest way to answer "what did this container
actually change on disk" without shelling in and poking around manually, and it's also how
interactive image creation (`docker container commit`) is verified before committing.

## Bind-mount source code for a live development loop

Mount the host's working tree into the container instead of rebuilding the image on every
change:
```bash
docker container run --rm -it \
  -v $(pwd):/usr/src/app \
  -w /usr/src/app \
  node:23-bookworm sh
```
This alone gets you live files inside the container, but the running process still needs to notice
the change and restart — that's the next piece.

## Hot reload: restart the process automatically on file change

**Node.js** — `nodemon` is the standard tool:
```dockerfile
FROM node:23-bookworm
WORKDIR /app
COPY package.json /app/
RUN npm install
COPY . /app
RUN npm install -g nodemon
CMD ["nodemon", "index.js"]
```
Run the container with the source bind-mounted (`-v $(pwd):/app`) and edits on the host are
picked up immediately — no rebuild, no restart command.

**Any other language via the same tool** — `nodemon --exec` isn't Node-specific under the hood;
it just watches files and re-runs a given command:
```bash
nodemon --exec python main.py
```
**Framework-native alternatives** are often preferable when available — e.g., Flask's own
`flask run --debug` gives auto-restart plus an interactive debugger page without an extra
dependency. Prefer the framework's built-in dev-mode flag over a generic file-watcher when one
exists; it usually integrates better with that framework's error reporting.

## Remote debugging: attach VS Code's debugger into the container

The pattern is the same across languages: the process inside the container starts with a
debug/inspect flag listening on a port, that port is published to the host, and VS Code's
`launch.json` is configured to *attach* (not launch) to it.

```jsonc
// .vscode/launch.json
{
  "version": "0.2.0",
  "configurations": [
    {
      "type": "node",
      "request": "attach",
      "name": "Attach to Node in Docker",
      "address": "localhost",
      "port": 9229,
      "localRoot": "${workspaceFolder}",
      "remoteRoot": "/usr/src/app",
      "restart": true
    }
  ]
}
```
```bash
docker container run --rm -it \
  -v $(pwd):/usr/src/app \
  -p 9229:9229 \
  node:23-bookworm node --inspect=0.0.0.0:9229 index.js
```
Set a breakpoint in VS Code, hit "Attach to Node in Docker," and execution stops on the host
editor even though the process is running inside the container. The same shape applies to Java
(JDWP on a published debug port), Python (`debugpy`), and .NET (the VS Code remote debugger) —
only the debug-protocol flag and default port change.

For a fuller dev environment (editor, extensions, and terminal all running inside the container,
not just the debugged process), VS Code's Remote-Containers/Dev Containers extension runs the
whole editor server inside a container built from your project's own Dockerfile, driven by a
`.devcontainer/devcontainer.json` — useful when a workstation is locked down and can't have the
language toolchain installed directly.

## Running tests in containers

The point of running tests inside a container isn't just "because Docker" — it's that the exact
same image that will ship to production is what gets tested, closing the "works in CI, fails in
prod" gap caused by host/CI-runner environment drift.

```bash
# Unit/integration tests: run the app image with the test command instead of the normal CMD
docker container run --rm myapp:test npm test

# Functional/black-box tests: a separate dedicated test container hits the running app over the network
docker compose -f compose.yml -f compose.test.yml run --rm tests
```
Typical CI shape: build the image once, run unit tests inside it, run functional tests from a
sibling container against a running instance (via Compose), and only push/tag the image if every
stage passes — the same artifact that passed tests is the one that gets deployed, not a rebuild.

## Running one-off admin tasks without installing anything on the host

If you need a tool you don't have installed locally, run it in a throwaway container instead of
installing it on the host:
```bash
docker container run --rm -it \
  -v $(pwd):/usr/src/app -w /usr/src/app \
  perl:slim sh -c "cat sample.txt | perl -lpe 's/^\s*//'"

docker container run --rm -it \
  -v $(pwd):/usr/src/app -w /usr/src/app \
  python:3-alpine python stats.py sample.txt
```
`--rm` means the container (and its writable layer) disappears the moment the command
finishes — nothing lingers on the host beyond the mounted files' own changes. This is also the
clean way to run an *old* version of a tool (e.g., a legacy Perl script) without version-conflicting
with whatever's installed natively.

## Automating Docker itself from inside a container

Bind-mounting the Docker socket lets a containerized tool (e.g., a custom CI runner image) issue
Docker commands against the *host's* Docker Engine, without installing Docker Engine inside the
container:
```bash
docker run --rm -it \
  -v /var/run/docker.sock:/var/run/docker.sock \
  docker:cli sh
# now `docker container ls`, `docker image ls`, etc. control the HOST's Docker daemon
```
This is different from — and generally safer than — **Docker-in-Docker (DinD)**, which runs a
full nested Docker Engine inside the container. DinD adds real complexity (nested storage/network
namespace handling) and is generally discouraged; socket-mounting the CLI against the host
daemon is the more common and better-understood pattern. Either way, treat the Docker socket as
root-equivalent access to the host — see docker-container-security-hardening before wiring this
into anything that isn't a trusted, controlled CI environment. For CI specifically, tools like Kaniko
or Buildah can build images without needing socket access or privileged mode at all.

## Engine housekeeping

Docker accumulates disk usage from layers, stopped containers, and unused volumes far faster
than most people expect in an active dev loop:
```bash
docker image prune -f          # dangling (untagged, unreferenced) images - safe to force
docker container prune -f      # stopped containers - safe to force
docker volume prune            # WARNING: deletes unused volumes - deliberately omit -f here
docker system df               # see what's actually consuming space before nuking everything
```
Treat volume pruning differently from image/container pruning: volumes hold data, and that data's
lifecycle is often intentionally longer than any single container's. Never script `docker volume
prune -f` into a routine cleanup job that also touches anything resembling production-adjacent
data.

## Common pitfalls

- **Bind-mounting source without a restart mechanism** and wondering why code changes don't
  take effect — the file changes on disk, but the already-running process doesn't know to reload
  it. Pair the bind mount with `nodemon`/framework dev-mode/a debugger's own hot-reload.
- **Launching, not attaching, the debugger** — `launch.json` configured with `"request": "launch"`
  tries to start a new process on the host; containerized debugging needs `"request": "attach"`
  against the already-running, published debug port.
- **Testing a rebuilt image instead of the exact image that will ship** — defeats the entire point
  of container-based testing; build once, test that artifact, promote it unchanged.
- **Installing a rarely-needed CLI tool on the host** instead of running it once in a throwaway
  `--rm` container — clutters the host and doesn't help the next person/machine that needs the
  same one-off task done.
- **Reaching for full Docker-in-Docker** when socket-mounting the CLI against the host daemon
  would do the job with far less operational complexity.
- **Force-pruning volumes as part of routine disk cleanup** — the one Docker prune command
  where `-f` turns a housekeeping habit into potential data loss.

## Checklist

- [ ] Live-reload dev containers pair a bind-mounted source tree with an actual restart mechanism
      (nodemon, framework dev-mode, or equivalent) — not just the mount alone
- [ ] VS Code (or equivalent) debug configs use `"request": "attach"` against a published
      debug port, not `"request": "launch"`
- [ ] CI builds the image once and runs unit + functional tests against that exact artifact, rather
      than rebuilding between test stages
- [ ] One-off host tooling needs are satisfied with a disposable `--rm` container instead of
      installing onto the host
- [ ] Docker-socket-mounting is understood as root-equivalent host access, reserved for trusted
      automation rather than arbitrary containers
- [ ] `docker volume prune` is never force-scripted alongside routine image/container cleanup
