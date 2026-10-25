---
name: docker-networking-bridge-dns-reverse-proxy
description: Configures Docker's own single-host networking - the default and custom bridge networks (docker0, iptables/nftables-based NAT), Docker's embedded DNS server for name-based service discovery on user-defined networks, the host and none network drivers, sharing a container's network namespace with --network container:<name>, publishing ports with -p/-P versus EXPOSE, and fronting multiple containers with a Traefik reverse proxy using Docker-provider labels. Use when the user asks "docker network create," "why can't my containers resolve each other by name," "docker0 bridge," "difference between EXPOSE and -p," "docker --network host," "run a container in another container's network namespace," "Traefik with Docker labels," or "publish a port range / UDP port in Docker." This is Docker Engine's own CNM/iptables networking stack - distinct from podman-networking-netavark-pasta (Podman's Netavark/aardvark-dns/pasta implementation of the same concepts) - the mechanisms rhyme but the daemon, tooling, and default network name (`bridge` vs `podman`) differ.
---

# Docker Single-Host Networking

Docker's networking is built on the **Container Network Model (CNM)**, made concrete on a
single host through the **bridge** driver. When the Docker daemon first starts, it creates a Linux
bridge called `docker0` backing a network literally named `bridge` — this is where every
container lands by default unless told otherwise.

## The default `bridge` network has no name-based DNS

```bash
docker network inspect bridge | grep Subnet
# "Subnet": "172.17.0.0/16"
```
Containers on the default `bridge` network can reach each other only by IP — Docker's embedded
DNS server does **not** serve the default bridge, only **user-defined** networks. This is the
single most common "why can't my containers see each other by name" question, and the fix is
always the same: stop using the default network.

```bash
docker network create --driver bridge sample-net
docker container run --name web  --network sample-net -d nginx:1.29-alpine
docker container run --name api  --network sample-net -d myapi:1.0
# from inside `api`, `curl web` resolves via Docker's embedded DNS - no IP needed
```

Custom bridge networks also isolate containers from each other by default: two containers on
different user-defined networks cannot reach each other unless explicitly connected to a shared
network — a lightweight built-in firewall boundary you get just by choosing network topology.

Useful variants at creation time:
```bash
# Explicit subnet instead of Docker's auto-assigned next-free block
docker network create --driver bridge --subnet 10.1.0.0/16 test-net

# Dual-stack IPv4 + IPv6 (stable since Docker 20.10)
docker network create --driver bridge --ipv6 \
  --subnet 10.10.0.0/16 --subnet fd00:dead:beef::/48 mynet-v6

# Custom MTU, e.g. to avoid fragmentation behind a VPN/overlay
docker network create --driver bridge --opt com.docker.network.driver.mtu=1400 mynet-mtu
```
On modern distros (Ubuntu 24.04+, Fedora 38+) Docker's firewall rules are implemented with
`nftables` rather than `iptables` — inspect with `sudo nft list ruleset` instead of `sudo iptables -L`
when troubleshooting connectivity on newer hosts.

## `host` and `none`: the two special local drivers

**`--network host`** removes network-namespace isolation entirely — the container shares the
host's network stack and IP directly, with no NAT layer:
```bash
docker container run --rm -it --network host alpine:3.22 /bin/sh
```
Use cases are narrow: high-performance services where NAT overhead matters, or tools that need
to observe the host's network directly. Caveats that catch people out:
- **Linux only** in the sense that matters — on Docker Desktop (macOS/Windows) the container
  attaches to the lightweight VM's network, not your physical host, so you still need `-p` to reach
  it from the Mac/Windows side.
- **Security**: a compromised container on the host network can sniff *all* host network traffic.
  There is essentially no good reason to use `--network host` for a production business
  application — prefer a user-defined bridge network with explicit port publishing.

**`--network none`** gives the container only a loopback interface and nothing else — no `eth0`,
no routing table, no reachability in either direction:
```bash
docker container run --rm -it --network none alpine:3.22 /bin/sh
# ip addr show eth0 -> "can't find device 'eth0'"
```
Use it for security-sensitive batch/offline-processing containers that have no legitimate need for
any network access at all.

## Sharing another container's network namespace

`--network container:<name>` attaches a new container into an *existing* container's network
namespace instead of creating its own — both containers then share the same IP, ports, and
loopback, and can talk to each other over `localhost`:
```bash
docker network create --driver bridge test-net
docker container run --name web -d --network test-net nginx:1.29-alpine
docker container run -it --rm --network container:web alpine:3.22 /bin/sh
# inside this shell, `wget localhost:80` reaches the web container's Nginx directly
```
This is exactly the mechanism Kubernetes uses to give every container in a Pod a shared network
identity — understanding it here is the most direct route to understanding what a Pod's
"shared network namespace" actually means at the kernel level. It's also the standard pattern for
attaching a debugging toolbox to a running service without modifying its image:
```bash
docker container run --rm -it --network container:myapp nicolaka/netshoot
# now curl/dig/tcpdump/traceroute run in myapp's exact network context
```
Sharing a namespace also shares its network-level blast radius — if either container is
compromised, the attacker can observe or manipulate the other's traffic.

## Publishing ports: `EXPOSE` documents, `-p`/`-P` actually wires

`EXPOSE 80` in a Dockerfile is **metadata only** — it documents intent and does nothing at
runtime. Only `-p`/`--publish` (or `-P` for automatic mapping) on `docker container run` actually
opens a path from the host into the container, by allocating a host listener and installing
NAT/forward rules that DNAT host-port traffic to `<container-ip>:<container-port>`.

```bash
# Docker picks a free ephemeral host port (32768-60999 / 49152-65535) for every EXPOSEd port
docker container run --name web -P -d nginx:1.29-alpine
docker container port web            # 80/tcp -> 0.0.0.0:55000

# Explicit host:container mapping
docker container run --name web2 -p 8080:80 -d nginx:1.29-alpine

# UDP needs its own explicit mapping - TCP is the default and is NOT implied for UDP
docker container run -d -p 3000:4321/udp myudpsvc

# Port ranges, for a cluster of like services
docker container run -d -p 12000-12010:12000-12010 my/worker

# Bind only to loopback for local dev - don't expose dev services on the LAN/Wi-Fi
docker container run -d -p 127.0.0.1:8080:80 my/dev-app
```
Two containers cannot bind the same host port — give each a distinct host port, or better, put a
single reverse proxy in front and leave the backend services' ports unpublished on an internal
bridge network entirely.

## HTTP-level routing with Traefik

Once you're running more than a couple of HTTP services, publishing a raw host port per service
doesn't scale and leaks internal topology. A reverse proxy gives you one public entry point that
routes by hostname/path — the standard pattern for "strangling" a monolith into microservices
incrementally without breaking existing clients.

```bash
# Each backend service opts in via labels - nothing is proxied unless explicitly enabled
docker container run --rm -d --name catalog \
  --label traefik.enable=true \
  --label traefik.port=3000 \
  --label traefik.priority=10 \
  --label traefik.http.routers.catalog.rule="Host(\"acme.com\") && PathPrefix(\"/catalog\")" \
  acme/catalog:1.0

docker container run --rm -d --name city-tours \
  --label traefik.enable=true \
  --label traefik.port=5000 \
  --label traefik.priority=1 \
  --label traefik.http.routers.city-tours.rule="Host(\"acme.com\")" \
  acme/city-tours:1.0

# Traefik itself watches the Docker socket to discover labeled containers automatically
docker container run -d --name traefik \
  -p 80:80 -p 8080:8080 \
  -v /var/run/docker.sock:/var/run/docker.sock \
  traefik:v2.0 \
  --api.insecure=true \
  --providers.docker=true \
  --entrypoints.web.address=:80 \
  --providers.docker.exposedbydefault=false
```
The higher-priority, more-specific `/catalog` rule wins over the low-priority catch-all, so
`acme.com/catalog/...` routes to the new microservice while everything else keeps hitting the
monolith — you can extract routes one at a time. Note the same Docker-socket-mount pattern
used here (see docker-container-security-hardening for the security implications of mounting
`/var/run/docker.sock` into any container).

## Common pitfalls

- **Expecting name-based DNS resolution on the default `bridge` network** — it only works on
  user-defined bridge networks; create one and attach your containers to it instead.
- **Confusing `EXPOSE` with actually publishing a port** — `EXPOSE` is documentation baked into
  the image; only `-p`/`-P` at `docker run` time (or a Compose `ports:` entry) opens host access.
- **Using `--network host` for a production business service "to simplify things"** — it removes
  network isolation entirely; there's essentially never a good reason to do this outside
  performance-critical edge cases understood and accepted as a tradeoff.
- **Assuming UDP is covered when you mapped the TCP port** — TCP and UDP are mapped
  independently; specify `/udp` explicitly if you need it.
- **Publishing every backend service's port directly** instead of routing them all through a single
  reverse proxy — leaks internal topology and multiplies the surface you have to secure/TLS-terminate.
- **Forgetting that `--network container:<name>` shares blast radius**, not just connectivity — a
  compromised sidecar in that shared namespace can intercept the other container's traffic.

## Checklist

- [ ] Multi-container name resolution uses a custom bridge network, never the default `bridge`
- [ ] Internal-only services stay unpublished on a private bridge network; only the reverse proxy
      (or a genuinely public service) publishes a host port
- [ ] `--network host` usage is deliberate and justified, not a default fallback for "networking
      isn't working"
- [ ] UDP ports are mapped explicitly with `/udp` when the protocol needs it, not assumed to ride
      along with a TCP mapping
- [ ] Local-dev-only services bind to `127.0.0.1` rather than `0.0.0.0` to avoid LAN exposure
- [ ] A reverse proxy (Traefik, Nginx) fronts multi-service HTTP traffic rather than exposing every
      backend port directly
