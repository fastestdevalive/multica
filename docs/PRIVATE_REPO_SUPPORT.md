# Private Repository Support Plan

## Problem

Multica's `repocache.Cache` runs `git clone --bare` and `git fetch origin` as plain shell commands with no credential injection (`cache.go:158-174`, `cache.go:185-210`). Authentication only works if the daemon host happens to have SSH keys or a git credential helper configured at the OS level. This means:

- Private repos silently fail to clone — the daemon logs `clone failed` and moves on
- No per-workspace or per-repo credential management
- No way for workspace owners to add private repos through the UI
- Self-hosted users must configure credentials outside of Multica
- The hosted product cannot support private repos at all

## Goals

1. Allow workspace owners to configure credentials for private repositories via the workspace settings UI
2. Support GitHub/GitLab personal access tokens (PATs) for HTTPS repos
3. Support SSH deploy keys for SSH-based repos
4. Encrypt credentials at rest in the database
5. Inject credentials into daemon git operations without exposing them to agent processes
6. Keep the existing public-repo flow unchanged (zero config for public repos)

## Design Decisions

### Per-repo credentials (not per-host)

Credentials are attached directly to entries in the existing `workspace.repos` JSONB array rather than a separate table with host-pattern matching. Rationale:

- Simpler mental model: "this repo uses this token"
- Avoids ambiguity when multiple credentials match the same host
- Aligns with how the repos UI already works — each repo row gets an optional credential field
- Fewer moving parts: no new table, no matching logic, no separate CRUD endpoints

The tradeoff is that users with many repos on the same host must configure credentials per-repo. This is acceptable for the typical 1-5 repo workspace. If demand arises for host-level credentials, we can add them later without breaking the per-repo model.

### Credential storage: separate table (not inline in JSONB)

Even though credentials are *associated* per-repo, the encrypted values live in a dedicated `repo_credential` table keyed by `(workspace_id, repo_url)`. The `repos` JSONB continues to store only `{url, description}`. Reasons:

- The `repos` column flows through many code paths (daemon sync, agent context, frontend) — embedding secrets in it would require redaction everywhere
- A separate table makes encryption, access control, and audit logging straightforward
- The daemon resolves credentials at git-operation time via a targeted API call, not by receiving them in the workspace sync payload

### URL Normalization

Repo URLs must be normalized before storing or looking up credentials. Without normalization, `https://github.com/org/repo` and `https://github.com/org/repo.git` resolve to different rows, and a user who adds a repo with one form won't get their credential applied when the daemon uses the other form.

Normalization rules (applied in a `normalizeRepoURL(url string) string` helper):
1. Lowercase the scheme and host
2. Strip trailing `.git`
3. Strip trailing `/`
4. Strip default ports (`:80` for http, `:443` for https)

This function is called in three places: when writing a credential (PUT endpoint), when reading a credential (GET/DELETE endpoints and the resolve endpoint), and in `repocache/cache.go` before looking up credentials via the resolver.

The `UNIQUE(workspace_id, repo_url)` constraint stores normalized URLs only.

### Encryption

AES-256-GCM with a server-level key (`MULTICA_CREDENTIAL_KEY` env var). Each credential row uses a unique nonce. HKDF key derivation with the row ID as context provides per-row key isolation.

`MULTICA_CREDENTIAL_KEY` is **required**. The server refuses to start if it is missing or shorter than 32 bytes, returning a clear error: `MULTICA_CREDENTIAL_KEY must be set to a 32+ byte secret to enable credential storage`. There is no plaintext fallback — silent degradation would allow credentials to land in production databases without encryption.

## Architecture

### Database Schema

Migration `0XX_repo_credentials.up.sql` (number assigned at merge time — check `server/migrations/` for the next available number):

```sql
CREATE TABLE repo_credential (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    workspace_id    UUID NOT NULL REFERENCES workspace(id) ON DELETE CASCADE,
    repo_url        TEXT NOT NULL,
    cred_type       TEXT NOT NULL CHECK (cred_type IN ('pat', 'ssh_key')),
    encrypted_value BYTEA NOT NULL,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(workspace_id, repo_url)
);

CREATE INDEX idx_repo_credential_workspace ON repo_credential(workspace_id);
```

Payload schemas by `cred_type`:

- **pat**: `{"token": "ghp_xxx"}` — works for GitHub PATs, GitLab PATs, Bitbucket app passwords
- **ssh_key**: `{"private_key": "-----BEGIN OPENSSH PRIVATE KEY-----\n...", "passphrase": ""}` — deploy keys or user SSH keys

### API Endpoints

Added to the existing `handler` package, mounted under workspace middleware (inherits auth + membership check):

| Method | Path | Description |
|--------|------|-------------|
| `PUT` | `/workspaces/{id}/repos/{url}/credential` | Create or update a credential for a repo |
| `GET` | `/workspaces/{id}/repos/{url}/credential` | Check if a credential exists (returns `{exists: true, cred_type, created_at}`, never the value) |
| `DELETE` | `/workspaces/{id}/repos/{url}/credential` | Delete a credential |

The `url` parameter is the URL-encoded repo URL. Using PUT (upsert) avoids the need for separate create/update flows.

**Daemon-only endpoint** for credential resolution at git-operation time:

```
POST /internal/repo-credentials/resolve
Headers: Authorization: Bearer <daemon-token>
Body: {"workspace_id": "...", "repo_url": "..."}
Response: {"cred_type": "pat", "value": {"token": "ghp_xxx"}}
```

This endpoint is authenticated via the daemon's registration token (already used for task claims in `handler/daemon.go`). Returns 404 if no credential exists for the repo (public repo — no injection needed).

**Daemon-side caching:** The daemon caches resolved credentials in memory with a 5-minute TTL to avoid hammering the server on every `git fetch` during the 30-second sync loop. The cache is keyed by `(workspaceID, normalizedRepoURL)` and is invalidated on daemon restart. PATs and SSH keys don't rotate frequently, so 5 minutes is safe.

```go
type cachedCred struct {
    cred      *Credential
    fetchedAt time.Time
}

type credCache struct {
    mu    sync.Mutex
    items map[string]cachedCred // key: "wsID/normalizedURL"
    ttl   time.Duration
}
```

The `credCache` lives in the daemon's `CredentialResolver` implementation. On every resolve call it checks the cache first; only on a miss (or expired entry) does it call the server.

### Credential Injection in repocache

The `Cache` struct gains a `CredentialResolver` interface:

```go
// CredentialResolver fetches decrypted credentials for a repo URL.
// Returns nil, nil when no credential is configured (public repo).
type CredentialResolver interface {
    Resolve(ctx context.Context, workspaceID, repoURL string) (*Credential, error)
}

type Credential struct {
    Type  string          // "pat" or "ssh_key"
    Value json.RawMessage // type-specific payload
}
```

The resolver is injected into `Cache` at construction time. `daemon.go` provides an implementation that calls the server's `/internal/repo-credentials/resolve` endpoint.

#### Git operations with credentials

`gitCloneBare` and `gitFetch`/`runGitFetch` are updated to accept an optional `*Credential` parameter. When non-nil:

**For PATs (HTTPS repos):**

```go
func withPATCredential(cmd *exec.Cmd, token string) (cleanup func()) {
    // Write askpass script to a temp file:
    //   #!/bin/sh
    //   echo "$MULTICA_GIT_TOKEN"
    // Set env on the command:
    //   GIT_ASKPASS=<tempfile>
    //   GIT_TERMINAL_PROMPT=0
    //   MULTICA_GIT_TOKEN=<token>
    // Return a cleanup func that removes the temp file.
}
```

The token is passed via an environment variable on the child process (not on the command line, avoiding `/proc/*/cmdline` exposure). The askpass script simply echoes it. The temp file has `0700` permissions and is deleted immediately after the git command completes.

**For SSH keys:**

```go
func withSSHCredential(cmd *exec.Cmd, privateKey string) (cleanup func()) {
    // Write private key to temp file with 0600 permissions.
    // Set env on the command:
    //   GIT_SSH_COMMAND="ssh -i <tempfile> -o StrictHostKeyChecking=accept-new -o IdentitiesOnly=yes"
    // Return a cleanup func that removes the temp file.
}
```

`IdentitiesOnly=yes` prevents ssh-agent from offering other keys that might trigger rate limits.

#### Integration points in cache.go

1. **`Sync()` (line 66)** — Before cloning or fetching each repo, call `resolver.Resolve(ctx, workspaceID, repo.URL)`. Pass the result to the git operation. Sync needs a `context.Context` parameter added.

2. **`CreateWorktree()` (line 283)** — The `gitFetch` call at line 300 also needs credential injection. The resolver is already on the `Cache` struct.

3. **`gitCloneBare()` (line 158)** and **`runGitFetch()` (line 204)** — Updated signatures to accept `*Credential`, apply credential env vars to the `exec.Cmd`, and call cleanup after.

### Agent Isolation

Agents never see credentials. The security boundary is already clean:

- **`/repo/checkout` endpoint** (`health.go:87`) — Agents call this HTTP endpoint on the daemon. The daemon runs git operations in its own process with injected credentials. The agent receives only the worktree path and branch name.
- **Agent environment** (`daemon.go` task spawning) — No git credential env vars are passed to agent processes. The agent's git commands operate on the local worktree (already cloned), so no remote auth is needed for local operations.
- **Agent push** — Agents currently cannot push (no remote write credentials). If push support is needed later, it would go through a new daemon endpoint, maintaining the same isolation model.

### Frontend Changes

Extend the existing `RepositoriesTab` (`packages/views/settings/components/repositories-tab.tsx`) — each repo row gains an expandable credential section:

```
┌─────────────────────────────────────────────────────┐
│ https://github.com/org/private-repo         [🗑]    │
│ Our backend service                                  │
│ ┌─────────────────────────────────────────────────┐ │
│ │ Auth: PAT  ▾   Token: ••••••••••  [Update] [×] │ │
│ └─────────────────────────────────────────────────┘ │
├─────────────────────────────────────────────────────┤
│ https://github.com/org/public-repo          [🗑]    │
│ Design system                                        │
│   No credentials (public repo)    [+ Add auth]      │
└─────────────────────────────────────────────────────┘
```

Implementation:
- Credential state is managed per-repo within the `RepositoriesTab` component
- Saving credentials calls `PUT /workspaces/{id}/repos/{url}/credential` separately from the repos save (credentials are not part of the `repos` JSONB update)
- **Validate on save**: before persisting, the server runs `git ls-remote <repo_url>` with the provided credential. If it fails, the endpoint returns a 422 with the git error message. This catches typos and expired tokens immediately rather than silently failing on the next sync.
- Credential values are only shown during initial entry — after save, the UI shows a masked placeholder and the type label (e.g. "PAT configured")

New API methods in `packages/core/api/client.ts`:

```typescript
setRepoCredential(wsId: string, repoUrl: string, body: { cred_type: string; value: object }): Promise<void>
getRepoCredential(wsId: string, repoUrl: string): Promise<{ exists: boolean; cred_type?: string; created_at?: string }>
deleteRepoCredential(wsId: string, repoUrl: string): Promise<void>
```

### CLI Extension

For self-hosted users who configure repos via the CLI:

```bash
# Set a PAT for a repo
multica repo credential set <repo-url> --type pat --token "ghp_xxx"

# Set an SSH key for a repo  
multica repo credential set <repo-url> --type ssh_key --key-file ~/.ssh/deploy_key

# Check if a credential exists
multica repo credential get <repo-url>

# Remove a credential
multica repo credential delete <repo-url>
```

These are subcommands of `repo` (not a new top-level command) since credentials are tightly coupled to repos. Implementation in `server/cmd/multica/cmd_repo.go`.

### Redaction

Extend `server/pkg/redact/` to catch common token patterns in daemon logs:

- `ghp_[A-Za-z0-9]{36}` (GitHub PAT)
- `glpat-[A-Za-z0-9\-]{20}` (GitLab PAT)
- `github_pat_[A-Za-z0-9]{22}_[A-Za-z0-9]{59}` (GitHub fine-grained PAT)
- `AKIA[A-Z0-9]{16}` (AWS access key — defense in depth)

The daemon logger already uses the redact package; adding patterns there covers all log output.

## File Changes

| File | Change |
|------|--------|
| `server/migrations/0XX_repo_credentials.up.sql` | New `repo_credential` table (number at merge time) |
| `server/migrations/0XX_repo_credentials.down.sql` | Drop table |
| `server/pkg/db/queries/repo_credential.sql` | sqlc queries: upsert, get, delete, resolve |
| `server/internal/handler/repo_credential.go` | HTTP handlers for credential CRUD |
| `server/internal/handler/daemon.go` | Add `/internal/repo-credentials/resolve` endpoint |
| `server/internal/service/credential.go` | Encryption/decryption logic |
| `server/internal/daemon/repocache/cache.go` | Add `CredentialResolver` interface, inject into `Sync`/`CreateWorktree`/`gitCloneBare`/`runGitFetch` |
| `server/internal/daemon/repocache/credential.go` | `withPATCredential`, `withSSHCredential` helpers |
| `server/internal/daemon/daemon.go` | Implement `CredentialResolver` via HTTP client with in-memory TTL cache, pass to `Cache` |
| `server/cmd/multica/cmd_repo.go` | Add `credential` subcommands |
| `server/pkg/redact/redact.go` | Add token patterns |
| `packages/core/api/client.ts` | Add credential API methods |
| `packages/core/types/index.ts` | Add `RepoCredentialStatus` type |
| `packages/views/settings/components/repositories-tab.tsx` | Per-repo credential UI |

## Security Considerations

1. **Encryption at rest** — AES-256-GCM with per-row key derivation. `MULTICA_CREDENTIAL_KEY` is required; server refuses to start without it.
2. **No plaintext in transit (within the system)** — Credentials flow from DB → server → daemon over the internal API. In production, this should be TLS. For local dev, it's localhost-only.
3. **No plaintext in logs** — Redaction patterns in `pkg/redact`. Temp files for askpass/SSH keys use restrictive permissions.
4. **No agent exposure** — Credentials are injected into daemon-controlled git processes only. Agent processes never receive credential env vars or temp files.
5. **Minimal credential lifetime** — Askpass scripts and SSH key temp files exist only for the duration of the git command (created, used, deleted in a single function scope via `defer`).
6. **Access control** — Credential endpoints require workspace membership (owner/admin role). The daemon-only resolve endpoint uses the daemon's registration token.

## Implementation Order

Each step is independently shippable and testable:

1. **URL normalization helper** — Implement and unit-test `normalizeRepoURL()`. Used everywhere else.
2. **Database migration + encryption service** (`repo_credential` table + `service/credential.go`) — `MULTICA_CREDENTIAL_KEY` required; add server startup check. Run `make sqlc` after adding queries.
3. **Server API endpoints** (CRUD + resolve) — Normalize URLs on write/read. Include `git ls-remote` validate-on-save in the PUT handler. Test with `curl`.
4. **Credential injection in repocache** (`CredentialResolver` interface + `withPATCredential`/`withSSHCredential`) — Unit test with a mock resolver. Integration test with a local git server.
5. **Daemon resolver implementation** — Wire the HTTP client with in-memory TTL cache (5 min) to the server's resolve endpoint. End-to-end test: configure a credential, add a private repo, verify the daemon clones it.
6. **Frontend UI** — Extend `RepositoriesTab` with per-repo credential fields. Show validation error from the server if `git ls-remote` fails.
7. **CLI commands** — `multica repo credential set/get/delete`.
8. **Redaction patterns** — Extend `pkg/redact`.
9. **GitHub App support** (future) — Add `github_app` cred type that generates ephemeral installation tokens. Deferred because PATs cover the MVP use case.

## Open Questions

1. **Credential rotation notifications** — Should we detect expired/revoked tokens (e.g., clone fails with 401) and surface this in the UI? Useful but adds complexity to the error reporting path. Recommend: log a warning in the daemon, defer UI notification to a follow-up.
2. **Credential sharing across workspaces** — Some orgs may want a single GitHub App installation credential shared across workspaces. Deferred: per-workspace credentials are the right MVP, org-level credentials can be layered on top.
3. **`git push` support** — Agents currently can't push to remotes. Should credential injection also cover push operations via a daemon endpoint? Recommend: yes, but as a separate follow-up issue since it requires a new daemon endpoint and agent CLI command.
