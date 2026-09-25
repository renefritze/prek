# Configuration includes

This document specifies `includes`, a way for a project configuration to pull
hook definitions from other configuration files. An include can be a local
file, a remote file fetched over HTTPS, or a file inside a Git repository at a
given revision. Remote includes are cached, and they can be pinned to a SHA-256
digest of their content. Git includes reuse the store's repository clones and are
pinned by their `rev`.

Tracking issue: [j178/prek#1238](https://github.com/j178/prek/issues/1238).
The design follows the direction discussed in
[this comment](https://github.com/j178/prek/issues/1238#issuecomment-3806430461):
an include names a location and may carry a content digest. In this spec the
digest is optional.

## Motivation

Organizations with many repositories tend to copy the same hook list into every
repository. Keeping those copies in sync is manual work: each `rev` bump, new
hook, or changed argument has to be repeated everywhere, and the copies drift.

Monorepos and multi-package repositories have a smaller version of the same
problem. Several projects in one workspace often share a baseline such as
whitespace fixers, a license check, and a secret scanner, and today each project
config has to repeat it.

Tools in adjacent spaces solve this with includes. Task supports remote
Taskfiles, GitLab CI has `include:`, and Renovate has shareable presets. `prek`
should let a configuration say "these hooks come from that file" and then treat
the result as one project configuration.

## Goals

- Include hook definitions from one or more files, listed in order.
- Allow local paths, remote HTTPS URLs, and files in Git repositories in the
  same list.
- Cache remote includes so ordinary runs work offline and do not pay a network
  round trip on every commit.
- Let users pin a remote include to the exact bytes they reviewed, without
  requiring it.
- Fail loudly on anything that would change which hooks run in a way the user
  cannot see: an unreachable include with no cached copy, a digest mismatch,
  two sources defining the same hook, or unpinned remote content that changed
  since the user last accepted it.

## Non-goals

- Overriding or patching included hooks. The issue mentions "additions or
  overrides". This spec only supports additions. Two sources defining the same
  hook is an error, not an override. See [Future work](#future-work).
- Nested includes. An included file cannot include other files.
- Authentication for private HTTPS includes. Private sources are supported
  through [Git includes](#git-includes), which use the user's Git credentials.
- Includes in hook manifests (`.pre-commit-hooks.yaml`).
- Rewriting included files with `prek update`. `prek update` does bump the
  `rev` of Git includes, because that value lives in the main config. See
  [`prek update` and Git includes](#prek-update-and-git-includes).
- Top-level settings in included files, such as `default_stages` or `exclude`.
  Only the main configuration controls project-wide behavior.

## Configuration

### `includes`

A new optional top-level key, `includes`, is added to the project configuration.

=== ".pre-commit-config.yaml"

    ```yaml
    includes:
      - ci/hooks/base.yaml
      - https://example.com/org/prek/python.yaml
      - url: https://example.com/org/prek/security.toml
        sha256: 3a6eb0790f39ac87c94f3856b2dd2c5d110e6811602261a9a923d3bb23adc8b7
      - path: ../shared/rust.yaml
      - repo: https://github.com/org/prek-shared
        rev: v1.4.0
        path: go.yaml

    repos:
      - repo: local
        hooks:
          - id: project-check
            name: Project check
            language: system
            entry: ./scripts/check.sh
    ```

=== "prek.toml"

    ```toml
    includes = [
      "ci/hooks/base.yaml",
      "https://example.com/org/prek/python.yaml",
      { url = "https://example.com/org/prek/security.toml", sha256 = "3a6eb0790f39ac87c94f3856b2dd2c5d110e6811602261a9a923d3bb23adc8b7" },
      { path = "../shared/rust.yaml" },
      { repo = "https://github.com/org/prek-shared", rev = "v1.4.0", path = "go.yaml" },
    ]

    [[repos]]
    repo = "local"
    hooks = [
      { id = "project-check", name = "Project check", language = "system", entry = "./scripts/check.sh" },
    ]
    ```

- **Type**: list of include entries
- **Default**: empty list
- **Scope**: the main project configuration only
- **Order**: significant, see [Hook order](#hook-order)

`includes` is a `prek`-only key. `repos` stays required in the main
configuration. A configuration that only includes other files writes
`repos: []`. Keeping `repos` required means existing configurations keep their
current error messages.

### Include entries

Each entry is either a string or a table.

A string is classified by one rule. If it matches
`^[A-Za-z][A-Za-z0-9+.-]*://`, it is a URL: `https` makes a remote include,
`http` does too when the rules in [Remote URL rules](#remote-url-rules) allow
it, and any other scheme, such as `file://`, `git+https://`, or `s3://`, is a
parse error. Every other string is a local path. Windows drive paths such as
`C:\x.yaml` and `C:/x.yaml` do not match the URL pattern, so they stay paths.

A table has one of three shapes:

| Shape | Keys | Meaning |
| -- | -- | -- |
| Local | `path` | Local file path. |
| Remote | `url`, optional `sha256` | HTTPS URL, optionally pinned to the SHA-256 of the file's bytes. |
| Git | `repo`, `rev`, `path` | File at `path` inside the Git repository `repo` at revision `rev`. |

The shape is decided by the keys present: `url` means remote, `repo` means Git,
and `path` alone means local. Git includes have no string shorthand, because a
Git include always needs three values.

Table rules, all enforced during parsing so errors carry a line and column:

- `url` together with `repo` or `path` is an error.
- `repo` without `rev` or without `path` is an error. `rev` without `repo` is an
  error.
- `sha256` is only valid with `url`. With `path` it is an error because local
  files are already under the user's control and a digest would only go stale.
  With `repo` it is an error because Git includes are pinned through `rev`,
  not through a digest. Only a commit SHA `rev` is an immutable pin: tags can be
  moved by the repository maintainer, and branches do not pin anything. Use a
  commit SHA, for example via `prek update --freeze`, when you need integrity
  for a Git include. See [Freshness and pinning](#freshness-and-pinning).
- Unknown keys in an include table are an error, not a warning. A typo like
  `sha265` would otherwise disable pinning without any visible sign.
- `sha256` accepts 64 hex characters, case-insensitive, with an optional
  `sha256:` prefix. This is the format `Sha256Digest::from_str` already accepts.
- An empty string, or a table value that is an empty string, is an error.

### Remote URL rules

- The scheme must be `https`. Plain `http` is accepted only when the host is a
  loopback address (`localhost`, `127.0.0.0/8`, or `::1`). This supports local
  development servers and the test suite, and it never exposes real traffic to
  a downgrade.
- URLs with user info (`https://user:token@host/...`) are rejected. Credentials
  in a committed config file leak into logs, warnings, and the cache metadata.
  Private includes are future work.
- The URL is parsed and normalized with `reqwest::Url`. The normalized form is
  used as the cache identity and in messages.
- Redirects are checked hop by hop, before the next request is sent. A redirect
  policy is a property of the client, not of a request, so remote includes use
  a dedicated client. It is built with the same settings as
  `http::REQWEST_CLIENT` (TLS backend, custom certificates, proxy), plus a
  `reqwest::redirect::Policy::custom` that refuses a hop to plain `http` on a
  non-loopback host and a hop to a URL with user info, and keeps the default
  limit of 10 hops. Checking only `response.url()` would be too late, because
  the plaintext request would already have been sent. The final URL is checked
  again after the response as a second guard.
- The file format is chosen from the extension of the URL path, ignoring the
  query string and fragment. A `.toml` path is parsed as TOML. Everything else
  is parsed as YAML. This matches how `load_config` treats local files.

### Local path rules

- Relative paths resolve against the directory of the configuration file that
  lists them. They do not resolve against the process working directory, the
  workspace root, or the Git root. This matches how relative `repo:` paths are
  already resolved.
- Absolute paths are allowed. A leading `~` is expanded with the existing
  `fs::expand_tilde`.
- The path must name a regular file. A missing path or a directory is an error.
- The file format is chosen by extension, the same way as for the main config.
- Local includes may live outside the Git repository. Those files are not
  subject to the staged-config check described in
  [Command behavior](#command-behavior).

### Git include rules

- `repo` accepts the same values as a remote hook repository's `repo`: HTTPS,
  SSH, SCP-style, and `file://` URLs, plus local repository paths. A relative
  local repository path resolves against the directory of the configuration file
  that lists it, using `resolve_relative_repo_sources`. `local`, `meta`, and
  `builtin` are errors here.
- `rev` accepts the same values as a hook repository's `rev`: a tag, a commit
  SHA, or a branch. A branch produces the existing mutable-`rev` warning.
- `path` is relative to the repository root. It must not be absolute, and after
  normalization it must not start with `..`. This lexical check happens at parse
  time.
- No component of `path` may be a symlink in the repository tree, whether it is
  the file itself or a parent directory. A lexical check alone is not enough:
  ordinary reads follow symlinks, so a committed symlink could point outside the
  clone. Resolution and `prek update` check this the same way, against the tree
  at `rev` rather than the filesystem: `git ls-tree <rev> -- <prefix>` for each
  prefix of `path` must report a tree (`040000`) for every parent and a regular
  blob (`100644` or `100755`) for the file. A symlink (`120000`) or a submodule
  (`160000`) anywhere is an error. The file is then read with
  `git show <rev>:<path>`, so runtime resolution and update validation read
  exactly the same bytes and never follow a link.
- The file format is chosen by the extension of `path`, the same way as for
  local files.

### Duplicate and self includes

These are errors:

- The same local file listed twice, compared after canonicalization, so
  `./a.yaml` and `a.yaml` count as the same file.
- The same remote URL listed twice, compared after normalization.
- The same Git `(repo, rev, path)` listed twice. `path` is compared after
  normalization. The same `repo` and `rev` with different `path`s is fine and
  shares one clone.
- A local include that resolves to the main configuration file itself.

Listing the same file twice would make every hook in it collide with itself. The
error names the duplicate directly instead of reporting confusing hook
collisions.

## Included file format

An included file uses the same syntax as a project configuration, in YAML or
TOML, but only a subset of top-level keys is meaningful.

Allowed keys:

| Key | Behavior |
| -- | -- |
| `repos` | Required. Same schema as the main config. `repos: []` is valid. |
| `priorities` | Optional. Priority aliases scoped to this included file. |
| `minimum_prek_version` | Optional. Enforced exactly as in the main config. |
| `ci`, `minimum_pre_commit_version` | Ignored silently, as in the main config. |
| `x-*` | Ignored silently, as in the main config. |

Keys that are known project settings are errors when they appear in an included
file:

`includes`, `default_install_hook_types`, `default_language_version`,
`default_stages`, `default_env`, `files`, `exclude`, `fail_fast`, `orphan`,
`update`.

The reason is that these keys change which files hooks see, which stages they
run in, or how the whole project behaves. Silently dropping `exclude: ^vendor/`
from an included file would run its hooks on vendored code. Applying it would
make one include change the behavior of hooks from other sources. Both
outcomes are surprising, so the author has to publish a file shaped for
inclusion. Allowing these keys scoped to the included hooks is listed in
[Future work](#future-work).

The `includes` key gets its own error message
(`nested includes are not supported`) because it is the most likely mistake.

Any other unknown key produces the usual unused-key warning. The warning names
the include source, for example:

```text
warning: Ignored unexpected keys in `https://example.com/org/prek/python.yaml`: `repo_defaults`
```

The existing mutable-`rev` warning also applies to remote repositories declared
in included files, and to the `rev` of Git includes. It names the include source
next to each repository.

### Repository paths inside included files

Existing configs may use a local directory as `repo:` (for example
`repo: ../hooks-repo` with a `rev`). Inside an included file:

- **Local include**: relative repository paths resolve against the included
  file's directory, using the same `resolve_relative_repo_sources` logic as the
  main config.
- **Remote and Git includes**: a `repo:` value that is a filesystem path,
  relative or absolute, is an error. A shared file cannot know the layout of the
  machine it runs on. Resolving against a Git checkout is also rejected, because
  the checkout lives in the store and a hook repository nested inside it is
  almost certainly a mistake. URL-like repositories (anything containing `://`,
  or SCP-style `user@host:path`) and `local`, `meta`, and `builtin` are
  allowed.

### Paths inside hook definitions

Includes do not change where hooks run. Every hook, whatever file defined it,
runs in its project directory, and `entry`, `files`, `exclude`, `args`, and
`language: script` paths are interpreted against the project, exactly as if the
hook were written in the main config. An included local hook with
`entry: ./scripts/lint.sh` runs the project's `scripts/lint.sh`, not a script
next to the included file. The reference docs must say this explicitly, because
people will expect paths relative to the included file.

## Merge semantics

Resolving includes produces an ordered list of **configuration sources** for a
project: every include in listed order, then the main configuration. Hook
construction iterates these sources in that order.

### Hook order

Hooks from includes come first, in the order the includes are listed, followed
by hooks from the main config's `repos`. This matches reading the file top to
bottom, since `includes` normally sits above `repos`.

Order matters because a hook with no `priority` gets an implicit priority from
its position, so hooks without explicit priorities run sequentially in this
merged order.

### Priorities

- A priority alias is resolved against the `priorities` table of the file that
  defines the hook. An included file cannot use an alias declared only in the
  main config, and the main config cannot use an alias declared only in an
  include. Unknown aliases are errors, as today.
- The same alias name may be declared with different values in different files.
  They do not conflict because each is scoped to its own file.
- After aliases resolve to integers, scheduling compares priorities across all
  hooks in the project, whichever source they came from. The existing rule that
  priorities are compared across one project's hooks still holds. That set now
  includes included hooks.

### Top-level settings

The main config's top-level settings apply to every hook in the project,
including included hooks: `files`, `exclude`, `default_stages`,
`default_language_version`, `default_env`, `fail_fast`, and `orphan` behave as
if the included hooks had been written in the main file.

### Hook collisions

Each hook has a set of **selector names**: its `id`, plus its `alias` if it has
one. These are the names `prek run <hook>`, `--skip`, and `SKIP` match against.

It is an error when a selector name is defined in more than one configuration
source of the same project. This covers:

- an include and the main config,
- two includes,
- an `id` in one source matching an `alias` in another, and
- `alias` against `alias`.

It also covers `meta` and `builtin` hooks. Two sources that both add
`check-hooks-apply` collide.

Configuration data alone is not enough. A remote hook repository's manifest
can set `alias` on a hook, and `HookSpec::from_remote` keeps that alias unless
the configured hook overrides it. So detection runs in two passes, both before
any hook is installed or run:

1. **Configured names.** After all of a project's includes are resolved, the
   configured `id`s and `alias`es are checked. This catches most collisions
   early, including in `prek validate-config`, and needs no hook repository.
   Only Git include repositories are cloned before this pass, because their
   files are the configuration data.
2. **Effective names.** After hook repositories are cloned and their manifests
   read, `init_hooks` checks the effective selector names of the built hooks:
   each hook's `id` plus its resolved `alias`, wherever that alias came from.
   The error format is the same, and a manifest-provided name is marked
   `(alias from manifest)`.

`prek validate-config` runs only the first pass, because it does not clone hook
repositories. The reference docs say so.

Duplicate selector names **within one source** stay allowed. `pre-commit`
allows listing the same hook twice with different arguments, and existing
configs rely on that. Only collisions across sources are errors.

Projects are independent. Two projects in a workspace may define the same hook
id, including through the same include, as they can today.

Error format:

```text
error: Hook `ruff` is defined in more than one configuration source of project `.`:
  - `https://example.com/org/prek/python.yaml`: repos[0].hooks[1] (id)
  - `.pre-commit-config.yaml`: repos[2].hooks[0] (id)
hint: Hook ids and aliases must be unique across a configuration and its includes. Remove one of the definitions, or give it a different `id` or `alias`.
```

When several names collide, all of them are reported in one error, sorted by
the order of their first definition. The ordering is deterministic so snapshots
are stable.

## Remote includes

### Fetching

- Remote includes are fetched with the dedicated client described in
  [Remote URL rules](#remote-url-rules). It is built from the same settings as
  `http::REQWEST_CLIENT`, so proxy settings, `PREK_NATIVE_TLS`,
  `SSL_CERT_FILE`, and `SSL_CERT_DIR` behave as they do for other downloads,
  along with the shared 30 second connect and read (inactivity) timeouts.
- Each request also sets a 30 second total timeout with
  `RequestBuilder::timeout`. The shared read timeout only fires when no bytes
  arrive, so a server that keeps trickling data could otherwise hold a commit
  indefinitely. The total timeout is a parameter of the fetch function so tests
  can use a short value.
- Responses larger than 1 MiB are rejected. The limit is checked against
  `Content-Length` when present and enforced while streaming.
- Any status other than `200` is a failure, except `304` on a conditional
  request that has a cached entry.
- The SHA-256 of the response body is computed while streaming, with
  `checksum::HashReader`.
- Within one `prek` process, each distinct normalized URL is fetched at most
  once, even when several projects in a workspace include it. Distinct URLs are
  fetched concurrently, bounded by the existing internal concurrency limit.

### Integrity pinning

`sha256` is optional.

- **Unpinned** includes follow the freshness rules below and pick up upstream
  changes when they revalidate.
- **Pinned** includes are content-addressed. The digest covers the raw response
  bytes, with no newline or encoding normalization, so
  `curl -fsSL <url> | sha256sum` gives the value to write. A pinned include is
  used only if its bytes hash to the pinned value. Otherwise it is an error.
  Updating a pinned include means changing the digest in the config, which
  shows up in review.

Unpinned includes are not trusted blindly. They go through
[Change confirmation](#change-confirmation), so changed content never runs
without an explicit approval.

### Cache

Remote includes are cached in the store, under `CacheBucket::Prek`:

```text
$PREK_HOME/cache/prek/includes/
├── entries/<url-key>.json   # one per normalized URL
└── blobs/<sha256>           # raw content, named by its digest
```

`<url-key>` is the hex SHA-256 of the normalized URL string. An entry file
looks like this:

```json
{
  "version": 1,
  "url": "https://example.com/org/prek/python.yaml",
  "sha256": "3a6eb0790f39ac87c94f3856b2dd2c5d110e6811602261a9a923d3bb23adc8b7",
  "fetched_at": 1790000000,
  "etag": "\"5f2b-1a\"",
  "last_modified": "Tue, 22 Sep 2026 08:00:00 GMT"
}
```

Writes must be crash-safe and safe when several processes run at once:

1. Stream the body into a temp file in the store scratch directory.
2. Parse and validate the content as an included file. **Content that fails to
   parse or validate is never written to the cache.**
3. If `blobs/<sha256>` already exists and re-hashes to its name, keep it and
   drop the temp file. Otherwise, whether the blob is missing or corrupt, move
   the verified temp file onto `blobs/<sha256>` with `fs::rename_with_retry`,
   which replaces an existing file atomically. On Windows, `std::fs::rename`
   uses `MoveFileExW` with `MOVEFILE_REPLACE_EXISTING`, and `rename_with_retry`
   already retries the sharing violations antivirus scanners cause.
4. Write the entry JSON to a temp file and rename it over
   `entries/<url-key>.json`.

A reader always loads the entry first and then the blob it names, so it never
sees an entry paired with the wrong content. Every read re-hashes the blob,
which is cheap at 1 MiB or less. A blob whose hash does not match its name is
treated as missing, as if the cache had been tampered with or truncated. An
entry that is missing, cannot be parsed, or has an unknown `version` is treated
as a cache miss and logged at debug level. It is not an error.

A pinned include looks up `blobs/<pin>` directly and does not need an entry. So
two URLs serving identical bytes share one blob, and a pinned include keeps
working after the upstream URL changes content, as long as the pinned blob is
cached.

### Freshness

An unpinned cache entry is **fresh** when `now - fetched_at` is less than the
TTL, and **stale** otherwise.

- The default TTL is 3600 seconds, the same as the workspace discovery cache.
- `PREK_INCLUDE_CACHE_TTL=<seconds>` overrides it. `0` means always
  revalidate. An invalid value produces a warning and falls back to the default,
  the same way `DownloadChecksumPolicy::from_env` handles bad values.
- The global `--refresh` flag makes every unpinned include stale for that
  invocation. It does not affect pinned includes, because their content cannot
  change.
- Revalidating a stale entry sends `If-None-Match` and `If-Modified-Since` when
  the entry has an `etag` or `last_modified`. A `304` response rewrites the
  entry with a new `fetched_at` and keeps the blob.

A fresh entry never touches the network, so a normal commit makes no HTTP
requests while the TTL holds.

### Resolution matrix

| Pin | Cache state | Network result | Outcome |
| -- | -- | -- | -- |
| none | fresh, no `--refresh` | not contacted | Use cached content. |
| none | stale or `--refresh` | `200`, valid | Update the cache and use the new content. |
| none | stale or `--refresh` | `304` | Update `fetched_at` and use cached content. |
| none | stale or `--refresh` | failure | **Warn** and use cached content. |
| none | stale or `--refresh` | `200`, invalid content | **Error.** The cache is left unchanged. |
| none | missing | `200`, valid | Cache and use it. |
| none | missing | failure | **Error.** |
| pinned | blob present and verified | not contacted | Use cached content. |
| pinned | blob missing or corrupt | `200`, digest matches, valid | Cache and use it. |
| pinned | blob missing or corrupt | `200`, digest mismatch | **Error.** Nothing is cached. |
| pinned | blob missing or corrupt | failure | **Error.** |

"Failure" means a DNS, connect, TLS, or timeout error, a non-`200` status, a
rejected redirect, or a body over the size limit.

Invalid content is an error even when an older cached copy exists. Falling back
would hide a broken upstream file until the cache was cleared.

The stale-fallback warning is printed once per URL per invocation:

```text
warning: Failed to refresh included configuration `https://example.com/org/prek/python.yaml`; using the copy cached 3h ago
  caused by: error sending request for url (https://example.com/org/prek/python.yaml)
```

The hard error when there is no usable cache:

```text
error: Failed to fetch included configuration `https://example.com/org/prek/python.yaml`
  caused by: HTTP status server error (503 Service Unavailable) for url (https://example.com/org/prek/python.yaml)
hint: No cached copy is available, so `prek` cannot tell which hooks this configuration defines. Check the URL and your network connection, then try again.
```

The digest mismatch reuses the existing `Sha256Digest::verify` wording:

```text
error: SHA256 checksum mismatch for `https://example.com/org/prek/security.toml`: expected 3a6e…c8b7, got 9f86…0f08
hint: The remote file changed. Review the new content and update the `sha256` in `.pre-commit-config.yaml`.
```

## Git includes

A Git include reads one file from a Git repository at a revision. It reuses the
store's existing repository clones instead of adding a new cache.

### Cloning

- Each Git include becomes a `config::RemoteRepo` with no hooks, built with
  `RemoteRepo::new(repo, rev, Vec::new())`. Its store key, store path, and
  relative-source handling are the same as for a hook repository.
- Git includes from all selected projects are deduplicated by `RemoteRepoKey`
  first, the same way `remote_configs_to_clone` deduplicates hook repositories.
  `Store::clone_repos` does not deduplicate its input, so passing duplicates
  would clone the same `(repo, rev)` several times at once. The deduplicated
  set is cloned in one `Store::clone_repos` batch. That gives parallel clones,
  the auth-failure retry with terminal prompts outside CI, the
  `.prek-repo.json` marker, and crash-safe persistence through
  `fs::rename_with_retry`, with no new code.
- A Git include and a hook repository with the same `(repo, rev)` share one
  clone. So do several Git includes that name different `path`s in the same
  `(repo, rev)`.
- After cloning, `<checkout>/<path>` is read and handed to the same parser and
  validator used for local includes. Then it goes through the checks in
  [Included file format](#included-file-format).
- `git::clone_repo` uses the user's Git configuration, so credential helpers,
  SSH keys, and `url.<base>.insteadOf` rewrites work. This is the supported way
  to include files from private repositories.

### Freshness and pinning

Store clones are immutable per `(repo, rev)`. Once the marker exists, the clone
is never fetched again. Git includes inherit that behavior:

- Only a commit SHA `rev` is pinned. A tag `rev` stays fixed only while the
  existing clone lives: if the maintainer moves the tag, a new clone on another
  machine, in CI, or after `prek cache clean` gets the new content. Use a
  commit SHA when you need the same content everywhere.
- A branch `rev` does not follow the branch after the first clone. This matches
  hook repositories and triggers the same mutable-`rev` warning.
- `--refresh` does not re-clone, again matching hook repositories.
  `prek cache clean` drops every clone.
- There is no TTL, no conditional request, and no stale fallback.
- A Git include moves forward by changing its `rev`, by hand or with
  `prek update`. The change shows up in review like any other `rev` bump.

### Resolution matrix for Git includes

| Clone state | Clone result | Outcome |
| -- | -- | -- |
| present (marker exists) | not attempted | Read `path` from the clone. |
| missing | success | Persist the clone, then read `path`. |
| missing | failure | **Error.** The existing `Failed to clone repo` error, wrapped with the include source. |
| present or cloned | `path` missing, or not a regular file | **Error.** |
| present or cloned | `path` fails to parse or validate | **Error.** |

Error format for a missing path:

```text
error: Included configuration `go.yaml` was not found in `https://github.com/org/prek-shared` at `v1.4.0`
hint: Check the `path`, or choose a `rev` that contains it.
```

### `prek update` and Git includes

`prek update` bumps the `rev` of every Git include in the main config, with the
same rules it applies to hook repositories. The `rev` lives in the main config,
so this does not rewrite included files.

**Selection.** A Git include goes through the same `select_update_revision`
path as a hook repository:

- It picks the newest tag that passes the tag filters: `--include-tag`,
  `--exclude-tag`, `--repo-include-tag`, `--repo-exclude-tag`, and the
  project's `update.repos.<repo>` settings, resolved with
  `UpdateSettings::resolve` and keyed by the include's `repo` value.
- Cooldowns apply, and a cooldown never downgrades.
- `--bleeding-edge` moves to the tip of the default branch.
- `--freeze` writes the commit SHA and a `# frozen: <tag>` comment on the
  include's `rev` line. Stale `# frozen:` comments are reported with the
  existing frozen-mismatch warnings.
- `--repo` and `--exclude-repo` match Git include repositories. Checks for
  unknown `--repo` values, `--repo-include-tag`/`--repo-exclude-tag` keys, and
  `update.repos` entries count Git include repositories as configured.

**Validation.** A hook repository candidate is accepted only if the
repository's manifest still has every configured hook id
(`checkout_and_validate_manifest`). A Git include candidate is accepted only if
`path` exists at the candidate revision, is a regular file, and parses and
validates as an included file with the same code used during resolution. If
validation fails, that target fails: the error is shown like a manifest
validation failure, the `rev` is left unchanged, and the command exits with a
failure status, as it does for hook repositories today. The updater does not
try an older tag instead.

**Sharing.** A Git include and a hook repository with the same source share one
fetch, because they are grouped into one `RepoSource`. They are separate
targets, because a hook repository has to keep its hook ids and an include has
to keep its file.

**Output.** Git includes use the same update, up-to-date, skipped-downgrade, and
failure lines as hook repositories. The label adds the include path:

```text
[https://github.com/org/prek-shared (include go.yaml)] updating v1.4.0 -> v1.6.0
```

**Scope.** Only the main config is rewritten. Repositories inside included files
keep their `rev`s, and local include files are never modified.

## Change confirmation

Unpinned remote content can change without any change in the repository. To
keep that from swapping in new hooks silently, `prek` uses trust on first use
(TOFU) for every include whose content is not pinned:

- HTTPS includes without `sha256`.
- Git includes whose `rev` is not a full commit SHA. Store clones don't change,
  but a moved tag or a branch gives different content on a new clone, for
  example on another machine or after `prek cache clean`.

Local includes, pinned HTTPS includes, and Git includes at a commit SHA skip
this check. The user controls local files, and a pin is already an explicit
approval.

### Trust store

`$PREK_HOME/include-trust.json` records, per machine, the digest last accepted
for each source:

```json
{
  "version": 1,
  "sources": {
    "https://example.com/org/prek/python.yaml": {
      "sha256": "3a6eb0790f39ac87c94f3856b2dd2c5d110e6811602261a9a923d3bb23adc8b7",
      "accepted_at": 1790000000
    },
    "git+https://github.com/org/prek-shared@v1.4.0:go.yaml": {
      "sha256": "9f86d081884c7d659a2feaa0c55ad015a3bf4f1b2b0b822cd15d6c15b0f00a08",
      "accepted_at": 1790000000
    }
  }
}
```

- The key is the normalized URL for HTTPS includes, and `repo`, `rev`, and
  `path` for Git includes, so one approval covers every project on the machine
  that includes the same source.
- The digest covers the raw bytes of the included file, the same bytes that
  `sha256` pins.
- The file is written atomically (a temp file and then a rename) while the
  store lock is held. `run` already holds that lock during hook
  initialization.
- A missing file means nothing has been accepted yet. A file that can't be
  parsed, or that has an unknown `version`, produces a warning and is treated
  as empty. Anyone who can corrupt it could also edit it, so failing hard would
  add no protection.

### Decisions

The check runs right after include resolution, before the collision checks and
before any hook repository is cloned:

| Trust store | Resolved digest | Outcome |
| -- | -- | -- |
| no entry | any | **First use.** Record the digest and continue, with a note on stderr naming the source and digest. |
| same digest | equal | Continue silently. |
| different digest | differs, interactive | Show the source, the old and new digests, and a unified diff of the included file when the old blob is still cached. Ask `Accept the new content? [y/N]`. Yes records the digest and continues. Anything else is a hard error. |
| different digest | differs, non-interactive | **Hard error.** Nothing runs. |
| different digest | differs, `PREK_ACCEPT_INCLUDE_CHANGES=1` | Record the digest, print the same note as on first use, and continue. |

A run is interactive when it is not under CI (`EnvVars::is_under_ci`) and
`prek` can open the controlling terminal for reading (`/dev/tty` on Unix,
`CONIN$` on Windows) with stderr attached to a terminal. The prompt reads from
the controlling terminal and not from stdin, because Git hooks such as
`pre-push` use stdin for their own input. Tests provide answers through the
internal `PREK_INTERNAL__INCLUDE_PROMPT_INPUT` variable, which names a file to
read answers from, following the existing `PREK_INTERNAL__*` pattern.

Non-interactive error format:

```text
error: Included configuration `https://example.com/org/prek/python.yaml` changed since it was last accepted
  accepted: 3a6e…c8b7
  current:  9f86…0f08
hint: Review the change, then run `prek util trust-includes`, or set `sha256` on the include to pin it. In CI, set `PREK_ACCEPT_INCLUDE_CHANGES=1` to accept changes automatically.
```

### Accepting changes explicitly

`prek util trust-includes` resolves the includes of the selected projects with
network access and shows every source whose digest is new or different, with
the same diff as the prompt. Then it records them. `--yes` skips the per-source
confirmation, and `--dry-run` only lists what would change. It changes no hook
state and runs no hooks.

### Interaction with the cache and GC

- The download cache and the trust store are separate. A fetch that brings new
  content updates the cache as described in [Cache](#cache). Rejecting the new
  content does not roll the cache back, so the next run prompts or fails again
  until the change is accepted or the include is pinned.
- Blobs named by a trust store entry are kept by `prek cache gc`. The last
  accepted content therefore stays available for the diff, and for pinning the
  include to the old digest while offline.
- `prek cache clean` removes the trust store together with the rest of the
  store, so the next run is a first use again.

## Error model

Every include problem is a hard error for the project that owns it. A hard
error means the command exits with the normal error status **before any hook is
cloned, installed, or executed** for the affected invocation. Hard errors are:

- invalid include entries, such as a bad scheme, user info, a malformed digest,
  unknown table keys, `path` and `url` together, a Git include without `rev`,
  or a Git `path` that escapes the checkout,
- a missing or unreadable local include, or one that is a directory,
- duplicate includes and self-includes,
- nested includes and forbidden top-level keys in an included file,
- parse and validation errors in included files, including
  `minimum_prek_version` and unknown priority aliases,
- filesystem `repo:` paths in a remote or Git include,
- a remote fetch failure with no usable cached copy,
- a Git include clone failure, or a Git include `path` missing at that `rev`,
- a digest mismatch,
- unpinned content that changed since it was accepted, when the change is
  declined or the run is non-interactive without
  `PREK_ACCEPT_INCLUDE_CHANGES=1`, and
- hook selector collisions across sources.

The only non-fatal include condition is the stale-cache fallback for unpinned
remote includes.

Include errors get their own error type, `config::IncludeError`. They must never
be an `Io(NotFound)` inside `config::Error`, because `Error::warn_parse_error`
deliberately skips `NotFound` to stay quiet about a missing main config. A
missing include must not be swallowed the same way.

## Command behavior

Include resolution is async and happens where hooks are materialized, not
during workspace discovery. Discovery only needs `orphan` and each project's
existence, and those come from the main config alone. This keeps the parallel
directory walker synchronous and network-free, and it means the workspace
discovery cache needs no changes. Local includes are re-read on every
resolution, so editing one takes effect on the next run without `--refresh`.

| Command | Include behavior |
| -- | -- |
| `prek run`, `prek hook-impl` | Resolve with network allowed, then collision check, then build hooks. `--refresh` revalidates unpinned includes. |
| `prek list` | Same as `run`, so included hooks are listed. |
| `prek exec` | Same as `run`. |
| `prek install --prepare-hooks`, `prek prepare-hooks` | Same as `run`, so hook environments for included hooks are prepared. |
| `prek install` (shims only) | No resolution. `default_install_hook_types` comes from the main config only. |
| `prek validate-config` | Resolves includes with network allowed and reports include errors and collisions. It becomes async, receives the `Store`, and holds `store.lock_async()` while resolving, the same way `run` does, because Git include clones and the include cache write to the shared store. It never prompts and never writes the trust store. A changed unpinned include is reported as a warning, because the configuration itself is valid. |
| `prek util trust-includes` | New. Resolves with network allowed and records accepted digests, see [Accepting changes explicitly](#accepting-changes-explicitly). |
| `prek update` | No include resolution. Updates `repos` and Git include `rev`s in the main config, see [`prek update` and Git includes](#prek-update-and-git-includes). Included files are never rewritten. |
| `prek cache gc` | Best-effort cache-only resolution, see below. Never uses the network or clones. |
| `prek util yaml-to-toml` | Converts `includes` entries, both string and table forms. Included files are not converted or fetched. |
| `prek try-repo` | Unaffected. The generated config has no includes. |
| Shell completion | Best-effort cache-only resolution. Missing sources are skipped, and hooks from local, already cached remote, and already cloned Git includes are offered. |
| `check-hooks-apply`, `check-useless-excludes` meta hooks | Strict cache-only resolution. The surrounding `run` has already populated the cache, so a missing source is an error. |

### Staged configuration check

When `prek run` requires a clean worktree, it currently fails if a project
config file is not staged. Local includes that live inside the Git worktree are
added to that check, because an unstaged include changes which hooks run just as
an unstaged main config would. Local includes outside the worktree, remote
includes, and Git includes are not checked. The check uses the parsed `includes` paths and does
not need network resolution.

### Cache GC

`store.track_configs` keeps tracking main config files only. `prek cache gc`,
for each tracked config that still exists:

1. Parses its `includes` without network access, with best-effort cache-only
   resolution.
2. Keeps `entries/<url-key>.json` for every remote URL listed, and the blobs
   those entries name.
3. Keeps `blobs/<pin>` for every pinned include.
4. Marks the store clone of every Git include as used, through the same
   `store.repo_path` key logic used for hook repositories.
5. Resolves local includes, cached remote includes, and already cloned Git
   includes, so remote repositories and hook environments referenced only by
   included files are also kept.

Entries, blobs, and Git include clones that nothing references are removed.
`--dry-run` and `--verbose` report them in an `includes` section, like other
removed items.

Include cache keys are global, and `config-tracking.json` only records main
config paths, so GC cannot tell which entries, blobs, or clones belonged to a
config it cannot parse. If any tracked config, or any of its local or cached
includes, fails to parse, GC skips the include sweep for that run: no include
entries, include blobs, or Git include clones are removed. `--verbose` names the
config that caused the skip. The other sweeps behave as they do today.

## Workspace behavior

- Each project resolves its own includes. Includes are not inherited from a
  parent project, and `orphan` is unchanged.
- Resolution runs only for **selected** projects, the ones `init_hooks`
  receives. A broken include in a project outside the selection does not fail
  the run.
- Fetches are deduplicated across projects, as described in
  [Fetching](#fetching). Git include clones are deduplicated by the store key,
  as described in [Cloning](#cloning).
- Collisions are checked per project.

## Security

Included files are executable project configuration. A remote or Git include
can add `repo: local` hooks whose `entry` runs arbitrary commands, so including
one grants whoever controls that URL or repository code execution on every
contributor's machine and in CI.

`docs/security.md` gets a new section, "Review and pin included
configurations":

- Treat an include URL like a dependency. Prefer hosts your organization
  controls.
- An unpinned include can change without a change in your repository. `prek`
  records the content it first saw on each machine, and asks before using
  changed content, or fails in CI. Review the diff before accepting. Setting
  `PREK_ACCEPT_INCLUDE_CHANGES=1` in CI turns this protection off for that job,
  so prefer pinning there.
- A `sha256` pin makes the included content immutable. Review the content
  before updating the pin.
- A pin is only as good as your review. `prek` checks that the bytes match, not
  that they are safe.
- Credentials in include URLs are rejected. For private shared configuration,
  use a Git include, which authenticates through your Git credential setup.
- For Git includes, a commit SHA `rev` is the equivalent of a `sha256` pin. A
  tag can be moved by the repository maintainer, and a branch is not a pin.

For a pinned HTTPS include, `prek` never runs hooks from content whose digest
does not match the pin. For any include, it never runs hooks from content that
failed to parse. Unpinned HTTPS includes, and Git includes at a tag or branch,
run whatever the source serves, so they are only as trustworthy as that
source.

## Compatibility

- `includes` is `prek`-only. Upstream `pre-commit` warns about an unexpected
  root key and runs without the included hooks. Older `prek` versions do the
  same through the unused-key warning. The reference docs recommend setting
  `minimum_prek_version` to the first version that supports includes, so older
  `prek` fails instead of silently skipping hooks.
- Configurations without `includes` behave exactly as before. No network access
  happens, no cache directory is created, and no messages change.
- `docs/compatibility.md` lists `includes` among the `prek`-only extensions.

## Implementation

### New and changed types

`crates/prek/src/config/include.rs` (new):

```rust
/// One entry of the top-level `includes` list, as written in the config.
pub(crate) enum Include {
    Local { path: PathBuf },
    Remote { url: reqwest::Url, sha256: Option<Sha256Digest> },
    Git { repo: RemoteRepo, path: RelativeIncludePath },
}

/// A configuration file included by a project.
pub(crate) struct IncludedConfig {
    pub repos: Vec<Repo>,
    pub priorities: BTreeMap<PriorityAlias, u32>,
    pub minimum_prek_version: Option<String>,
    _unused_keys: BTreeMap<String, serde_json::Value>,
}

/// Where a set of hook definitions came from, for ordering and messages.
pub(crate) enum ConfigSource {
    Main(PathBuf),
    LocalInclude(PathBuf),
    RemoteInclude(reqwest::Url),
    GitInclude { repo: String, rev: String, path: PathBuf },
}
```

`RelativeIncludePath` is a newtype whose constructor rejects absolute paths and
paths that escape the root after normalization. Once an `Include::Git` exists,
its path is known to be safe to join onto a checkout, and the type system
enforces that instead of a runtime check at the join.

- `Include` gets a hand-written `Deserialize` visitor that accepts a string or
  a map. It follows the style of the `Repo` visitors in `config/repo.rs`, so
  every rule in [Include entries](#include-entries) fails with a positioned
  serde error. It also gets a `schemars::JsonSchema` implementation, a `oneOf`
  of string and object.
- `IncludedConfig` reuses `deserialize_and_validate_minimum_version`. Forbidden
  keys are detected by name in `_unused_keys`, using a
  `FORBIDDEN_INCLUDE_KEYS` constant next to `EXPECTED_UNUSED`. This reuses the
  existing unused-key mechanism instead of adding a second parser.
- `Include::Git` reuses `RemoteRepo`, so `resolve_relative_repo_sources` covers
  relative repository paths for Git includes. The loop gains a second pass over
  `includes` next to the existing pass over `repos`.
- `Config` gets `#[serde(default)] pub includes: Vec<Include>`.
  `load_config` resolves relative local include paths against the config file's
  directory, next to `resolve_relative_repo_sources`, so later stages only see
  absolute paths.
- `config::IncludeError` (thiserror) has one variant per hard error in
  [Error model](#error-model). `workspace::Error` gets
  `Include(#[from] IncludeError)`.

`crates/prek/src/includes.rs` (new) holds fetching, caching, and Git include
reads. Git includes call `Store::clone_repos` directly and add no cache code of
their own:

```rust
pub(crate) enum IncludeFetch {
    /// Use the network for missing or stale entries.
    Network { refresh: bool },
    /// Never use the network or clone. A missing remote entry or Git clone is
    /// an error. Used by the meta hooks, which run after `run` resolved
    /// everything.
    CacheOnly,
    /// Never use the network or clone. A missing remote entry or Git clone is
    /// skipped and reported in the result, after its cache key or store key is
    /// recorded. Used by shell completion and cache GC, where a valid config
    /// may name sources that were never fetched.
    CacheOnlyBestEffort,
}

/// The resolved configuration sources of one project, in hook order.
pub(crate) struct ProjectSources {
    includes: Vec<(ConfigSource, IncludedConfig)>,
}

pub(crate) async fn resolve_includes(
    store: &Store,
    projects: &[Arc<Project>],
    fetch: IncludeFetch,
) -> Result<Vec<ProjectSources>, IncludeError>;
```

Both cache-only modes do no network I/O but share the async signature. The
two synchronous callers, shell completion and cache GC, use
`CacheOnlyBestEffort` through a small
`resolve_includes_cached` wrapper that runs the same cache lookup and parse code
without a runtime. The parse, validation, and collision code is shared, so the
two paths cannot drift.

### Hook construction

- `ProjectInitPlan` changes from a flat `repo_configs` list to a list of
  per-source plans, each carrying its `ConfigSource`, its `priorities` table,
  and its filtered repo configs. `remote_configs_to_clone` and `build_hooks`
  iterate those plans in order.
- Priority resolution moves from `Hook::from_spec`, which reads
  `project.config().priorities`, to plan construction, where the defining
  source's table is known. `HookSpec` carries the resolved `Priority`.
- `Workspace::init_hooks` and `Project::init_hooks` call `resolve_includes`,
  then `check_hook_collisions`, then build plans. Both take the `refresh` flag
  the CLI already passes around.
- `check_hook_collisions(sources: &[(ConfigSource, &[Repo])]) -> Result<(), IncludeError>`
  is a pure function, so it can be unit tested without any I/O.
- The second collision pass runs in `build_hooks` once every hook is built and
  before `init_hooks` returns, so no caller can install or run a hook first. It
  checks each built `Hook`'s `id` and resolved `alias` and records which
  source produced it.

### Update rewriting

The updater matches `rev` sites to configured entries by position today:

- `collect_repo_sources` bails when the number of `rev` lines found by
  `read_frozen_refs` differs from the number of remote repositories.
- `render_updated_yaml_config` bails when the count of `rev:` lines differs.
- `render_updated_toml_config` walks `[[repos]]`, but the TOML `rev =` regex in
  `read_frozen_refs` would also count lines in `[[includes]]` tables.

A config with a block-style Git include therefore breaks `prek update` unless
the mapping changes. This is required even apart from updating includes:

- **YAML:** each `rev:` line belongs to its enclosing top-level key, which is
  the last column-0 `key:` line before it. Lines under `repos` map to remote
  repositories in order, and lines under `includes` map to Git includes in
  order, whichever section comes first in the file.
- **TOML:** sites are found by structure with `toml_edit`: `[[repos]]` tables,
  inline tables in `includes = [...]`, and `[[includes]]` tables. The `rev =`
  detection in `read_frozen_refs` uses the same sections, so frozen comments
  line up.
- If the number of `rev` sites under `includes` doesn't match the number of
  Git includes, for example because of flow-style
  `- {repo: ..., rev: ..., path: ...}`, the updater warns once that Git include
  revisions in that file can't be updated, and still updates `repos`. The
  existing all-or-nothing behavior for `repos` is unchanged.

The types become explicit about which kind of entry they refer to:

```rust
/// What must still hold at a candidate revision.
enum UpdateRequirement<'a> {
    /// The manifest must still define these hook ids.
    Hooks(Vec<&'a str>),
    /// This file must exist and be a valid included configuration.
    IncludeFile(&'a RelativeIncludePath),
}

/// Which `rev` site in a config file a usage refers to.
enum RevSlot {
    Repo(usize),
    Include(usize),
}

/// New revisions for one config file, by site kind.
struct ConfigRevisions {
    repos: Vec<Option<Revision>>,
    includes: Vec<Option<Revision>>,
}
```

- `RepoTarget.required_hook_ids` becomes `requirement: UpdateRequirement`. It
  is part of `RepoTargetKey`, so a hook repository and a Git include with the
  same `repo` and `rev` stay separate targets.
- `RepoUsage.remote_index` becomes `slot: RevSlot`. `remote_count` becomes
  per-kind counts.
- `ProjectUpdates` maps each config file to `ConfigRevisions`, and
  `write_new_config`, `render_updated_yaml_config`, and
  `render_updated_toml_config` take `ConfigRevisions`.
- `evaluate_repo_target` dispatches on `UpdateRequirement`. `Hooks` calls
  `checkout_and_validate_manifest`, and `IncludeFile` calls a new
  `checkout_and_validate_include` next to it in `repository.rs`. That function
  runs the same `git ls-tree` symlink check as resolution, reads the file with
  `git show <rev>:<path>`, and runs the included-file parser.

### Touched files

- `crates/prek/src/config/mod.rs`: `includes` field, include path resolution,
  error variant, and the unused-key and mutable-`rev` warnings for included
  files.
- `crates/prek/src/config/include.rs` (new): entry parsing, `IncludedConfig`,
  and collision checks.
- `crates/prek/src/includes.rs` (new): fetching, cache, freshness, and Git
  include clones through `Store::clone_repos`.
- `crates/prek/src/workspace.rs`: plans, `init_hooks`, and the
  `check_configs_staged` extension.
- `crates/prek/src/hook.rs`: priority resolution input.
- `crates/prek/src/cli/validate.rs`: async validation with includes, taking the
  `Store` and holding its lock during resolution.
- `crates/prek/src/main.rs`: pass the `Store` to `validate_configs`.
- `crates/prek/src/cli/cache_gc.rs`: include cache pruning, Git include clones,
  and included repos.
- `crates/prek/src/cli/completion.rs`: included hook ids.
- `crates/prek/src/hooks/meta_hooks.rs`: included hooks in meta checks.
- `crates/prek/src/cli/yaml_to_toml.rs`: `includes` conversion.
- `crates/prek/src/cli/update/{mod,source,config,repository,display}.rs`:
  section-aware `rev` mapping, Git include targets, include validation, and
  include labels in output.
- `crates/prek/src/include_trust.rs` (new): the trust store, the decision
  table, the prompt, and the diff.
- `crates/prek/src/cli/trust_includes.rs` (new) and `crates/prek/src/cli/mod.rs`:
  `prek util trust-includes`.
- `crates/prek-consts/src/env_vars.rs`: `PREK_INCLUDE_CACHE_TTL`,
  `PREK_ACCEPT_INCLUDE_CHANGES`, and `PREK_INTERNAL__INCLUDE_PROMPT_INPUT`.
- `prek.schema.json`: regenerated with `PREK_GENERATE=1`.
- Docs: see [Documentation](#documentation).

## Testing

Tests follow the repository conventions: `insta` snapshots regenerated with
`cargo insta review`, `cmd_snapshot!` for CLI behavior, and focused tests
instead of the full integration suite.

### Test infrastructure

**`TestHttpServer`** in `crates/prek/tests/common/mod.rs`. It uses only
`std::net::TcpListener` on `127.0.0.1:0` and a background thread, with no new
dependency, following the raw listener used in the `http.rs` unit tests. It
provides:

- `url(path) -> String`,
- `set(path, Response { status, body, headers })`, which can change routes
  between commands in one test,
- `requests(path) -> Vec<RecordedRequest>` with the request headers, to assert
  request counts and conditional headers,
- `close()` to simulate an unreachable host (the port refuses connections), and
- a snapshot filter that maps `127.0.0.1:<port>` to `[SERVER]`.

Loopback `http` is a supported product rule, so tests need no hidden test-only
escape hatch.

A second filter replaces cache ages such as `cached 3s ago` with
`cached [AGE] ago`.

For unit tests in `includes.rs`, extend the existing `serve_once` pattern into
a `serve_sequence` helper that answers a fixed list of responses and records
requests.

### Unit tests: `config/include.rs`

Parsing, each in YAML and TOML unless marked:

1. `parse_include_string_local_relative`: `ci/a.yaml` becomes `Local`, resolved
   against the config directory after `load_config`.
2. `parse_include_string_local_absolute` and `parse_include_string_tilde`.
3. `parse_include_string_https`: becomes `Remote` with no pin.
4. `parse_include_table_url_with_sha256`: lowercase, uppercase, and
   `sha256:`-prefixed digests parse to the same value.
5. `parse_include_table_path`.
6. `reject_include_table_path_and_url`: positioned error snapshot.
7. `reject_include_table_without_location`.
8. `reject_include_sha256_with_path`.
9. `reject_include_invalid_sha256`: too short, non-hex, and 65 characters.
10. `reject_include_unknown_table_key`: `sha265` is rejected with a positioned
    error. This is a regression guard for silently lost pins.
11. `reject_include_insecure_http`: `http://example.com/a.yaml`.
12. `allow_include_loopback_http`: `localhost`, `127.0.0.1:8080`, `[::1]`.
13. `reject_include_other_schemes`: `file://`, `ftp://`, `git+https://`.
14. `reject_include_userinfo`: `https://u:p@host/a.yaml`.
15. `reject_include_empty`: `""` and `{ path = "" }`.
16. `config_without_includes_defaults_empty`: regression guard. The existing
    `parse_repos` debug snapshots gain `includes: []` and are regenerated once.
17. `include_format_from_url_path`: `a.toml`, `a.toml?token=x`, `a.yaml`, `a`,
    and `a.TOML`.

Included file validation:

18. `included_config_allows_repos_priorities_minimum_version`.
19. `included_config_rejects_each_forbidden_key`: table-driven over
    `FORBIDDEN_INCLUDE_KEYS`, asserting the key and source appear in the error.
20. `included_config_rejects_nested_includes`: dedicated message.
21. `included_config_ignores_ci_and_extension_keys`: no unused-key warning.
22. `included_config_collects_unknown_keys`: paths such as `repos[0].foo` are
    reported against the include source.
23. `included_config_requires_repos`.
24. `included_config_minimum_version_too_new`: uses `VERSION_FILTER`.
25. `included_priority_alias_scoped_to_file`: an include alias works, an alias
    from the main config is rejected in the include, and the reverse is also
    rejected.
26. `remote_include_rejects_path_repo`: relative, absolute, and Windows drive
    paths.
27. `remote_include_allows_url_and_special_repos`: `https://`, SCP-style,
    `file://`, `local`, `meta`, `builtin`.
28. `local_include_resolves_relative_repo_against_include_dir`.

Duplicates:

29. `reject_duplicate_local_include`: `./a.yaml` and `a.yaml`.
30. `reject_duplicate_remote_include`: URLs equal after normalization, such as
    host case and a default port.
31. `reject_self_include`.

Collisions (`check_hook_collisions`, pure):

32. `no_collision_distinct_ids`.
33. `collision_include_vs_main`.
34. `collision_between_includes`.
35. `collision_alias_vs_id`, in both directions.
36. `collision_alias_vs_alias`.
37. `collision_meta_hooks`.
38. `duplicate_within_one_source_allowed`: regression guard for `pre-commit`
    behavior.
39. `collision_error_lists_all_names_in_definition_order`: snapshot of the
    whole message.
40. `collision_ignores_hook_name`: equal `name` values with different `id`s are
    fine.

### Unit tests: `includes.rs`

Each test uses a temp `Store` and a local server.

41. `fetch_miss_populates_cache`: the entry and blob exist, and the blob name
    equals its digest.
42. `fresh_entry_makes_no_request`: the server records zero requests.
43. `stale_entry_revalidates_with_conditional_headers`: `If-None-Match` and
    `If-Modified-Since` are sent. A `304` bumps `fetched_at` and keeps the blob.
44. `stale_entry_replaced_on_200`.
45. `stale_entry_network_failure_falls_back`: returns a `Stale` outcome carrying
    the age and the cause.
46. `miss_network_failure_is_error`: connection refused.
47. `miss_http_error_is_error`: 404 and 500, with the status in the error.
48. `refresh_revalidates_fresh_unpinned`.
49. `pinned_hit_makes_no_request_even_with_refresh`.
50. `pinned_miss_fetches_and_verifies`.
51. `pinned_mismatch_is_error_and_not_cached`: no blob or entry is written.
52. `pinned_corrupt_blob_refetches`: after a successful fetch, the tampered
    blob at the same path is replaced with the verified bytes.
53. `pinned_corrupt_blob_offline_is_error`.
54. `invalid_content_not_cached`: with a stale valid entry, a `200` with broken
    YAML is an error and the old entry and blob are untouched.
55. `oversized_content_rejected`: rejected via `Content-Length` and via a
    chunked body with no length.
56. `redirect_to_insecure_http_rejected`: the final-URL check, unit tested on
    the helper. The per-hop policy is covered by test 166.
57. `corrupt_entry_json_is_miss`, and `unknown_entry_version_is_miss`.
58. `concurrent_fetch_same_url_is_consistent`: two tasks fetch the same URL,
    and the final entry and blob agree.
59. `fetch_deduplicated_within_process`: two projects including one URL produce
    one request.
60. `cache_key_uses_normalized_url`.
61. `ttl_from_env`: default, `0`, a valid value, and an invalid value that warns
    and uses the default.
62. `cache_only_mode_never_requests`: a missing entry is an error, and a stale
    entry is used without a warning.

### Integration tests: `crates/prek/tests/includes.rs` (new)

All of these use `cmd_snapshot!`. Tests that check that hooks did not run use a
local hook that writes a marker file, and assert the file is absent.

Basic behavior:

63. `local_include_runs_included_hooks`.
64. `multiple_includes_run_in_listed_order_before_main`: sequential output
    order in the snapshot.
65. `mixed_local_and_remote_includes`.
66. `toml_main_includes_yaml_and_yaml_main_includes_toml`.
67. `remote_toml_include_by_extension`.
68. `include_with_empty_repos`.
69. `included_local_hook_entry_runs_from_project_root`: the entry script sits
    next to the project, not next to the include.
70. `remote_include_with_remote_repo`: a `create_hook_repo` fixture referenced
    by a `file://` URL from a served include. The repo is cloned and the hook
    runs.
71. `include_applies_main_top_level_settings`: the main config's `exclude` and
    `default_stages` affect included hooks.
72. `include_priorities_schedule_across_sources`: aliases from the include and
    numbers from the main config interleave as expected.

Paths and workspace:

73. `include_relative_to_config_not_cwd`: run from a subdirectory with `--cd`.
74. `include_with_explicit_config_flag`: `--config other/cfg.yaml` resolves
    includes relative to `other/`.
75. `workspace_project_includes_shared_file`: `a/` and `b/` both include
    `../shared.yaml`, and each gets its own hooks with no collision.
76. `workspace_shared_remote_include_fetched_once`: the server sees one request.
77. `workspace_broken_include_in_unselected_project`: `prek run a/` succeeds
    while `b/` has a missing include.
78. `editing_local_include_takes_effect_without_refresh`: regression guard for
    the workspace cache.

Hard errors:

79. `missing_local_include_is_error`.
80. `include_directory_is_error`.
81. `self_include_is_error`, and `duplicate_include_is_error`.
82. `nested_include_is_error`.
83. `forbidden_key_in_include_is_error`: `default_stages`.
84. `remote_first_fetch_failure_is_error`: the server is closed. Exit status and
    the hint are in the snapshot, and no marker file exists.
85. `remote_first_fetch_http_500_is_error`.
86. `pinned_mismatch_is_error`: no hook runs.
87. `collision_include_vs_main_is_error`: no hook runs, including hooks from
    other projects in the same invocation.
88. `collision_between_includes_is_error`.
89. `collision_alias_vs_id_is_error`.
90. `include_minimum_prek_version_is_error`.
91. `remote_include_path_repo_is_error`.
92. `insecure_http_include_is_error`.

Caching:

93. `cached_remote_include_used_while_fresh`: first run fetches, then the server
    closes, and the second run succeeds with no warning and no request.
94. `stale_remote_include_falls_back_with_warning`: `PREK_INCLUDE_CACHE_TTL=0`,
    server closed, a warning, exit success.
95. `refresh_flag_revalidates`: the request count goes up with `--refresh` and
    not without it.
96. `upstream_change_picked_up_after_ttl`: serve v1, then v2 with TTL `0` and
    `PREK_ACCEPT_INCLUDE_CHANGES=1`. The v2 hook runs. Test 182 covers the same
    change without approval.
97. `pinned_include_works_offline_with_refresh`.
98. `broken_upstream_does_not_poison_cache`: after a valid fetch, the server
    returns broken YAML. The run errors, then the server is fixed, and the run
    uses the old cached content if the TTL has not expired, or the new content
    if it has.

Other commands:

99. `validate_config_with_includes`: valid, collision, missing include, and an
    unreachable remote include.
100. `list_shows_included_hooks`: text and JSON output.
101. `run_selects_and_skips_included_hooks`: `prek run <included-id>`, `--skip`,
     and `SKIP=`.
102. `unstaged_local_include_blocks_run`: the include is inside the repo.
103. `include_outside_repo_not_staged_checked`.
104. `git_commit_runs_included_hooks`: `prek install`, then `git commit` goes
     through `hook-impl`.
105. `prepare_hooks_fails_on_missing_include`: `prek install --prepare-hooks`
     errors. This is the regression guard for `warn_parse_error` swallowing
     `NotFound`.
106. `update_ignores_included_repos`: only main config revs change, and the
     include file is byte-identical afterwards.
107. `meta_check_hooks_apply_sees_included_hooks`.
108. `mutable_rev_warning_names_include_source`.
109. `unknown_key_warning_names_include_source`.

### Unit tests: Git includes

In `config/include.rs`:

110. `parse_git_include_table`: `repo`, `rev`, and `path` in YAML and TOML.
111. `reject_git_include_missing_rev`, `reject_git_include_missing_path`, and
     `reject_rev_without_repo`.
112. `reject_git_include_with_url`, and `reject_git_include_with_sha256`.
113. `reject_git_include_special_repo`: `local`, `meta`, and `builtin`.
114. `relative_include_path_rejects_escape`: `../x.yaml`, `a/../../x.yaml`,
     `/abs.yaml`, and a Windows drive path. `a/./b.yaml` and `a/../b.yaml` are
     accepted and normalized.
115. `git_include_relative_repo_resolves_against_config_dir`.
116. `reject_duplicate_git_include`: the same `(repo, rev, path)` twice, with
     `path` spelled differently. The same `(repo, rev)` with two different
     paths is accepted.
117. `git_include_rejects_path_repo_inside_included_file`.
118. `collision_git_include_vs_other_sources`: a `GitInclude` source against the
     main config, a local include, and a remote include.
119. `mutable_rev_warning_covers_git_include`: a branch `rev` is warned about
     with the include source, and tags and SHAs are not.

In `includes.rs`:

120. `git_includes_cloned_in_one_batch`: two projects and three Git includes
     over two `(repo, rev)` pairs produce two clones.
121. `git_include_shares_clone_with_hook_repo`: the same `(repo, rev)` used as a
     hook repository and as a Git include is cloned once.
122. `cache_only_mode_git_include_missing_clone_is_error`: nothing is cloned.

### Integration tests: Git includes

These use the `create_repo` fixture from `tests/common/mod.rs`, which makes a
local Git repository that works as a `repo:` source, so no server is needed.

123. `git_include_runs_included_hooks`.
124. `git_include_at_tag_and_sha`: two tags with different content, and each
     `rev` runs its own hooks.
125. `git_include_branch_rev_does_not_refresh`: new commits on the branch do not
     change the hooks, with and without `--refresh`, and the mutable-`rev`
     warning appears.
126. `git_include_missing_path_is_error`: no hook runs.
127. `git_include_path_escape_is_error`.
128. `git_include_clone_failure_is_error`: a nonexistent repository, and no
     hook runs.
129. `git_include_works_offline_when_cloned`: the source repository is deleted
     after the first run, and the second run succeeds.
130. `git_include_collision_with_local_include_is_error`.
131. `git_include_mixed_with_local_and_remote`: the order of the three source
     kinds is kept.
132. `workspace_shared_git_include_cloned_once`.
133. `git_include_relative_repo_path`: `repo: ../shared-config` resolves
     against the config directory, including with `--config`.
134. `git_include_toml_file`: TOML chosen by the extension of `path`.
135. `validate_config_with_git_include`: valid, missing path, and clone
     failure.
136. `update_bumps_git_include_rev`: `prek update` moves the include `rev` to
     the newest tag, and the include file in the source repository is not
     touched.

### Unit tests: `prek update` and Git includes

In `cli/update/config.rs`:

137. `yaml_rev_sites_includes_before_repos`: each `rev:` line maps to the right
     entry.
138. `yaml_rev_sites_includes_after_repos`.
139. `yaml_rev_sites_git_includes_only`: no remote repositories.
140. `yaml_rev_sites_ignore_non_git_includes`: string, local, and remote entries
     next to one Git include, and only its line counts.
141. `yaml_flow_style_git_include_skipped`: the result says include revisions
     can't be updated, and repository revisions are still rewritten.
142. `yaml_include_rev_keeps_quotes_and_comment`: quote style and a trailing
     non-frozen comment are kept.
143. `yaml_include_rev_frozen_comment_spacing`: an existing `# frozen:` spacing
     is kept, and the default spacing is used for a new one.
144. `toml_rev_sites_inline_includes`.
145. `toml_rev_sites_array_of_tables_includes`: `[[includes]]`.
146. `toml_include_rev_frozen_comment`.
147. `read_frozen_refs_is_section_aware`: YAML and TOML with `rev` sites in both
     sections.

In `cli/update/source.rs` and `cli/update/repository.rs`:

148. `hook_repo_and_git_include_share_repo_source`: one `RepoSource` and two
     targets.
149. `git_include_uses_repo_update_settings`: `update.repos.<repo>` cooldown and
     tag filters apply.
150. `checkout_and_validate_include_missing_path`: the candidate is rejected.
151. `checkout_and_validate_include_invalid_file`: a forbidden key and a parse
     error at the candidate tag are rejected.
152. `checkout_and_validate_include_valid`.

### Integration tests: `prek update` and Git includes

In `crates/prek/tests/update.rs`, using `create_repo` fixtures with several
tags:

153. `update_git_include_to_latest_tag`: YAML and TOML configs.
154. `update_git_include_freeze`: writes a SHA and `# frozen: <tag>`.
155. `update_git_include_cooldown`: a tag inside the cooldown window is skipped,
     and nothing is downgraded.
156. `update_git_include_tag_filters`: `--repo-exclude-tag` and project
     `update.repos` settings.
157. `update_git_include_repo_selector`: `--repo <include repo>` updates only
     the include, and `--exclude-repo` skips it.
158. `update_git_include_bleeding_edge`.
159. `update_git_include_candidate_missing_path`: the failure is reported, the
     `rev` is unchanged, and the exit status is failure.
160. `update_git_include_dry_run`: output snapshot with the include label, and
     the config is byte-identical afterwards.
161. `update_hook_repo_and_git_include_same_source`: both are updated in one
     run, and the source is fetched once.
162. `update_with_flow_style_git_include`: warning, and repositories are still
     updated.
163. `update_workspace_git_includes`: two projects with the same include, each
     config updated.

### Tests added after review

164. `collision_effective_names_second_pass` (unit): a manifest-provided alias
     that matches an `id` in another source is caught by the second pass, and
     the message marks it `(alias from manifest)`.
165. `manifest_alias_collision_is_error` (integration): the error is reported
     and no hook is installed or run.
166. `redirect_hop_to_insecure_http_refused` (unit, `includes.rs`): the server
     redirects to a non-loopback `http` URL, the policy refuses the hop, and no
     request reaches the target. A hop to a URL with user info is refused too.
167. `cache_only_best_effort_skips_missing_entry` (unit, `includes.rs`): a
     missing remote entry is skipped and reported, and other includes still
     resolve.
168. `cache_only_best_effort_skips_missing_git_clone` (unit, `includes.rs`).
169. `string_include_classification` (unit, `config/include.rs`): `C:\x.yaml`,
     `C:/x.yaml`, and `a:b.yaml` are paths, and `file://`, `git+https://`, and
     `s3://` strings are parse errors.
170. `cache_gc_skips_git_clone_sweep_when_config_unparseable`
     (`tests/cache.rs`): an unreferenced Git include clone survives GC while
     any tracked config fails to parse.
171. `cache_gc_skips_include_sweep_when_config_unparseable` (`tests/cache.rs`):
     the same for include entries and blobs, with the `--verbose` reason in the
     snapshot.
172. `git_include_symlink_file_is_error` (integration): `path` names a symlink
     committed in the include repository that points outside the clone.
173. `git_include_symlink_parent_is_error` (integration): a parent directory
     of `path` is a symlink.
174. `checkout_and_validate_include_rejects_symlink` (unit): a candidate tag
     where `path` became a symlink is rejected by `prek update`.
175. `remote_include_total_timeout` (unit, `includes.rs`): a server that keeps
     sending one byte at a time fails once the (shortened) total timeout
     passes, although no single read stalls.
176. `validate_config_holds_store_lock` (integration): `validate-config` with a
     Git include waits for a store lock held by another process, then
     succeeds.

### Tests for change confirmation

177. `trust_store_roundtrip` (unit): write, read, and atomic replace. A corrupt
     file or an unknown `version` gives a warning and an empty store.
178. `trust_decision_table` (unit): every row of the decision table, including
     the env var override, with the prompt answer injected.
179. `trust_skipped_for_pinned_sources` (unit): pinned HTTPS includes, Git
     includes at a full commit SHA, and local includes never touch the trust
     store.
180. `trust_key_for_git_include` (unit): the key covers `repo`, `rev`, and
     `path`, and a different `path` in the same repository is a separate key.
181. `include_first_use_is_accepted` (integration): the first run notes the
     digest on stderr and runs the hooks, and the second run is silent.
182. `include_change_noninteractive_is_error` (integration): after v1 is
     accepted, the server serves v2. The run fails with the snapshot above, and
     no hook runs.
183. `include_change_accepted_by_env` (integration):
     `PREK_ACCEPT_INCLUDE_CHANGES=1` accepts v2 and the next run is silent
     without the variable.
184. `include_change_prompt_yes_and_no` (integration): the answers `y` and `n`
     through `PREK_INTERNAL__INCLUDE_PROMPT_INPUT`. The diff is in the snapshot,
     `y` runs the hooks, and `n` fails without changing the trust store.
185. `trust_includes_command` (integration): `--dry-run` lists changes,
     `--yes` records them, and a following run is silent.
186. `git_include_moved_tag_after_cache_clean` (integration): accept a tag,
     move it in the source repository, run `prek cache clean`, and the next run
     reports the change.
187. `validate_config_reports_pending_change` (integration): a warning, exit
     success, and the trust store is unchanged.
188. `cache_gc_keeps_trusted_blobs` (`tests/cache.rs`): the blob named by the
     trust store survives GC after the cache entry moved on.
189. `include_prompt_does_not_read_stdin` (integration): a `pre-push` run
     through `hook-impl` with ref lines on stdin still gets its refs, and the
     prompt answer comes from the prompt input.

Additions to existing test files:

- `tests/cache.rs`: `cache_gc_keeps_referenced_include_entries`,
  `cache_gc_removes_unreferenced_include_entries` (after the include is
  removed from the config), `cache_gc_keeps_pinned_blob`,
  `cache_gc_keeps_repos_referenced_by_includes`,
  `cache_gc_keeps_git_include_clone`,
  `cache_gc_removes_git_include_clone_after_removal`, and a
  `--dry-run --verbose` output snapshot.
- `tests/yaml_to_toml.rs`: `yaml_to_toml_converts_includes`, covering string
  entries, table entries with `sha256`, and Git include tables.
- `tests/run/completion.rs`: `completion_offers_included_hook_ids`, with a
  local include, a cached remote include, and a cloned Git include.
- `schema.rs::generate_json_schema`: regenerate `prek.schema.json` with
  `PREK_GENERATE=1` and review the new `includes` definition.

### Regression tests

These guard existing behavior, and each maps to a risk this change introduces:

| Risk | Guard |
| -- | -- |
| Configs without includes change behavior | Existing suites pass unchanged, apart from the regenerated `Config` debug snapshots in test 16. |
| Network access without includes | Test 42's zero-request harness, plus a run with a closed server and no includes. |
| `pre-commit` duplicate hook lists break | Test 38, plus an integration run with the same hook twice in one file. |
| Priority aliases resolve against the wrong table | Tests 25 and 72. |
| Missing include swallowed as "no config" | Test 105. |
| Workspace cache hides include edits | Test 78. |
| Silent pin loss from typos | Test 10. |
| Stale fallback hides broken upstream | Tests 54 and 98. |
| `prek update` writes to the wrong file | Tests 106 and 136. |
| A Git include breaks `prek update` rev mapping | Tests 137 to 141, 147, and 162. |
| `prek update` output changes for configs without includes | Existing `tests/update.rs` snapshots pass unchanged. |
| `prek update` modifies local include files | Test 106, plus a byte-for-byte check in test 153. |
| GC deletes caches still in use | The `tests/cache.rs` additions. |
| Git include reads files outside its checkout | Tests 114 and 127. |
| Manifest aliases bypass collision checks | Tests 164 and 165. |
| A Git include reads outside the clone through a symlink | Tests 172 to 174. |
| GC deletes caches of a config it cannot parse | Tests 170 and 171. |
| Unpinned content changes hooks without approval | Tests 182, 184, and 186. |
| The prompt steals Git hook stdin | Test 189. |
| Git includes change hook repository clone behavior | Tests 121 and 132, plus the existing clone and `try-repo` suites passing unchanged. |

## Delivery plan

The implementation lands as five PRs. Each one works on its own and brings its
own tests and docs, so it can ship in a release without the ones after it.

| PR | Scope | Depends on | Size |
| -- | -- | -- | -- |
| 1 | Section-aware `rev` mapping in `prek update` | none | about 1 day |
| 2 | `includes` key, full entry syntax, local includes | none | 3 to 4 days |
| 3 | Git includes | 2 | about 2 days |
| 4 | `prek update` for Git include `rev`s | 1, 3 | about 1 day |
| 5 | HTTPS includes, cache, and pinning | 2 | 3 to 4 days |

PRs 1 and 2 can be developed in parallel. PR 5 only needs PR 2, so it can
proceed alongside PRs 3 and 4. Git includes come before HTTPS includes because
they are cheaper, reuse the existing clone store, and already cover private
configuration.

### PR 1: section-aware `rev` mapping

A refactor of `crates/prek/src/cli/update/` with no behavior change, as
described in [Update rewriting](#update-rewriting): `RevSlot`,
`ConfigRevisions`, and section-aware `read_frozen_refs`,
`render_updated_yaml_config`, and `render_updated_toml_config`. It lands before
any include support so the refactor can be reviewed against unchanged output.

- Tests: 137 to 147. The rewriting functions work on file text, so these tests
  do not need `includes` parsing.
- Guard: every existing `tests/update.rs` snapshot stays byte-identical.

### PR 2: `includes` and local includes

- The `includes` key and the full entry syntax for all three source kinds, with
  every parse-time rule in [Include entries](#include-entries),
  [Remote URL rules](#remote-url-rules), and
  [Git include rules](#git-include-rules).
- Resolution of local includes only. A `url` or `repo` entry that parses fails
  at resolution with a hard error saying that the source kind is not supported
  yet. It is never warned about and skipped, because skipping would silently
  drop hooks.
- The included file format, merge semantics, per-file priorities, collision
  checks, the error model, the staged-config check, and the command behavior
  for local includes: `run`, `list`, `exec`, `prepare-hooks`,
  `validate-config`, completion, the meta hooks, `yaml-to-toml`, and cache GC
  for repositories referenced by local includes.
- The `prek.schema.json` update and the reference, compatibility, and cookbook
  docs for local includes.
- Tests:
  - unit tests 1 to 40 and 110 to 117,
  - integration tests 63, 64, 66, 68, 69, 71 to 75, 77 to 83, 87 to 90, 92,
    and 100 to 109,
  - the local parts of 99,
  - a temporary `unsupported_include_source_is_error` test, which PRs 3 and 5
    replace with their positive tests,
  - `yaml_to_toml_converts_includes`, and the local part of
    `completion_offers_included_hook_ids`,
  - review follow-ups 164, 165, and 169.

### PR 3: Git includes

- [Git includes](#git-includes): cloning through `Store::clone_repos`, reading
  `path`, the resolution matrix, the `GitInclude` source, cache GC marking of
  include clones, and the Git parts of the security docs.
- `validate-config` takes the `Store` and holds its lock during resolution. PR 3
  is the first PR in which validation writes to the store.
- [Change confirmation](#change-confirmation): the trust store, the prompt,
  `PREK_ACCEPT_INCLUDE_CHANGES`, and `prek util trust-includes`, first used for
  Git includes that are not at a commit SHA.
- Tests:
  - unit tests 118 to 122,
  - integration tests 123 to 135,
  - `cache_gc_keeps_git_include_clone` and
    `cache_gc_removes_git_include_clone_after_removal`,
  - the Git part of `completion_offers_included_hook_ids`,
  - review follow-ups 168, 170, 172, 173, and 176,
  - change confirmation tests 177 to 180, 185 to 187, and 189,
  - the regression rows for checkout escapes and hook repository clones.

### PR 4: `prek update` for Git includes

- [`prek update` and Git includes](#prek-update-and-git-includes):
  `UpdateRequirement`, `checkout_and_validate_include`, Git include targets,
  repository selectors, and output labels. The rewriting side is already in
  place from PR 1.
- Tests: 136, 148 to 163, and 174.

### PR 5: HTTPS includes

- [Remote includes](#remote-includes): fetching, the cache layout and atomic
  writes, freshness, `--refresh`, conditional requests, `sha256` pinning, the
  size limit, the `RemoteInclude` source, `PREK_INCLUDE_CACHE_TTL`, cache GC
  for include entries and blobs, and the HTTPS parts of the security docs.
- The `TestHttpServer` helper in `tests/common/mod.rs`.
- Tests:
  - unit tests 41 to 62,
  - integration tests 65, 67, 70, 76, 84 to 86, 91, and 93 to 98,
  - the remote parts of 99,
  - the include-cache tests in `tests/cache.rs`, and the remote part of
    `completion_offers_included_hook_ids`,
  - review follow-ups 166, 167, 171, and 175,
  - change confirmation for HTTPS includes: tests 181 to 184 and 188.

### Releases between PRs

If a release ships before every source kind is supported, a config that uses
an unsupported kind fails with the "not supported yet" error from PR 2. The
reference docs name the first `prek` version for each source kind, and
recommend setting `minimum_prek_version` to it, so that older versions fail
early with the version message instead.

## Documentation

- `docs/reference/configuration.md`: a new "`includes`" section under top-level
  keys, with both formats, entry forms, path rules, the included file format,
  hook order, collision rules, caching, and Git includes. `repos` gets a note
  that included hooks come first.
- `docs/reference/environment-variables.md`: `PREK_INCLUDE_CACHE_TTL` and
  `PREK_ACCEPT_INCLUDE_CHANGES`.
- `docs/security.md`: the section described in [Security](#security).
- `docs/compatibility.md`: `includes` listed as `prek`-only.
- `docs/monorepos.md` or `docs/cookbook.md`: a short recipe for a shared baseline
  include.
- `docs/reference/cli.md`: regenerate if the `--refresh` help text changes to
  mention includes.

## Future work

- Overrides: letting the main config adjust `args`, `files`, or `stages` of an
  included hook by id, instead of redefining it.
- Top-level settings scoped to the hooks of one include.
- Private HTTPS includes with a token from the environment. Git includes already
  cover private sources.
- `prek update --includes`, which writes `sha256` pins for HTTPS includes and
  bumps `rev`s in local included files.
- A setting that requires every remote include to be pinned.
- Include sources in `prek list --output-format=json`.
- Nested includes, with cycle detection.

## Open questions

- **TTL default.** One hour matches the workspace cache and keeps commits
  offline-friendly. A longer default reduces request volume for large teams,
  and a shorter one propagates upstream changes faster.
- **Forbidden keys versus warnings.** This spec rejects project-level keys in
  included files. Warning and ignoring them would let people include a full
  existing `.pre-commit-config.yaml`, as the example in the issue does, at the
  cost of silently different behavior.
- **Relative repository paths inside Git includes.** This spec rejects them.
  Resolving them against the checkout would allow hook repositories vendored
  next to the shared config, but it would make a store path part of the
  configuration.
- **Hook order.** Includes before main follows reading order. Main-first would
  let project hooks such as formatters run before shared checks without
  explicit priorities.
