---
name: docker-container-security-hardening
description: Secures the Docker container lifecycle end to end - generating an SBOM (Syft), scanning images for CVEs (Trivy, Docker Scout, Snyk), signing and verifying images with Cosign/Sigstore, hardening running containers (non-root USER, --cap-drop ALL, --read-only rootfs, --no-new-privileges, seccomp/AppArmor profiles, resource limits), and managing secrets (Docker/Swarm secrets, Compose secrets, BuildKit build-time secret mounts, external vaults) instead of baking them into images or environment variables. Use when the user asks "scan my Docker image for vulnerabilities," "sign a Docker image," "run a container as non-root," "docker run --cap-drop," "read-only container filesystem," "how do I pass a secret into a container," "generate an SBOM," "docker secret create," or "harden my Dockerfile for production." Distinct from podman-image-signing-sigstore-trust (the same Cosign/Sigstore mechanics from the Podman side) and docker-dockerfile-build-optimization (build-time layer/size concerns, not runtime security posture).
---

# Docker Container Security Hardening

Container security spans the whole lifecycle: know what's *in* your image (SBOM), verify it
hasn't been tampered with (signing), catch known vulnerabilities before they ship (scanning),
then constrain what a container can actually do at runtime even if something in it is
compromised (hardening) — plus keep credentials out of the image and layers entirely (secrets).
No single layer is sufficient alone; treat this as defense in depth.

## Know what's in the image: SBOM

A Software Bill of Materials is a machine-readable inventory of every package/library in an
image — required by many compliance regimes, and the prerequisite for answering "are we
affected by CVE-X" without re-scanning everything from scratch.

```bash
# Syft (Anchore) generates SBOMs in standard formats
syft nginx:latest -o cyclonedx-json > nginx-sbom.json
```

## Verify provenance: signing with Cosign/Sigstore

An unsigned or tampered image pulled from a registry is one of the most common supply-chain
risks — nothing stops a compromised registry account (or a malicious mirror) from swapping in a
lookalike image under the same tag.

```bash
cosign generate-key-pair                       # produces cosign.key (private) + cosign.pub (public)
cosign sign --key cosign.key <registry>/<image>:<tag>     # image must already be pushed
cosign verify --key cosign.pub <registry>/<image>:<tag>
```
Cosign also supports keyless signing via OIDC and KMS-backed keys for teams that don't want to
manage a static private key file. Signing alone isn't enforcement — pair it with a policy
admission controller (Kubernetes Gatekeeper, Connaisseur, or a registry with built-in content
trust like Harbor) that actually *rejects* unsigned images at deploy time, or the signature is just
metadata nobody checks.

## Scan for known vulnerabilities

```bash
docker scout cves <image>:<tag>          # bundled with Docker Desktop
trivy image <image>:<tag>                # popular standalone scanner
snyk test --docker <image>:<tag>         # commercial, has a free tier
```
Scanning is not a one-time gate — a library that's clean today can have a CVE published against it
next week with no change to your image at all. Re-scan actively-deployed images on a schedule,
not only at build time. When a scan turns up a critical/high finding, the fix is almost always
"rebuild against a newer base image tag" rather than patching in place:
```bash
# before: python:3.8.14-slim-buster (has known CVEs)
# after:
FROM python:3.14-slim
```
then rebuild, re-scan, and roll out via a normal rolling update — the application code hasn't
changed, only the base layer.

## Runtime hardening

Layer these together; each closes a different avenue of attack, and none of them by itself is
sufficient.

**Run as a non-root user.** Add a dedicated user in the Dockerfile rather than relying on
`--user` overrides at run time:
```dockerfile
RUN groupadd -r appuser && useradd -r -g appuser appuser
COPY --chown=appuser:appuser . /app
USER appuser
```
A process that escapes the application logic (e.g., via a deserialization bug) inherits only that
user's privileges — not root — inside the container.

**Drop Linux capabilities you don't need.**
```bash
docker run --rm --cap-drop ALL --cap-add NET_BIND_SERVICE myapp:latest
```
`--cap-drop ALL` strips everything first; add back only the specific capability the process
genuinely requires (e.g., `NET_BIND_SERVICE` to bind a port below 1024 without running as root).

**Prevent privilege escalation.**
```bash
docker run --rm --no-new-privileges myapp:latest
# macOS alternative: --security-opt no-new-privilege:true
```
Blocks a process from gaining privileges beyond what it started with, even via a setuid binary or
kernel exploit path.

**Mount the root filesystem read-only, and only the specific paths that need to be writable.**
```bash
docker run --rm --read-only \
  -v /usr/src/app/logs \
  -v /usr/src/app/tmp \
  myapp:latest
```
Any write attempt outside the explicitly writable mounts fails outright — a strong constraint
against a compromised process dropping a backdoor or tampering with application files.

**Use minimal base images.** `scratch` or distroless images (no shell, no package manager)
shrink the attack surface directly — there's simply less present to exploit or pivot through. This
is the same motivation as multi-stage builds (see docker-dockerfile-build-optimization):
build-time tooling never needs to exist in the runtime image at all.

**Apply kernel-level constraints beyond the defaults.**
```bash
docker run --rm \
  --security-opt seccomp=default.json \
  --security-opt apparmor=my-profile \
  myapp:latest
```
Docker ships a default seccomp profile that already blocks many dangerous syscalls; a custom
profile narrows further for a specific app's real syscall surface. AppArmor/SELinux profiles add
mandatory access control on top. Also worth setting explicitly: `--icc=false` at the daemon level
to stop containers reaching each other by default (see docker-networking-bridge-dns-reverse-proxy
for network-boundary isolation instead), and never run with `--privileged` unless there is
genuinely no alternative.

**Cap resource usage** so a single runaway or compromised container can't starve the host or
serve as a denial-of-service vector: `--memory`, `--cpus`, `--pids-limit`, `--ulimit`.

**A representative hardened `docker run`, all together:**
```bash
docker run --rm \
  -p 4567:4567 \
  --read-only \
  --cap-drop ALL --cap-add NET_BIND_SERVICE \
  --no-new-privileges \
  --memory 256m --cpus 0.5 \
  myapp:latest
```
Every one of these is a tradeoff against flexibility or debuggability (a read-only rootfs breaks an
app that assumes it can write anywhere; a minimal image has no shell to `exec` into for
debugging). Apply them deliberately, verify the app still works under each constraint, and lean
on a debug-only sidecar image rather than loosening the production image's hardening.

**Add a `HEALTHCHECK`** so the container is *detected* as unhealthy and restarted, not just
hardened against attack:
```dockerfile
HEALTHCHECK --interval=30s --timeout=5s --start-period=5s --retries=3 \
  CMD curl -f http://localhost:4567/ || exit 1
```

## Runtime security monitoring

Static scanning and hardening only guard against *known* threats and *anticipated* misuse.
Runtime security tools (Falco is the standard open-source option) observe actual syscall/file/
network behavior inside running containers and alert on anomalies — privilege escalation
attempts, unexpected outbound connections, unauthorized file writes — that only manifest while
the container is live.

## Secrets: never bake them into an image or plain environment variable

`ENV DB_PASSWORD=SuperSecret123` is a real and common mistake — images are cached, shared,
and inspectable; anyone who can `docker history`/pull the image recovers the secret, environment
variables leak into logs and process listings, and rotating the secret means rebuilding and
redeploying everywhere it's baked in. Public-image scans have repeatedly found meaningful
percentages of images with accidentally leaked credentials — treat this as a real, common failure
mode, not a hypothetical.

**Docker Swarm secrets** (first-class, encrypted at rest and in transit, scoped to services that
request them):
```bash
docker swarm init
docker secret create db_password db_password.txt
docker service create --name myapp --secret db_password \
  -e DB_PASSWORD_FILE=/run/secrets/db_password myapp:latest
```
The app reads the file at `/run/secrets/db_password`, never an environment variable. Rotating a
secret means creating a new one and swapping it in — Swarm does not support in-place secret
edits:
```bash
docker service update --secret-rm db_password \
  --secret-add source=db_password_v2,target=db_password myapp
```
Only works for `docker service` (Swarm mode) — standalone `docker run` containers can't consume
Swarm secrets at all.

**Compose secrets** (lighter-weight, no rotation/encryption-at-rest/fine-grained access control —
see docker-compose-multi-service-development for the file syntax). Good enough for local dev;
not a substitute for Swarm secrets or a real vault in production.

**Build-time secrets** — for credentials the *build* needs (private registry/artifact tokens) without
persisting them in any image layer, use BuildKit's `RUN --mount=type=secret` (see
docker-dockerfile-build-optimization for the full syntax) rather than `--build-arg`.

**External secret managers** (HashiCorp Vault, AWS/Azure/GCP secret managers) for anything
beyond single-host Docker: store centrally with RBAC and audit trails, fetch via an authenticated
call (OIDC/IAM role) at container startup or through a sidecar, and rotate centrally without
touching the image at all.

## Common pitfalls

- **Treating a vulnerability scan as a one-time build-time gate** — new CVEs get published
  against unchanged images constantly; re-scan deployed images on a schedule.
- **Signing images without an enforcement point** — a signature nobody verifies at deploy time is
  just metadata; wire up actual policy enforcement (admission controller or registry-level content
  trust).
- **Running as root because "it's the image default"** — most application processes never need
  root inside the container; add a dedicated `USER` in the Dockerfile.
- **Reaching for `--privileged` to fix a permissions error** — nearly always the wrong fix; grant
  the one specific capability actually needed with `--cap-add` instead.
- **Baking secrets into `ENV`, `ARG`, or a committed file** — persists in image history/layers
  indefinitely, survives long after the secret is "rotated" elsewhere.
- **Using Docker/Compose secrets and assuming Swarm-level guarantees** — Compose secrets lack
  encryption-at-rest, rotation, and audit; know which tier of secrets management you're actually
  getting.
- **Mounting `/var/run/docker.sock` into a container without recognizing it as root-equivalent
  access to the whole host's Docker daemon** — treat it with the same caution as handing out root.
- **Over-hardening without testing** — a read-only rootfs or a minimal distroless image that
  breaks the app in ways only discovered in production is worse than a slightly softer, verified
  configuration.

## Checklist

- [ ] An SBOM is generated and kept for every image that ships to production
- [ ] Images are signed (Cosign/Sigstore) and something actually verifies the signature before
      deployment, not just at push time
- [ ] Images are scanned for CVEs both at build time and on a recurring schedule against
      already-deployed tags
- [ ] The Dockerfile creates and switches to a non-root `USER`
- [ ] Runtime flags drop unnecessary capabilities (`--cap-drop ALL` plus a minimal `--cap-add`
      allowlist), prevent privilege escalation (`--no-new-privileges`), and constrain resources
      (`--memory`, `--cpus`, `--pids-limit`)
- [ ] The root filesystem runs read-only wherever the app allows it, with only the specific
      writable paths it needs explicitly mounted
- [ ] No secret exists in `ENV`, `ARG`, a committed file, or a build layer — secrets flow through
      Docker/Swarm secrets, Compose secrets, BuildKit secret mounts, or an external vault
      appropriate to the deployment's scale
- [ ] A `HEALTHCHECK` is defined so failures are detected and recovered from, not just prevented
