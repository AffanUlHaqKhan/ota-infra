# ota-infra

Glue for the other five repos: deployment, CI pipelines, signing keys, device provisioning,
disaster recovery. No product code lives here.

Kept separate because it holds the things that are cross-cutting and dangerous — the pipeline
that can ship to every device, and the procedures around the keys that make that possible.

---

## Layout

```
compose/      full-stack local bring-up (server + repo + db + web UI)
ci/           pipelines: build → test → sign → publish → canary → promote
keys/         key procedures and PUBLIC certs only — never private keys
terraform/    cloud resources: object storage, CDN, database, DNS, certificates
docs/         provisioning, key rotation, disaster recovery, runbooks
```

---

## Local full stack

```bash
docker compose -f compose/full-stack.yml up --build
```

Expects the sibling repos checked out next to this one. Brings up API, nginx repo host,
postgres and the web UI, and initialises an empty archive repo on first run.

This is the environment for developing the server and UI without touching hardware. Combine it
with a QEMU device image from `meta-ota-platform` for a complete loop on one laptop.

---

## CI pipeline

[ci/build-and-publish.yml](ci/build-and-publish.yml). Stages:

```
build    bitbake ota-image                     self-hosted runner
test     boot in QEMU, agent + HMI smoke test
sign     sign commit and target metadata       signing key from CI secret store
publish  push-ostree.sh → object storage, POST /api/v1/targets
canary   POST /api/v1/rollouts {group: canary}
promote  MANUAL APPROVAL → staged production rollout
```

Three rules that make or break this pipeline:

1. **`SSTATE_DIR` and `DL_DIR` on persistent shared storage.** A cold Yocto build is hours.
   With a warm sstate cache it is minutes. Without this the pipeline is theatre — nobody will
   run it on every commit.

2. **Production promotion is a human gate.** Automated canary is fine; automated fleet-wide
   deployment is how a single bad merge becomes a recall. GitHub Actions `environment:` with
   required reviewers, or the equivalent.

3. **The hardware rollback test gates releases.** QEMU cannot test the bootloader path, and the
   bootloader path is the safety net. A self-hosted runner with one real device, deliberately
   shipped a broken commit, must recover unattended before any release goes out.

### Secrets the pipeline needs

| Secret | Used by | Notes |
|---|---|---|
| `OSTREE_GPG_PRIVATE_KEY` / `OSTREE_GPG_KEY_ID` | sign | commit signing key — *not* the root key |
| `OTA_PUSH_TOKEN` | publish, canary | operator-scoped API token; must not be able to promote to production |
| `OTA_RSYNC_TARGET` + ssh key | publish | write access to the repo storage |

Scope them tightly. The publish token needs `targets:create` and `rollouts:create` on the
canary group, nothing else.

---

## Signing

Two layers, both needed, protecting against different attackers.

### 1. OSTree commit signing
GPG or ed25519 signature over the commit. The agent verifies before deploy. Stops a
compromised mirror or a MITM from serving modified content.

Does **not** stop a compromised server from serving an old, vulnerable, *validly signed*
commit — a rollback attack.

### 2. Target metadata signing (TUF / Uptane)
Signed metadata with freshness guarantees. Four roles:

| Role | Key location | Signs |
|---|---|---|
| root | offline, hardware token, two physical copies | the other roles' keys |
| targets | CI signer | which commits are valid targets |
| snapshot | CI signer | consistent set of metadata |
| timestamp | online, on the server | freshness — short expiry, frequently re-signed |

This stops rollback, freeze and mix-and-match attacks. Uptane extends it with a second
independent metadata chain (director + image repo) so a compromised backend cannot direct a
specific device to a wrong-but-signed image — the automotive standard, and required by UNECE
R156 if that applies.

Tier choice is `DECISIONS.md` D5. Tier 1 alone is a prototype.

### Rules

- **No private key ever enters this repo.** `.gitignore` blocks the obvious patterns; that is a
  safety net, not a policy.
- Root key offline, on hardware, backed up in two physical locations. If it lives only on a
  laptop, that laptop is the product's single point of total compromise.
- Rotation procedure written down and *rehearsed* before shipping. Rotating a key for the first
  time during an incident is not a plan.
- `keys/` holds procedures and public certs. That is all.

---

## Device provisioning

Each device needs a unique identity before it can talk to the API.

1. **Key generation on-device** at first boot — into a TPM/secure element if the SoC has one,
   otherwise a file. The private key should never leave the device.
2. **CSR to a provisioning service**, authenticated by something factory-specific: a
   pre-shared bootstrap token burned after first use, a manufacturing certificate, or an
   operator-attended station.
3. **Signed client cert returned**, stored in `/var/lib/ota-agent/` — outside the OSTree
   commit, so images stay identical across the fleet and identity survives every update.
4. **Server registers the device**, assigns a default group and branch.

Absolute rule: **no shared client certificate in the image.** One extracted device credential
would otherwise impersonate the entire fleet. This is the most common serious mistake in
homegrown OTA systems.

Open questions in `DECISIONS.md` D6: self-run CA vs managed PKI, hardware-backed keys, and how
a device re-provisions after a mainboard swap.

---

## Cloud resources (`terraform/`)

Minimum production shape:

- Object storage (S3/GCS/MinIO) for the OSTree repo, with a CDN in front. Objects are immutable
  and content-addressed, so aggressive caching is correct; only `refs/` and `summary` need
  short TTLs.
- Managed postgres for the API.
- Load balancer terminating TLS, with a second listener doing mTLS for device routes.
- DNS + certificates. Note the device CA is *separate* from the public server certificate.

---

## Disaster recovery

Write these as runbooks in `docs/` and test them, in this order of severity:

| Loss | Impact | Recovery |
|---|---|---|
| OSTree repo | devices cannot update; running devices unaffected | restore from backup — it is a plain directory of immutable objects, `rsync` back |
| Database | rollout state, groups, history lost | devices keep working; rebuild inventory from heartbeats as they check in |
| API host | no updates until restored | stateless, redeploy |
| **Root signing key** | **every device must be re-provisioned by hand** | there is no remote recovery — this is the truck-roll scenario |

The asymmetry is the point: everything except the root key is recoverable from a backup. Treat
key custody with proportionally more paranoia than server uptime.

Also worth documenting: what a technician does with a device that is bricked past rollback
(`DECISIONS.md` D13), and how they authenticate to do it.
