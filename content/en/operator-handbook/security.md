---
title: "Security"
weight: 200
---

# Security {#security}

Three layers, each with its own posture :

- **RBAC** — group-based, scope-aware, verb-typed access control.
  Caller identity from OIDC ; ACL checks in the gRPC interceptor.
- **OIDC** — identity provider (dex by default), JWT-based tokens,
  per-DC JWKs caching so token validation never crosses a DC.
- **TLS** — agent-to-agent gRPC over mTLS, ACME-driven TLS at the
  proxy plane, etcd peer-to-peer over mTLS.

## Canonical runbook — RBAC {#canonical-runbook-rbac}

[**weft/docs/operations/rbac.md →**](https://github.com/openweft/weft/blob/main/docs/operations/rbac.md)

Covers the model :

- Group / scope / verb / resource matrix.
- How `*Caller` is built from OIDC claims in `auth.go`.
- ACL check pattern in handlers, with examples from `acl.go`.
- How to add a new ACL check without re-deriving the convention.

## OIDC posture {#oidc-posture}

- **Provider** : `dex` (deployed by `weft infra bootstrap`). Backed by
  a small SQLite store ; backups follow the per-volume path in
  [Backup & restore]({{< relref "backup-and-restore.md" >}}).
- **Issuer** : the dashboard runs an OIDC handler ; the CLI acquires
  tokens via `weft login` and caches them under
  `$XDG_CONFIG_HOME/weft/credentials.json` (per-user, mode 0600).
- **Claims used** : `groups` (for RBAC group lookup), `email` (for
  audit logging), `sub` (for stable user id).
- **Token validation** : every agent caches dex's JWKs locally with a
  short TTL ; a token presented to any agent can be validated
  locally, no cross-DC round-trip.
- **External IdP** : dex federates to your existing IdP (Okta / Azure
  AD / Google Workspace / Keycloak / GitHub OAuth). Configure via
  the dex `connectors` block in `cluster.hcl` ; see the dex docs.

To smoke-test the OIDC flow end-to-end :

```
$ weft login                  # opens browser, completes OIDC dance
$ weft project ls             # exercises a real authenticated RPC
$ weft admin whoami           # prints the *Caller as the server sees it
```

## TLS posture {#tls-posture}

| Surface                                  | TLS terminator             | Cert source                                              |
| ---------------------------------------- | -------------------------- | -------------------------------------------------------- |
| Agent-to-agent gRPC                      | weft-agent                 | mTLS ; certs minted at `weft up` time, rotated by agent. |
| Tenant ingress (HTTP)                    | Caddy (in weft-agent)      | ACME via Let's Encrypt or your internal ACME server.     |
| etcd peer + client                       | embedded etcd              | mTLS ; certs minted at `weft up` time.                   |
| Dashboard (HTTP)                         | Caddy (in weft-agent)      | Same ACME path as tenant ingress.                        |
| NATS                                     | embedded                   | mTLS ; same CA as agent-to-agent.                        |

The cluster runs its own CA for internal mTLS, generated at
`weft up` time and stored in etcd. Cert rotation is automatic ;
operators don't manage internal certs manually.

For the proxy plane's ACME state and the cross-host cert-sharing
recipe (avoiding ACME bursts when multiple agents could issue for the
same domain), see the [proxy runbook]({{< relref "proxy.md" >}}).

## Hardening checklist {#hardening-checklist}

| Item                                                                                | Where                                                                                                                          |
| ----------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------ |
| Run `weft-agent` as the `weft` user (not root) with minimal ambient caps.           | systemd unit shipped by cloud-init ; see [cloud-init runbook](https://github.com/openweft/weft/blob/main/docs/operations/cloud-init.md). |
| Restrict `/run/weft/agent.sock` to the operator group via systemd `SocketUser`.     | Same systemd unit.                                                                                                             |
| Bind the gRPC TCP listener to the management network, not `0.0.0.0`.                | `agent_config { grpc_listen = "10.0.0.1:7443" }` in `cluster.hcl`.                                                             |
| Use a strict OIDC group-to-RBAC mapping ; default-deny rather than default-allow.   | dex `staticClients` + RBAC config ; see runbook above.                                                                         |
| Rotate the cluster CA before its expiry (default : 10 years from `weft up`).        | `weft admin ca rotate` (roadmap ; today manual via etcd `etcdctl put`).                                                        |
| Backup credentials separately from etcd (dex secret keys must survive a restore).   | Per-volume snapshot of dex's state dir.                                                                                        |

## Audit logging {#audit-logging}

Every RPC server-side passes through the auth interceptor ; the
*Caller is logged with the RPC name and decision (allow / deny) at
INFO level. Forward agent logs to your SIEM via the systemd journal
exporter of your choice.

## Guest secrets {#guest-secrets}

A microVM fetches its secrets (a CubeFS key pair, a WireGuard private key)
over the guest-only vsock server with `weft.guest.v1.GuestSecrets/Fetch`
(weft v0.10.0, weft-proto v0.24.0), never from the pod spec. The pod spec
carries only `*_ref` fields holding a secret ID.

- **Identity.** The caller is the CID the host bound at StartVM, mapped to
  its pod; the guest's own claims are never read. When the host holds a
  per-boot token for the pod it is compared in constant time (KVM holds none
  today, so identity there is CID-only; a presented token with none held is
  refused).
- **Authorization.** Each pod has a table in cluster state (`guestsecrets`)
  mapping the secret IDs it may fetch to a
  [secretsource](https://github.com/go-authn/secretsource) reference and a
  rotation version. The guest names an ID and never supplies a reference.
  Unknown IDs, forbidden IDs and unbound CIDs all return `PermissionDenied`.
  The table is removed when the pod spec is cleared or the VM is unregistered.
- **Request.** `secret_id` (`[a-z0-9][a-z0-9._-]{0,62}`), `boot_token` (empty
  or exactly 32 bytes), `known_version`. The response holds `value` (at most
  64 KiB), `version`, and `unchanged` (true, with no value, when
  `known_version` equals the current version).
- **Handling.** The reference is resolved at request time; nothing is cached
  or written to disk, and the response value is wiped after it is sent. The
  guest should clear it after use.
- **Limits.** 10 fetches per second and 600 per hour per VM, 10 s per call.
- **Audit.** One line per fetch with pod, CID, ID, version and outcome; never
  the value or the token.

## Cross-references {#cross-references}

- [Backup & restore]({{< relref "backup-and-restore.md" >}}) — dex state survival.
- [Observability]({{< relref "observability.md" >}}) — RPC-level audit metrics
  (`grpc_server_handled_total` by `grpc_code`).
- [HA & DR]({{< relref "ha-and-dr.md" >}}) — what stays available when an OIDC
  provider is the failed component.
- Canonical RBAC runbook : [weft/docs/operations/rbac.md](https://github.com/openweft/weft/blob/main/docs/operations/rbac.md).
