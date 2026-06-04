# SRA Dev Demo — Secure Remote Access stack (Apache Guacamole)

**Created:** 2026-06-04
**Namespace:** `sra-dev-demo`
**Hostname:** `sra-demo.local.safeqbit.com`
**Path in repo:** `apps/safeqbit-local-hq/sra-dev-demo/`

A self-hosted, identity-aware secure remote access (SRA) stack for RDP / VNC /
SSH, built to mimic the capabilities of commercial OT SRA tools (e.g. Cyolo):
identity-aware access, session recording, supervised/live access, and credential
vaulting + injection — all integrated with the existing Authentik IdP.

The engine is **Apache Guacamole 1.6.0** (clientless HTML5 gateway). It was
chosen over Teleport because Teleport's OIDC/SAML SSO is Enterprise-only (OSS
Teleport only does GitHub SSO), which would break the Authentik requirement.

---

## How this maps to a Cyolo-style SRA tool

| OT SRA capability        | How it's delivered here                                                                 | Status |
|--------------------------|-----------------------------------------------------------------------------------------|--------|
| Identity-aware access    | OIDC → Authentik for authn; Guacamole JDBC RBAC + Authentik **groups → user groups** decide which connections each identity can even see | ✅ |
| Session recording        | guacd records every RDP/VNC/SSH session to a shared volume; replayable in-browser       | ✅ |
| Supervised / live access | Native session **sharing** (read-only observe / interactive collaborate) + active-session shadowing | ✅ |
| Credential vaulting + injection | Per-connection creds stored in the Guacamole Postgres DB and **injected by guacd** at connect time; the user never sees them | ✅ (DB-level) |
| Request → approve (JIT)  | **Not native to Guacamole.** See "Known gaps" below                                     | ⚠️ |

### Known gaps vs. Cyolo (be honest about these in the demo)

- **No just-in-time request→approve workflow.** Guacamole has static RBAC, not a
  "user requests access → approver grants a time-boxed session" flow. The closest
  approximations are (a) Authentik group-gated access (membership decides what's
  visible) and (b) a supervisor live-shadowing/sharing an in-progress session.
  A true JIT approval gate would need a custom layer in front (deferred).
- **Credential vaulting is DB-level, not a dedicated vault.** True external
  vaulting (HashiCorp Vault `${HASHIVAULT:...}` injection) is **not in any
  released Guacamole** — the HashiCorp token handler is merged but unreleased,
  targeting a future 1.7.0. Guacamole 1.6.0's vault extension supports Keeper
  Secrets Manager (KSM, commercial SaaS) only. Revisit when 1.7.0 ships; until
  then the encrypted DB store plays the vault role and demonstrates injection.

---

## Architecture / components

```
browser ──https──> nginx ingress (sra-demo.local.safeqbit.com)
                        │
                        ▼
                  guacamole (Tomcat webapp)  ──4822──>  guacd (proxy)
                        │                                   │
                   5432 │ (JDBC: users, conns, creds,       │ RDP 3389
                        │  history, sharing)                │ VNC 5900   ──> OT/IT targets
                        ▼                                   │ SSH  22
                  CNPG Postgres (guacamole-cnpg)            │
                        ▲                                   ▼
                        └───────── shared RWX recordings volume (nfs-truenas)
                                   (guacd writes, webapp replays)
```

- **guacd** — speaks the actual remote protocols to targets, performs credential
  injection and session recording. Initiates **outbound** connections to targets
  on 3389/5900/22, so the OT network must be routable from the pod network.
- **guacamole** — HTML5 UI, JDBC auth/RBAC + credential store, OIDC, replay player.
- **CNPG Postgres** (`guacamole-cnpg`, `instances: 1`) — the credential vault and
  system of record. Single instance because this is a demo (see
  `cnpg-strategy.md` for instance-count rationale).
- **Recordings PVC** (`guacamole-recordings`, `nfs-truenas`, RWX) — shared so guacd
  can write and the webapp can replay.

### Files

| File | Purpose |
|------|---------|
| `01-namespace.yaml` | namespace + velero biweekly label |
| `02-cnpg-cluster.yaml` | Postgres (credential vault / system of record) |
| `03-cnpg-scheduled-backup.yaml` | weekly volume-snapshot backup |
| `04-db-init-job.yaml` | one-shot, idempotent Guacamole schema loader |
| `05-pvcs.yaml` | RWX recordings volume |
| `06-guacd-deployment.yaml` | proxy daemon + service |
| `07-guacamole-deployment.yaml` | webapp + config ConfigMap + service |
| `08-ingress.yaml` | nginx + letsencrypt-prod, long WebSocket timeouts |

No SealedSecret is needed: the OpenID **implicit** flow carries no client secret,
and the Postgres password is auto-minted by the CNPG operator
(`guacamole-cnpg-app` secret, `password` key).

---

## Activation checklist

Deployment is GitOps — nothing is live until committed and Flux reconciles
(`apps` Kustomization has `prune: false`).

1. **Create the Authentik OIDC provider/application** (see next section) and note
   the **Client ID**.
2. **Fill the Client ID** into `07-guacamole-deployment.yaml` → ConfigMap
   `guacamole-config` → `OPENID_CLIENT_ID` (replace `REPLACE_WITH_AUTHENTIK_CLIENT_ID`).
   Confirm the application **slug** matches the `OPENID_JWKS_ENDPOINT` /
   `OPENID_ISSUER` URLs (default assumes slug `guacamole`). The `reloader`
   annotation restarts the pod automatically when the ConfigMap changes.
3. **Commit & push.** Flux applies the namespace, CNPG cluster, schema-init Job,
   then guacd + webapp. The init Job waits for Postgres and is a no-op once the
   schema exists.
4. **DNS:** ensure `sra-demo.local.safeqbit.com` resolves to the ingress (same as the
   other `*.local.safeqbit.com` hosts).
5. **Verify** (see below), then log in.

### Verify

```sh
kubectl -n sra-dev-demo get pods
kubectl -n sra-dev-demo get cluster guacamole-cnpg          # CNPG ready, 1/1
kubectl -n sra-dev-demo logs job/guacamole-db-init          # "Schema loaded." or "already present"
kubectl -n sra-dev-demo get ingress sra                     # address assigned
kubectl -n sra-dev-demo get certificate sra-tls             # READY=True
```

---

## Authentik OIDC provider setup

In the Authentik admin UI (`authentik01.local.safeqbit.com`):

1. **Providers → Create → OAuth2/OpenID Provider**
   - Name: `Guacamole SRA`
   - Authorization flow: your default explicit-consent (or implicit) flow
   - **Client type: Public** (implicit flow — Guacamole's openid extension uses
     `response_type=id_token`, so no client secret)
   - **Redirect URIs (exact match):** `https://sra-demo.local.safeqbit.com/`
     (include the trailing slash)
   - Signing key: your default
   - Scopes: `openid`, `email`, `profile`. To drive identity-aware access by
     group, also include a scope/property mapping that emits a **`groups`** claim.
2. **Applications → Create**
   - Name: `Guacamole SRA`, **Slug: `guacamole`** (must match the slug in the
     `OPENID_JWKS_ENDPOINT` / `OPENID_ISSUER` URLs in the ConfigMap)
   - Provider: the one just created
3. Copy the provider's **Client ID** → ConfigMap step above.
4. Bind the application to the Authentik groups/users who should reach the portal.

The resulting endpoints (slug `guacamole`):
- authorize: `https://authentik01.local.safeqbit.com/application/o/authorize/`
- jwks: `https://authentik01.local.safeqbit.com/application/o/guacamole/jwks/`
- issuer: `https://authentik01.local.safeqbit.com/application/o/guacamole/`

---

## First login & break-glass admin

The schema (`002-create-admin-user`) seeds a local admin: **`guacadmin` /
`guacadmin`**.

1. First, log in *locally* as `guacadmin` (the OIDC login button is also present).
   **Change the password immediately.** Keep this account as break-glass only.
2. Log in once via Authentik so your OIDC identity is auto-created in the DB
   (`POSTGRESQL_AUTO_CREATE_ACCOUNTS=true`).
3. As `guacadmin`, grant your OIDC user admin (Settings → Users), or — better —
   grant permissions to a **user group** whose name matches an Authentik group
   so membership flows in automatically (`OPENID_GROUPS_CLAIM_TYPE=groups`).

---

## Configuring a connection (with credential injection + recording)

Settings → Connections → New connection:

- **Protocol:** RDP / VNC / SSH
- **Network:** target hostname/IP + port (reachable from the cluster)
- **Authentication:** enter the target username/password here. These are the
  **vaulted, injected credentials** — stored in the DB and fed to the target by
  guacd; the end user connecting through the portal never sees them.
- **Recording** (for session recording + replay):
  - Recording path: `/recordings`
  - Recording name: `${HISTORY_UUID}` (required for the in-UI player to find it)
  - Enable "Automatically create recording path"
- Grant the connection to the appropriate **user group** (identity-aware access).

### Supervised / live access (sharing & shadowing)

- On a connection, define a **Sharing Profile** (read-only "observe" or
  interactive "collaborate"). During an active session the user can generate a
  share link a supervisor opens to watch/join live.
- Admins can also see **active sessions** (Settings → Active Sessions) and join or
  kill them — the supervisor shadowing path.

### Replaying a recorded session

Settings → History → pick a session with a recording → play in-browser. The
webapp reads from `RECORDING_SEARCH_PATH=/recordings` (the shared RWX volume).
To export to video offline: `guacenc /recordings/<file>` produces an `.m4v`.

---

## Backups

- **CNPG:** weekly Longhorn volume snapshot (`03-cnpg-scheduled-backup.yaml`,
  Sun 03:00 UTC), retention via `configs/cnpg-backup-retention.yaml`.
- **Velero → B2:** `configs/velero-schedule-sra-dev-demo.yaml`, bi-monthly (8th &
  23rd 04:30 UTC). Captures the `guacamole-recordings` RWX PVC + k8s objects.
  **Retention: keep last 2** restore points — Velero has no native "keep N"
  count, so this is expressed as `ttl: 672h` (28 days, longer than the ~15-day
  interval so the prior backup survives, shorter than two intervals so the one
  before that expires → steady state of 2). Namespace also labelled
  `backup.policy/velero: biweekly`.
- See `backup-strategy.md` for the 3-layer model.

---

## Upgrades & troubleshooting

- **Version bump:** change the `1.6.0` tag in all three places (`guacd`,
  `guacamole`, and the init Job's `generate-schema` initContainer). If the schema
  changed between versions, apply Guacamole's `upgrade/` SQL scripts manually and
  delete the `guacamole-db-init` Job so it can re-run.
- **Init Job stuck:** it retries until Postgres is ready (`backoffLimit: 10`).
  Check `kubectl -n sra-dev-demo logs job/guacamole-db-init`. CNPG must be `1/1`.
- **OIDC redirect errors:** the Authentik redirect URI must exactly equal
  `https://sra-demo.local.safeqbit.com/` (trailing slash), and the app slug must match
  the JWKS/issuer URLs.
- **Session drops after ~60s of idle:** check the ingress proxy timeout
  annotations (set to 3600s here).
- **guacd can't reach a target:** verify pod-network → OT-network routing/firewall
  for 3389/5900/22; test with `kubectl -n sra-dev-demo exec deploy/guacd -- nc -vz <host> <port>`.
- **Recording won't play:** confirm the connection's recording name is
  `${HISTORY_UUID}` and both pods mount `guacamole-recordings`.

Related: `cnpg-strategy.md`, `authentik-migration.md`, `backup-strategy.md`.
