# Fork maintenance

This is tokimonki's fork of [durable-streams/durable-streams](https://github.com/durable-streams/durable-streams).
We maintain a custom build of the Caddy-based stream server with patches that upstream
has not yet merged or that are specific to our deployment.

## Branch structure

```
main              ← exact mirror of upstream, never commit directly
tokimonki         ← main + our patches, this is what we build from
fix/*             ← branches for upstream PRs (leave as-is until merged)
```

**`main`** tracks upstream exactly. Every commit on `main` exists in
`durable-streams/durable-streams:main`. Never push our own commits here.

**`tokimonki`** is our production branch. It is always `main` + a small set of
cherry-picked patches on top. Binaries are built from this branch. When upstream
absorbs one of our patches, we drop it from `tokimonki`.

**`fix/*`** branches are for open PRs to upstream. They stay as-is (with their
own merge history) because they are referenced by upstream PRs. Do not rebase
or force-push these.

## Current patches on `tokimonki`

| Commit | Description | Upstream PR | Drop when |
|--------|-------------|-------------|-----------|
| `f3c63ca` | Fix SSE streaming with forward_auth (`http.NewResponseController`) | [#253](https://github.com/durable-streams/durable-streams/pull/253) | Upstream merges #253 |
| `9b65b2c` | Add `ggicci/caddy-jwt` module for local JWT verification | N/A (tokimonki-specific) | Never — permanent divergence |
| `073b6e0` | Add `tokimonki/durable-streams-authorisation` module for permission enforcement | N/A (tokimonki-specific) | Never — permanent divergence |

The JWT and authorisation modules are permanent additions. caddy-jwt verifies stream
access tokens locally using a public key, eliminating the forward_auth callback to
Rails that caused Puma thread deadlocks. durable-streams-authorisation enforces
read/write permissions and server-only methods on top of the authenticated identity.

## Syncing with upstream

### 1. Sync `main`

From inside this repo:

```bash
gh repo sync          # syncs local main from upstream parent
```

Or if the remote fork is behind:

```bash
gh repo sync tokimonki/durable-streams   # syncs remote fork's main from parent
git pull origin main                      # pull to local
```

### 2. Rebase `tokimonki` onto the new `main`

```bash
git checkout tokimonki
git rebase main
```

If there are conflicts, they will be in our patched files. Resolve them by
examining what upstream changed and adapting our patch.

If rebase is painful (many conflicts), an alternative:

```bash
git checkout main
git branch -D tokimonki
git checkout -b tokimonki
git cherry-pick <patch1> <patch2> ...
```

This recreates `tokimonki` from scratch. Use the commit hashes from the
"Current patches" table above, updating them after each cherry-pick.

### 3. Verify

```bash
# Patches ahead of main
git log main..tokimonki --oneline

# Nothing behind main
git log tokimonki..main --oneline    # should be empty

# Build compiles
cd packages/caddy-plugin && go build ./cmd/caddy/
```

### 4. Push

```bash
git push origin tokimonki --force-with-lease
```

Force-push is expected for `tokimonki` since we rebase. Use `--force-with-lease`
to avoid overwriting someone else's push.

### 5. Rebuild binaries

From the exchange repo root:

```bash
bin/build-durable-streams
```

The script checks the submodule exists and is on the `tokimonki` branch, builds
a test binary to verify both `durable_streams` and `jwt` modules are compiled in,
then builds darwin/arm64 and linux/amd64 binaries into `config/caddy/`. Commit
the binaries via GitButler.

## Dropping a patch after upstream merges it

When upstream merges one of our PRs (e.g., #253):

1. Sync `main` (the fix is now in `main`)
2. Recreate `tokimonki` from the new `main`, cherry-picking only the remaining patches
3. Update the "Current patches" table in this file
4. Rebuild binaries

## Adding a new patch

1. Make the change on `tokimonki` (or cherry-pick from a `fix/*` branch)
2. Add it to the "Current patches" table with upstream PR link (if any)
3. Rebuild and test
4. Push `tokimonki` and commit new binaries to the exchange repo

## Why not xcaddy?

[xcaddy](https://github.com/caddyserver/xcaddy) is Caddy's official build tool
for composing plugins. We can't use it yet because:

1. Our SSE fix (PR #253) is not merged upstream, so the published plugin doesn't
   include it. `xcaddy` pulls from published module versions.
2. We need the fix + `caddy-jwt` in the same binary.

Once upstream merges #253 and publishes a new release, we could switch to:

```bash
xcaddy build \
  --with github.com/durable-streams/durable-streams/packages/caddy-plugin \
  --with github.com/ggicci/caddy-jwt
```

This would eliminate the need for a source fork entirely. Evaluate when #253 lands.

## Sync schedule

Check upstream monthly, or when we see activity on our open PRs. The command
to check:

```bash
gh api "repos/durable-streams/durable-streams/commits?per_page=5" \
  --jq '.[] | "\(.sha[0:7]) \(.commit.message | split("\n")[0])"'
```

Compare against `git log main --oneline -1` to see if we're behind.

## Related files in tokimonki-exchange

| Path | Role |
|------|------|
| `config/caddy/durable-streams/` | This submodule (the fork) |
| `config/caddy/durable-streams-server-darwin-arm64` | Built binary (macOS dev) |
| `config/caddy/durable-streams-server-linux-amd64` | Built binary (production) |
| `config/caddy/Caddyfile` | Generated by `durable_streams-rails` `ServerConfig` |
| `config/durable_streams.yml` | Stream server config (read by `ServerConfig`) |
| `gems/durable_streams-rails/lib/durable_streams/rails/server_config.rb` | Generates the Caddyfile |
| `docs/references/durables/durable-streams/` | Upstream reference (read-only, for code reading) |
