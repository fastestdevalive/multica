# Private Repository Support Plan

## Problem

Multica currently relies on the daemon host's system-level git configuration (SSH agent, credential helpers) to authenticate with git remotes. This means:

- Private repos only work if the daemon operator has manually configured SSH keys or credential helpers
- No per-workspace or per-repo credential management
- No way for workspace owners to add private repos through the UI
- Self-hosted users must configure credentials outside of Multica

## Goals

1. Allow workspace owners to configure credentials for private repositories via the UI
2. Support GitHub/GitLab personal access tokens (PATs) for HTTPS repos
3. Support SSH deploy keys for SSH-based repos
4. Encrypt credentials at rest
5. Inject credentials into git operations without leaking them to agent processes

## Architecture

### Credential Storage

Add a new `workspace_credential` table:

```sql
CREATE TABLE workspace_credential (
    id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    workspace_id UUID NOT NULL REFERENCES workspace(id) ON DELETE CASCADE,
    name        TEXT NOT NULL,              -- user-facing label, e.g. "GitHub PAT"
    cred_type   TEXT NOT NULL,              -- "pat" | "ssh_key" | "github_app"
    host_pattern TEXT NOT NULL,             -- glob match on repo host, e.g. "github.com"
    encrypted_value BYTEA NOT NULL,         -- encrypted credential payload (JSON)
    created_at  TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at  TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(workspace_id, name)
);
```

The `encrypted_value` column stores an AES-256-GCM encrypted JSON blob. The encryption key is derived from a server-level secret (`MULTICA_CREDENTIAL_KEY` env var) using HKDF with the credential ID as context, so each row has a unique derived key.

Payload schemas by `cred_type`:

- **pat**: `{"token": "ghp_xxx"}`
- **ssh_key**: `{"private_key": "-----BEGIN OPENSSH...", "passphrase": ""}`
- **github_app**: `{"app_id": 12345, "installation_id": 67890, "private_key": "-----BEGIN RSA..."}`

### API Endpoints

Add to the existing handler layer:

| Method | Path | Description |
|--------|------|-------------|
| `POST` | `/workspaces/{id}/credentials` | Create a credential |
| `GET` | `/workspaces/{id}/credentials` | List credentials (values redacted) |
| `DELETE` | `/workspaces/{id}/credentials/{cred_id}` | Delete a credential |
| `PUT` | `/workspaces/{id}/credentials/{cred_id}` | Update a credential |

Credentials are **never** returned in plaintext after creation. The list endpoint returns metadata only (name, type, host pattern, created/updated timestamps).

### Credential Injection

Credentials must be injected into git operations in `repocache/cache.go` without exposing them to the agent process.

#### For PATs (HTTPS)

Use git's `GIT_ASKPASS` mechanism:

1. Before running `git clone` or `git fetch`, the daemon resolves matching credentials for the repo URL by host pattern.
2. Write a short-lived askpass script to a temp file that echoes the token.
3. Set `GIT_ASKPASS=/tmp/multica-askpass-xxxx` and `GIT_TERMINAL_PROMPT=0` in the git command environment.
4. Delete the askpass script after the git operation completes.

This keeps the token out of the command line (avoiding `/proc` exposure) and out of the agent's environment.

#### For SSH Keys

1. Write the private key to a temp file with `0600` permissions.
2. Set `GIT_SSH_COMMAND="ssh -i /tmp/multica-key-xxxx -o StrictHostKeyChecking=accept-new"` in the git command environment.
3. Delete the key file after the git operation completes.

#### For GitHub App

1. Generate a short-lived installation access token via the GitHub API using the app's private key.
2. Use the resulting token as a PAT (same `GIT_ASKPASS` flow).

### Daemon Sync Changes

In `daemon.go`, the workspace sync loop already fetches workspace data. Extend the server response to include credential metadata (IDs + host patterns, **not** values). The daemon requests decrypted credential values only when needed for a git operation, via a new authenticated endpoint:

```
POST /internal/credentials/resolve
Body: {"workspace_id": "...", "repo_url": "..."}
Response: {"cred_type": "pat", "value": {"token": "ghp_xxx"}}
```

This endpoint is daemon-only (authenticated via the daemon's registration token) and returns the decrypted credential for a specific repo URL match.

### Agent Isolation

Agents must **never** see credentials:

- The `/repo/checkout` endpoint in `health.go` already runs git operations in the daemon process. No change needed — the agent calls the daemon, the daemon does the git work with injected credentials.
- Agent environment variables (`daemon.go`) remain unchanged — no git credentials are passed.
- The `GIT_ASKPASS` script and SSH key files are created and deleted within the daemon process, never written to the agent's working directory.

### Frontend Changes

Add a "Credentials" section to the workspace settings page:

1. **Credential list** — table showing name, type, host pattern, created date
2. **Add credential** — form with fields for name, type (dropdown), host pattern, and value (password input)
3. **Delete credential** — confirmation dialog
4. **Test credential** — optional: button that triggers a `git ls-remote` on a repo URL to verify access

Location: extend the existing workspace settings in `apps/web/` alongside the repos configuration.

### CLI Extension

Add credential management to the `multica` CLI for self-hosted users:

```
multica credential add --workspace <id> --name "GitHub" --type pat --host "github.com" --token "ghp_xxx"
multica credential list --workspace <id>
multica credential delete <cred-id>
```

## File Changes

| File | Change |
|------|--------|
| `server/migrations/0XX_workspace_credentials.up.sql` | New table |
| `server/pkg/db/queries/credential.sql` | sqlc queries |
| `server/internal/handler/credential.go` | New HTTP handlers |
| `server/internal/service/credential.go` | Business logic + encryption |
| `server/internal/daemon/repocache/cache.go` | Credential injection in `gitCloneBare`, `gitFetch` |
| `server/internal/daemon/repocache/askpass.go` | Askpass script helper |
| `server/internal/daemon/daemon.go` | Credential resolution during repo sync |
| `server/internal/handler/daemon.go` | New `/internal/credentials/resolve` endpoint |
| `server/cmd/multica/cmd_credential.go` | CLI commands |
| `apps/web/.../workspace-settings/` | Credentials UI section |

## Security Considerations

1. **Encryption at rest** — All credential values encrypted with AES-256-GCM. Server-level key required (`MULTICA_CREDENTIAL_KEY`).
2. **No plaintext in logs** — The existing `pkg/redact` package should be extended to catch common token patterns (`ghp_`, `glpat-`, `AKIA`).
3. **Temp file hygiene** — Askpass scripts and SSH key files are created with restrictive permissions and deleted immediately after use via `defer`.
4. **No agent exposure** — Credentials never appear in agent environment variables, working directories, or command-line arguments.
5. **Audit logging** — Credential creation, deletion, and usage should be logged (without values).
6. **Rotation** — Credentials can be updated in-place via the PUT endpoint. GitHub App tokens are inherently short-lived.

## Implementation Order

1. **Database migration + encryption service** — Foundation for credential storage
2. **API endpoints (CRUD)** — Server-side credential management
3. **Credential resolution endpoint** — Daemon can fetch decrypted credentials
4. **PAT injection in repocache** — `GIT_ASKPASS` flow for HTTPS repos
5. **SSH key injection in repocache** — `GIT_SSH_COMMAND` flow for SSH repos
6. **CLI commands** — Self-hosted user workflow
7. **Frontend UI** — Workspace settings integration
8. **GitHub App support** — Optional, adds ephemeral token generation
9. **Redaction + audit logging** — Harden against credential leaks

## Open Questions

- Should credentials be scoped per-repo or per-host? Per-host (via `host_pattern`) is simpler and covers most cases, but per-repo gives finer control. Starting with per-host and adding per-repo later is straightforward.
- Should we support OAuth flows (e.g., "Connect GitHub" button) in addition to manual PAT entry? This is a better UX but significantly more complex. Recommend deferring to a follow-up.
- For self-hosted deployments, should we support reading credentials from environment variables or a config file as an alternative to the database? This avoids the encryption key requirement for simple setups.
