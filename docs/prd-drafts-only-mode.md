# PRD: Drafts-Only Mode (`--no-send`)

## Problem

When gog is used by AI agents or automation, there's no way to grant write access while preventing outbound messages. An agent authorized with `gmail.modify` can search, read, organize mail, **and** send emails — all with the same token. Google doesn't offer a "drafts only" OAuth scope; `gmail.compose` still includes send.

Operators need a CLI-level guardrail: full read/write for internal operations, but hard-blocked on anything that sends a message to another person.

## Goal

Add a `--no-send` mode that blocks outbound message operations at the CLI level while preserving all other read/write functionality.

## Scope

### Services with outbound operations (messages that reach other people)

| Service | Outbound commands | Blocked in `--no-send` |
|---------|------------------|----------------------|
| **Gmail** | `gmail send`, `gmail drafts send`, top-level `send` | ✅ Blocked |
| **Chat** | `chat messages send`, `chat dm send` | ✅ Blocked |

### Services with NO outbound operations (self-contained writes)

These modify your own data only — no message leaves your account:

| Service | Write operations | Blocked? |
|---------|-----------------|----------|
| **Calendar** | `create`, `update`, `delete`, `respond`, `focus-time`, `ooo`, `working-location` | ❌ Allowed |
| **Drive** | `upload`, `mkdir`, `delete`, `move`, `unshare`, `comments create/reply` | ❌ Allowed |
| **Docs** | `create`, `write`, `insert`, `delete`, `update`, `edit`, `comments add/reply` | ❌ Allowed |
| **Slides** | `create`, `add-slide`, `delete-slide`, `update-notes` | ❌ Allowed |
| **Sheets** | `create`, `update`, `append`, `insert` | ❌ Allowed |
| **Contacts** | `create`, `update`, `delete` | ❌ Allowed |
| **Tasks** | `add`, `update`, `delete`, `lists create` | ❌ Allowed |
| **Forms** | `create` | ❌ Allowed |
| **Keep** | (read-only in gog) | N/A |
| **Groups** | (read-only in gog) | N/A |

### Edge cases to discuss

| Command | Outbound? | Decision |
|---------|-----------|----------|
| `calendar respond` (RSVP) | Sends notification to organizer | **Allow** — it's a response to something sent to you, not an unsolicited outbound message |
| `drive comments create/reply` | Visible to collaborators | **Allow** — comments are within a shared workspace context, not direct messaging |
| `docs comments add/reply` | Same as drive comments | **Allow** |
| `gmail autoforward update` | Could silently redirect mail | **TBD** — could argue this is a "send" in disguise |
| `gmail delegates add` | Grants mailbox access to someone | **TBD** — not a "send" but is a security-sensitive outbound action |
| `gmail filters create` (with forward) | Could auto-forward mail | **TBD** — same concern as autoforward |

## Design

### 1. Flag: `--no-send`

New root-level flag on `RootFlags`:

```go
type RootFlags struct {
    // ...existing flags...
    NoSend bool `name:"no-send" help:"Block outbound message commands (send email, send chat); drafts and all other writes allowed" default:"${no_send}"`
}
```

Environment variable: `GOG_NO_SEND=true`

### 2. Enforcement point

Single check function called early in `Run()`, similar to `enforceEnabledCommands`:

```go
func enforceNoSend(kctx *kong.Context, noSend bool) error {
    if !noSend {
        return nil
    }
    // Check if current command path matches a blocked command
    // Return clear error: "blocked by --no-send: use 'gmail drafts create' instead"
}
```

Blocked command paths:
- `gmail send`
- `gmail drafts send`
- `send` (top-level alias)
- `chat messages send`
- `chat dm send`

### 3. Error message

When blocked:

```
Error: command "gmail send" is blocked by --no-send mode

To create a draft instead:
  gog gmail drafts create --to user@example.com --subject "..." --body "..."

To disable this restriction, remove --no-send or unset GOG_NO_SEND.
```

### 4. Dry-run integration

When `--no-send` is active and `--dry-run` is also set, the dry-run output should note that the command would be blocked even without `--dry-run`.

### 5. Status output

`gog auth status` / `gog status` should show when `--no-send` is active so operators can verify the guardrail is on.

## Implementation Plan

### Task 1: Add `--no-send` flag to RootFlags
- Add field to `RootFlags` struct in `root.go`
- Add `GOG_NO_SEND` env var in kong vars
- Wire up in `Run()`

### Task 2: Create `enforce_no_send.go`
- Implement `enforceNoSend()` with blocked command list
- Call from `Run()` after `enforceEnabledCommands`
- Clear error messages with suggested alternatives

### Task 3: Status display
- Show `no-send: active` in `gog status` / `gog auth status` output
- Show in dry-run output

### Task 4: Tests
- Unit tests for `enforceNoSend()` with each blocked command
- Unit tests confirming allowed commands pass through
- Integration with `--dry-run`

### Task 5: Documentation
- Update README/help text
- Add to agent mode docs

## Open Questions

1. **Should `calendar respond` be blocked?** It notifies the organizer. Current proposal: allow.
2. **Should `gmail autoforward`, `gmail delegates`, `gmail filters` (with forward action) be blocked?** These are indirect "send" operations. Could be a separate `--no-send-strict` mode or included in v1.
3. **Should this be per-account or global?** Current proposal: global flag/env var. Per-account would require storing in config or keyring metadata.
4. **Name bikeshed:** `--no-send` vs `--drafts-only` vs `--safe-write` vs `--block-send`? `--no-send` is most explicit about what it does.
