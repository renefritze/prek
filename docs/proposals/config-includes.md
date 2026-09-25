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
  requiring it, and let `prek update` move those pins forward the same way it
  moves `rev`s.
- Fail loudly on anything that would change which hooks run in a way the user
  cannot see: an unreachable include with no cached copy, a digest mismatch, or
  two sources defining the same hook.

## Non-goals

- Overriding or patching included hooks. The issue mentions "additions or
  overrides". This spec only supports additions. Two sources defining the same
  hook is an error, not an override. See [Future work](#future-work).
- Nested includes. An included file cannot include other files.
- Authentication for private HTTPS includes. Private sources are supported
  through [Git includes](#git-includes), which use the user's Git credentials.
- Includes in hook manifests (`.pre-commit-hooks.yaml`).
- Rewriting included files with `prek update`. `prek update` does bump the
  `rev` of Git includes and the `sha256` of pinned HTTPS includes, because
  those values live in the main config. See
  [`prek update` and Git includes](#prek-update-and-git-includes) and
  [`prek update` and HTTPS pins](#prek-update-and-https-pins).
- Confirming changed content of unpinned includes on each machine (trust on
  first use). A per-machine approval gives no protection in ephemeral CI, and
  on developer machines it turns every upstream edit into a blocked commit.
  Pinning with `sha256` and updating the pin through `prek update` is the
  reviewable path.
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
- Query strings are allowed, because some hosts need them to select a ref or a
  raw view. They can still carry tokens, so every message and the `url` field
  of the cache entry show the URL with its query replaced by `?…`. The full
  URL is only used to send the request and, hashed, as the cache key.
- The URL is parsed and normalized with `reqwest::Url`. The normalized form is
  used as the cache identity and, with the query redacted, in messages.
- Redirects are checked hop by hop, before the next request is sent. A redirect
  policy is a property of the client, not of a request, so remote includes use
  a dedicated client. It is built with the same settings as
  `http::REQWEST_CLIENT` (TLS backend, custom certificates, proxy), plus a
  `reqwest::redirect::Policy::custom` that refuses a hop to plain `http`, and a
  hop to a URL with user info, and keeps the default limit of 10 hops. A hop to
  loopback `http` is allowed only when the configured URL is itself a loopback
  URL, so a remote host cannot point `prek` at services on the user's machine. Checking only `response.url()` would be too late, because
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
  SHA, or a branch. The existing mutable-`rev` warning applies with its
  existing heuristic, which is loose: it does not warn about a branch with a
  `.` in its name, or about a tag made only of hex digits. So in the docs, a
  **full commit SHA** means exactly 40 or 64 hex characters, and only that
  counts as a pin.
- `path` is relative to the repository root. It must not be absolute, and after
  normalization it must not start with `..`. This lexical check happens at parse
  time.
- No component of `path` may be a symlink in the repository tree, whether it is
  the file itself or a parent directory. A lexical check alone is not enough:
  ordinary reads follow symlinks, so a committed symlink could point outside the
  clone. The file is therefore never read from the working tree. It is read
  from Git objects, in two steps:
  1. `git ls-tree <tree-ish> -- <path>` returns the entry for `path`. Git's
     tree lookup never traverses a symlink or a submodule, so a parent that is
     not a real directory produces no entry, which is reported as a missing
     file. An entry whose mode is not a regular blob (`100644` or `100755`),
     such as a symlink (`120000`) or a submodule (`160000`), is an error.
  2. `git cat-file blob <oid>` reads the object id from that entry, so the
     bytes parsed are exactly the bytes that were checked.
- At run time, `<tree-ish>` is `HEAD` of the store clone. Store clones fetch
  `rev` with `git fetch origin <rev> --depth=1` and check out `FETCH_HEAD`, so a
  tag or branch named by `rev` does not exist as a local ref and cannot be used
  as the tree-ish. During `prek update`, `<tree-ish>` is the candidate
  revision in the updater's own repository, which does fetch tags. Both paths
  call the same function with a different tree-ish.
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
  relative or absolute, is an error, and so is a `file://` URL, which is a
  filesystem path spelled as a URL. A shared file cannot know the layout of the
  machine it runs on. Resolving against a Git checkout is also rejected, because
  the checkout lives in the store and a hook repository nested inside it is
  almost certainly a mistake. Network URLs (anything containing `://` other
  than `file://`, or SCP-style `user@host:path`) and `local`, `meta`, and
  `builtin` are allowed. Users who mirror repositories locally can still map a
  network URL to a local path with Git's `url.<base>.insteadOf`.

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

The position is the hook's index in the merged list, not in its own file. So
every main-config hook's implicit priority grows by the number of included
hooks, and it changes again whenever an include adds or removes a hook, which
for a remote include can happen without any change in the repository. An
explicit number in the main config that was chosen relative to those positions
can then tie with a different hook, and hooks with equal priorities run
concurrently. This spec keeps positional implicit priorities, because changing
them would change scheduling for configs without includes. The reference docs
say that in a config with includes, any hook whose relative order matters
should get an explicit priority, preferably through an alias, and that included
files should not use bare numbers.

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
  includes included hooks. Scoping applies to alias names only: `lint: 10` in
  an include and `lint: 10` in the main config name the same slot.

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

The check uses configured names only. It runs after all of a project's
includes are resolved and before hook selection filters are applied, so
`prek run`, `prek run <hook>`, `--group`, and `prek validate-config` all report
the same collisions for the same config. It needs no hook repository. Only Git
include repositories are cloned before it, because their files are the
configuration data.

An alias that a hook repository's manifest sets, and that the configured hook
does not override, is not part of the check. `HookSpec::from_remote` keeps
such an alias, so a manifest alias can equal a name in another source. That
case behaves like a duplicate within one file: `prek run <name>` selects both
hooks. Checking it would require every hook repository to be cloned, and
`HookInitFilters` deliberately skips cloning repositories whose hooks are
filtered out, so the check would pass or fail depending on which hooks were
selected.

Duplicate selector names **within one source** stay allowed. `pre-commit`
allows listing the same hook twice with different arguments, and existing
configs rely on that. Only collisions across sources are errors.

Projects are independent. Two projects in a workspace may define the same hook
id, including through the same include, as they can today.

Because collisions are errors and this spec has no way to drop or override an
included hook, two includes that both list a common hook, such as
`check-yaml`, cannot be combined. The consuming project cannot fix that on its
own. The include authors have to split the shared hook into its own include,
or the project copies the hooks it needs into its main config. Keeping this an
error leaves room to add overrides later without changing the meaning of
existing configs. See [Future work](#future-work).

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
- Each request also sets a total timeout with `RequestBuilder::timeout`, which
  covers connecting as well. The shared read timeout only fires when no bytes
  arrive, so a server that keeps trickling data could otherwise hold a commit
  indefinitely. The total timeout is 30 seconds when there is no usable cached
  copy, and 5 seconds when revalidating a stale entry. Revalidation sits on the
  commit path and has a fallback, so a dead VPN or a captive portal should cost
  a few seconds once per TTL, not half a minute. Both values are parameters of
  the fetch function so tests can use short values.
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
  Updating a pinned include means changing the digest in the config, by hand or
  with `prek update`, which shows up in review. See
  [`prek update` and HTTPS pins](#prek-update-and-https-pins).

Unpinned includes run whatever the URL serves after the cache revalidates.
When revalidation replaces cached content with different bytes, `prek` prints
a warning naming the URL and the old and new digests, once per URL per
invocation:

```text
warning: Included configuration `https://example.com/org/prek/python.yaml` changed (3a6e…c8b7 -> 9f86…0f08)
```

This makes a change visible, but it does not stop it. Freshness is per machine,
so two machines can run different content for up to one TTL.

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
TTL, and **stale** otherwise. An entry whose `fetched_at` is in the future, for
example after a clock change, is stale.

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
| none | stale or `--refresh` | `200`, invalid content | **Warn** and use cached content. The cache is left unchanged. |
| none | missing | `200`, valid | Cache and use it. |
| none | missing | `200`, invalid content | **Error.** Nothing is cached. |
| none | missing | failure | **Error.** |
| pinned | blob present and verified | not contacted | Use cached content. |
| pinned | blob missing or corrupt | `200`, digest matches, valid | Cache and use it. |
| pinned | blob missing or corrupt | `200`, digest mismatch | **Error.** Nothing is cached. |
| pinned | blob missing or corrupt | failure | **Error.** |

"Failure" means a DNS, connect, TLS, or timeout error, a non-`200` status, a
rejected redirect, or a body over the size limit.

Invalid content with a cached copy is handled like a network failure. An
upstream typo would otherwise block every consumer's commits within one TTL.
The warning includes the parse or validation error, so the broken file is not
hidden. Because the entry's `fetched_at` is not updated, every run keeps
revalidating and warning until upstream is fixed. The same applies when new
content requires a newer `prek` through `minimum_prek_version`: the old
content keeps working and the warning says to upgrade.

The stale-fallback warning is printed once per URL per invocation:

```text
warning: Failed to refresh included configuration `https://example.com/org/prek/python.yaml`; using the copy cached 3h ago
  caused by: error sending request for url (https://example.com/org/prek/python.yaml)
```

For invalid content, the cause is the parse or validation error of the new
content.

### `prek update` and HTTPS pins

`prek update` moves the `sha256` of every pinned HTTPS include in the main
config to the content the URL serves now. This is how a pinned include follows
upstream: one reviewed change to the main config per upstream change, instead
of hand-running `curl | sha256sum`.

- Each pinned include's URL is fetched with the network, ignoring the TTL and
  the cached entry, with the same client, size limit, and timeouts as a first
  fetch. Each distinct URL is fetched once per invocation.
- The new content must parse and validate as an included file, with the same
  code used during resolution. If it does not, or the fetch fails, that include
  fails like a hook repository whose candidate fails validation: the error is
  shown, the `sha256` is left unchanged, and the command exits with a failure
  status.
- If the digest is unchanged, the include is reported as up to date. Otherwise
  the new `sha256` is written in place, keeping quote style and the optional
  `sha256:` prefix, and the verified bytes are stored as `blobs/<sha256>` in the
  cache, so the next run needs no network.
- Unpinned includes are not touched. `prek update` never adds a pin.
- `--repo` and `--exclude-repo` match an HTTPS include by its normalized URL.
  Tag filters, `--freeze`, `--bleeding-edge`, and cooldowns do not apply,
  because a URL has no tags or release dates. The reference docs say that a pin
  update is therefore only as careful as the review of the diff it produces.
- `--dry-run` reports the change and writes nothing, including no cache blob.

Output uses the same update, up-to-date, and failure lines as repositories,
with shortened digests:

```text
[https://example.com/org/prek/security.toml (include)] updating 3a6e…c8b7 -> 9f86…0f08
```

The `sha256` sites in the main config are found the same section-aware way as
`rev` sites, see [Update rewriting](#update-rewriting). A pin written in a YAML
flow-style entry is skipped with the same warning as a flow-style Git include.
In TOML, replacing the string value of `sha256` needs no comment, so inline
tables are updated normally.

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
- After cloning, `path` is read from `HEAD` of the clone as described in
  [Git include rules](#git-include-rules), never from the working tree, and
  handed to the same parser and validator used for local includes. Then it goes
  through the checks in [Included file format](#included-file-format).
- `git::clone_repo` uses the user's Git configuration, so credential helpers,
  SSH keys, and `url.<base>.insteadOf` rewrites work. This is the supported way
  to include files from private repositories.

### Freshness and pinning

Store clones are immutable per `(repo, rev)`. Once the marker exists, the clone
is never fetched again. Git includes inherit that behavior:

- Only a full commit SHA `rev` is pinned. A tag `rev` stays fixed only while
  the existing clone lives: if the maintainer moves the tag, a new clone on
  another machine, in CI, or after `prek cache clean` gets the new content, and
  nothing reports the change. Use a commit SHA when you need the same content
  everywhere.
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
| present or cloned | `path` missing, behind a symlink or submodule, or not a regular file | **Error.** |
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
  existing frozen-mismatch warnings. A TOML inline table, such as an entry of
  `includes = [...]` in the example above, has no place for that comment: a
  comment inside a one-line inline table would swallow the rest of the line,
  including the closing `}`, and a comment after the element would swallow the
  separating comma. Under `--freeze`, a Git include written as an inline table
  is therefore handled like a YAML flow-style entry (see
  [Update rewriting](#update-rewriting)): its `rev` is left unchanged and the
  warning suggests the `[[includes]]` form. Without `--freeze`, inline tables
  are updated normally, because no comment is written.
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
keep their `rev`s, and local include files are never modified. Silently
skipping them would be a regression for a project that moves its repositories
into a shared local include, because `prek update` and tools that parse the
main config, such as pre-commit.ci autoupdate or Renovate, stop seeing them. So
`prek update` reads each local include (no network access is needed) and, for
every one that declares remote repositories, prints one warning per invocation:

```text
warning: `ci/hooks/base.yaml` is included by `.pre-commit-config.yaml` and declares 3 remote repositories. `prek update` does not update included files; update their `rev`s in `ci/hooks/base.yaml`.
```

Remote and Git includes are not read and get no warning, because their owner
updates them.

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
- filesystem `repo:` paths and `file://` URLs in a remote or Git include,
- a remote fetch failure with no usable cached copy,
- a Git include clone failure, or a Git include `path` missing at that `rev`,
- a digest mismatch, and
- hook selector collisions across sources.

The only non-fatal include conditions are the stale-cache fallback for unpinned
remote includes, which covers both failed and invalid refreshes, and the
warning for changed unpinned content.

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
| `prek validate-config` | Resolves includes with network allowed and reports include errors and collisions. It becomes async and receives the `Store`. When a config has remote or Git includes, it holds `store.lock_async()` while resolving, the same way `run` does, because Git include clones and the include cache write to the shared store. A config with only local includes, or none, is validated as before: no network, no store lock. |
| `prek update` | No include resolution. Updates `repos`, Git include `rev`s, and HTTPS include pins in the main config, see [`prek update` and Git includes](#prek-update-and-git-includes) and [`prek update` and HTTPS pins](#prek-update-and-https-pins). Reads local includes only to warn about repositories it does not update. Included files are never rewritten. |
| `prek cache gc` | Cache-only resolution, see below. Never uses the network or clones. |
| `prek util yaml-to-toml` | Converts `includes` entries, both string and table forms. Included files are not converted or fetched. |
| `prek try-repo` | Unaffected. The generated config has no includes. |
| Shell completion | Cache-only resolution. Missing sources are skipped, and hooks from local, already cached remote, and already cloned Git includes are offered. |
| `check-hooks-apply`, `check-useless-excludes` meta hooks | Same as `run`, without `--refresh`. These hooks build a new `Project` from each config file they receive (`load_meta_projects`), which can belong to a project the surrounding `run` did not resolve, so they cannot rely on the cache being populated. For the run's own projects the entries are fresh and no request is made. |

### Staged configuration check

When `prek run` requires a clean worktree, it currently fails if a project
config file is not staged. Local includes that live inside the Git worktree are
added to that check, because an unstaged include changes which hooks run just as
an unstaged main config would. Local includes outside the worktree, remote
includes, and Git includes are not checked. The check uses the parsed
`includes` paths and does not need network resolution.

### Cache GC

`store.track_configs` keeps tracking main config files only. `prek cache gc`,
for each tracked config that still exists:

1. Parses its `includes` without network access, with cache-only
   resolution.
2. Keeps `entries/<url-key>.json` for every remote URL listed, and the blobs
   those entries name.
3. Keeps `blobs/<pin>` for every pinned include.
4. Marks the store clone of every Git include as used, through the same
   `store.repo_path` key logic used for hook repositories.
5. Resolves local includes, cached remote includes, and already cloned Git
   includes, so remote repositories and hook environments referenced only by
   included files are also kept.

Entries and blobs that nothing references are removed. `--dry-run` and
`--verbose` report them in an `includes` section, like other removed items. Git
include clones are not a separate sweep: they live in `repos/` next to hook
repository clones, can be the same directory as one, and are removed by the
existing `repos/` sweep when nothing marks them.

Include cache keys are global, and `config-tracking.json` only records main
config paths, so GC cannot tell which entries or blobs belonged to a config it
cannot parse. If any tracked config, or any of its local or cached includes,
fails to parse, GC skips the include cache sweep for that run: no include
entries or blobs are removed. `--verbose` names the config that caused the
skip. The `repos/` sweep keeps today's behavior, which already removes the hook
repository clones of a config that fails to parse. Git include clones follow
the same rule and are cloned again on the next run. Skipping the whole `repos/`
sweep instead would change GC for every user with one broken config.

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
- A child project that includes a baseline which the parent project also runs
  gets those hooks twice on the child's files, once from each project, unless
  the child is `orphan`. The cookbook recipe for a shared baseline says so.

## Security

Included files are executable project configuration. A remote or Git include
can add `repo: local` hooks whose `entry` runs arbitrary commands, so including
one grants whoever controls that URL or repository code execution on every
contributor's machine and in CI.

`docs/security.md` gets a new section, "Review and pin included
configurations":

- Treat an include URL like a dependency. Prefer hosts your organization
  controls.
- An unpinned include can change without a change in your repository, and the
  new content runs on the next revalidation, on developer machines and in CI
  alike. `prek` warns when it sees new content, but it does not ask. Pin
  anything that runs where secrets are available.
- A `sha256` pin makes the included content immutable. `prek update` moves the
  pin to the current upstream content as a normal change to your config.
  Review the included file's diff, not just the new digest, before merging it.
- A pin is only as good as your review. `prek` checks that the bytes match, not
  that they are safe.
- Credentials in include URLs are rejected. For private shared configuration,
  use a Git include, which authenticates through your Git credential setup.
- For Git includes, a full commit SHA `rev` is the equivalent of a `sha256`
  pin, and `prek update --freeze` writes one. A tag can be moved by the
  repository maintainer, and a branch is not a pin.

For a pinned HTTPS include, `prek` never runs hooks from content whose digest
does not match the pin. For any include, it never runs hooks from content that
failed to parse. Unpinned HTTPS includes, and Git includes at a tag or branch,
run whatever the source serves, so they are only as trustworthy as that
source.

## Compatibility

- `includes` is `prek`-only. Older `prek` versions warn about the unused key
  and run without the included hooks. The reference docs recommend setting
  `minimum_prek_version` to the first version that supports includes, so older
  `prek` fails instead of silently skipping hooks.
- Upstream `pre-commit`, and services built on it such as pre-commit.ci, also
  warn about the unexpected root key and run without the included hooks. No
  setting makes them fail, so CI that runs `pre-commit` stays green while
  skipping the whole shared baseline. The reference docs state this at the top
  of the `includes` section: a config with includes must be run with `prek`.
- `prek validate-config` needs the network and the store for configs with
  remote or Git includes. Configs without them validate exactly as before.
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
    /// skipped and reported in the result, after its cache key or store key is
    /// recorded. Used by shell completion and cache GC, where a valid config
    /// may name sources that were never fetched.
    CacheOnly,
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

`prek cache gc` is already async and calls `resolve_includes` with
`CacheOnly`. Shell completion is the only synchronous caller. It uses a small
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
  is a pure function, so it can be unit tested without any I/O. It runs on all
  sources before `HookInitFilters` drops anything, so its result does not
  depend on which hooks were selected.

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
- `sha256:` lines under `includes` in YAML, and `sha256` keys in the same TOML
  sections, map to pinned HTTPS includes in order, the same way.
- If the number of `rev` sites under `includes` doesn't match the number of
  Git includes, for example because of flow-style
  `- {repo: ..., rev: ..., path: ...}`, the updater warns once that Git include
  revisions in that file can't be updated, and still updates `repos`. The same
  applies to `sha256` sites and pinned HTTPS includes, and, under `--freeze`,
  to Git includes written as TOML inline tables. The existing all-or-nothing
  behavior for `repos` is unchanged.
- Rewriting a `rev` in a TOML inline table only replaces the string value and
  never sets a comment decoration on it, because a comment there would end the
  inline table early.

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

/// New values for one config file, by site kind.
struct ConfigRevisions {
    repos: Vec<Option<Revision>>,
    includes: Vec<Option<Revision>>,
    pins: Vec<Option<Sha256Digest>>,
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
  calls the same Git include reader as resolution, with the candidate revision
  as the tree-ish, and runs the included-file parser.
- HTTPS pins are not `RepoTarget`s, because they have no revisions to select
  from. A separate `update_include_pins` step fetches each pinned URL once,
  validates the content, stores the blob, and fills `ConfigRevisions.pins`. It
  reuses the fetch and validation code in `includes.rs` and the output lines in
  `display.rs`.

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
  section-aware `rev` and `sha256` mapping, Git include targets, include
  validation, HTTPS pin updates, the local include warning, and include labels
  in output.
- `crates/prek-consts/src/env_vars.rs`: `PREK_INCLUDE_CACHE_TTL`.
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
    paths, and `file://` URLs.
27. `remote_include_allows_url_and_special_repos`: `https://`, `ssh://`,
    SCP-style, `local`, `meta`, `builtin`.
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
    YAML returns a `Stale` outcome carrying the parse error, and the old entry
    and blob are untouched. Without a cached entry, the same response is an
    error and nothing is written.
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
62. `cache_only_mode_never_requests`: a stale entry is used without a warning,
    and the server records zero requests.

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
70. `remote_include_with_remote_repo`: a served include references
    `https://example.invalid/hooks`, and the test's global Git configuration
    maps that URL to a `create_hook_repo` fixture with `url.<base>.insteadOf`.
    The repo is cloned and the hook runs. This also covers the documented way
    to use local mirrors, since `file://` is rejected in remote includes.
71. `include_applies_main_top_level_settings`: the main config's `exclude` and
    `default_stages` affect included hooks.
72. `include_priorities_schedule_across_sources`: aliases from the include and
    numbers from the main config interleave as expected. A second step adds a
    hook to the include and snapshots how the implicit priorities of main-config
    hooks shift, which is the behavior the reference docs describe.

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
96. `upstream_change_picked_up_after_ttl`: serve v1, then v2 with TTL `0`. The
    v2 hook runs. Test 184 covers the change warning.
97. `pinned_include_works_offline_with_refresh`.
98. `broken_upstream_does_not_poison_cache`: with TTL `0`, after a valid
    fetch the server returns broken YAML. The run succeeds with the old hooks
    and a warning that carries the parse error. Then the server serves fixed v2
    content, and the next run uses v2.

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
106. `update_ignores_included_repos`: only main config revs change, the
     include file is byte-identical afterwards, and the warning naming the local
     include and its repository count is in the snapshot.
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
117. `git_include_rejects_path_repo_inside_included_file`: a relative path, an
     absolute path, and a `file://` URL.
118. `collision_git_include_vs_other_sources`: a `GitInclude` source against the
     main config, a local include, and a remote include.
119. `mutable_rev_warning_covers_git_include`: the existing heuristic applies
     to Git include `rev`s, and the warning names the include source. `main` is
     warned about, and `v1.4.0` and a SHA are not.

In `includes.rs`:

120. `git_includes_cloned_in_one_batch`: two projects and three Git includes
     over two `(repo, rev)` pairs produce two clones.
121. `git_include_shares_clone_with_hook_repo`: the same `(repo, rev)` used as a
     hook repository and as a Git include is cloned once.
122. `git_include_reads_head_of_shallow_clone`: a Git include at a tag and one
     at a branch are cloned through the shallow path, where neither name exists
     as a local ref, and `path` is read from `HEAD`. This guards against using
     `rev` as the tree-ish.

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
146. `toml_include_rev_frozen_comment`: an `[[includes]]` table gets the
     `# frozen:` comment. Under `--freeze`, an inline-table Git include keeps its
     `rev` and produces the warning, and the output still parses as TOML.
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

164. `manifest_alias_not_a_collision` (integration): a manifest-provided alias
     that equals an `id` in another source is not an error, and
     `prek run <name>` selects both hooks, as for a duplicate within one file.
165. `collision_independent_of_selection` (integration): `prek run`,
     `prek run <unrelated-hook>`, and `prek run --group <group>` report the same
     collision error for the same config.
166. `redirect_hop_to_insecure_http_refused` (unit, `includes.rs`): the server
     redirects to a non-loopback `http` URL, the policy refuses the hop, and no
     request reaches the target. A hop to a URL with user info is refused too.
167. `cache_only_skips_missing_entry` (unit, `includes.rs`): a
     missing remote entry is skipped and reported, and other includes still
     resolve.
168. `cache_only_skips_missing_git_clone` (unit, `includes.rs`).
169. `string_include_classification` (unit, `config/include.rs`): `C:\x.yaml`,
     `C:/x.yaml`, and `a:b.yaml` are paths, and `file://`, `git+https://`, and
     `s3://` strings are parse errors.
170. `cache_gc_repo_sweep_unchanged_when_config_unparsable`
     (`tests/cache.rs`): while one tracked config fails to parse, the `repos/`
     sweep runs as today, and the clones referenced by the other configs,
     including Git include clones, are kept.
171. `cache_gc_skips_include_sweep_when_config_unparsable` (`tests/cache.rs`):
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
     passes, although no single read stalls. With a stale cached entry, the
     shorter revalidation timeout applies and the result is a `Stale` outcome.
176. `validate_config_holds_store_lock` (integration): `validate-config` with a
     Git include waits for a store lock held by another process, then
     succeeds.

### Tests for HTTPS pin updates and the third review

In `crates/prek/tests/update.rs`, with a local server:

177. `update_https_pin_moves_to_current_content`: YAML, TOML `[[includes]]`,
     and a TOML inline table. The pin is rewritten, the blob is cached, and the
     next `prek run` makes no request.
178. `update_https_pin_up_to_date`: the config is byte-identical and the
     up-to-date line is in the snapshot.
179. `update_https_pin_invalid_content_fails`: broken content at the URL fails
     that include, the pin is unchanged, and the exit status is failure.
180. `update_https_pin_ignores_unpinned`: an unpinned include is not fetched,
     and the config is byte-identical.
181. `update_https_pin_dry_run`: the change is reported, the config is
     byte-identical, and no blob is written.
182. `update_https_pin_repo_selector`: `--repo <url>` updates only that pin,
     and `--exclude-repo <url>` skips it.

Elsewhere:

183. `sha256_sites_map_to_pinned_includes` (unit, `cli/update/config.rs`): YAML
     and TOML, with unpinned and Git includes in between. A flow-style pinned
     entry is skipped with the warning.
184. `unpinned_change_warns` (integration): serve v1, then v2 with TTL `0`. The
     v2 hook runs, and the change warning with both digests appears once.
185. `query_string_redacted` (unit, `includes.rs`): a URL with `?token=secret`
     does not show the token in the stale warning, the fetch error, or the
     entry JSON, and the cache key still distinguishes two queries.
186. `redirect_to_loopback_only_from_loopback` (unit, `includes.rs`): a
     redirect from a non-loopback URL to `http://127.0.0.1` is refused before
     the request is sent, and a loopback-to-loopback redirect is followed.
187. `future_fetched_at_is_stale` (unit, `includes.rs`): an entry fetched "in
     the future" is revalidated.
188. `meta_hook_resolves_include_of_unselected_config` (integration):
     `check-hooks-apply` receives a config with a remote include that the
     surrounding run did not resolve, fetches it, and checks its hooks.
189. `validate_config_without_remote_includes_takes_no_lock` (integration):
     with only local includes, `validate-config` succeeds while another process
     holds the store lock.

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
| A broken upstream file blocks commits, or is hidden | Tests 54 and 98. |
| `prek update` writes to the wrong file | Tests 106 and 136. |
| A Git include breaks `prek update` rev mapping | Tests 137 to 141, 147, and 162. |
| `prek update` output changes for configs without includes | Existing `tests/update.rs` snapshots pass unchanged. |
| `prek update` modifies local include files | Test 106, plus a byte-for-byte check in test 153. |
| GC deletes caches still in use | The `tests/cache.rs` additions. |
| Git include reads files outside its checkout | Tests 114 and 127. |
| Collision errors depend on hook selection | Tests 164 and 165. |
| A Git include reads outside the clone through a symlink | Tests 172 to 174. |
| GC deletes include caches of a config it cannot parse, or changes the `repos/` sweep | Tests 170 and 171. |
| Git include reads fail for tag and branch revs | Test 122. |
| `--freeze` corrupts TOML inline tables | Test 146. |
| Unpinned content changes hooks silently | Tests 96 and 184. |
| Tokens in include URLs leak into messages | Test 185. |
| `validate-config` needs the store without remote includes | Test 189. |
| Git includes change hook repository clone behavior | Tests 121 and 132, plus the existing clone and `try-repo` suites passing unchanged. |

## Delivery plan

The implementation lands as six PRs. Each one works on its own and brings its
own tests and docs, so it can ship in a release without the ones after it.

| PR | Scope | Depends on | Size |
| -- | -- | -- | -- |
| 1 | Section-aware `rev` mapping in `prek update` | none | about 1 day |
| 2 | `includes` key, full entry syntax, local includes | none | 3 to 4 days |
| 3 | Git includes | 2 | about 2 days |
| 4 | `prek update` for Git include `rev`s | 1, 3 | about 1 day |
| 5 | HTTPS includes, cache, and pinning | 2 | 3 to 4 days |
| 6 | `prek update` for HTTPS pins | 1, 5 | about 1 day |

PRs 1 and 2 can be developed in parallel. PR 5 only needs PR 2, so it can
proceed alongside PRs 3 and 4. Git includes come before HTTPS includes because
they are cheaper, reuse the existing clone store, and already cover private
configuration.

### PR 1: section-aware `rev` mapping

A refactor of `crates/prek/src/cli/update/`, as described in
[Update rewriting](#update-rewriting): `RevSlot`, `ConfigRevisions` with its
`repos` and `includes` fields, and section-aware `read_frozen_refs`,
`render_updated_yaml_config`, and `render_updated_toml_config`. It lands before
any include support so the refactor can be reviewed against unchanged output.
Output is unchanged for every config `prek update` accepts today. The one
visible difference is that configs with `rev:` lines under other top-level
keys, such as an `x-` key holding YAML anchors, stop failing with a count
mismatch, because those lines no longer count as repository sites.

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
- Scope note: the `prek update` warning for repositories in local includes
  lands here, with test 106.

### PR 3: Git includes

- [Git includes](#git-includes): cloning through `Store::clone_repos`, reading
  `path`, the resolution matrix, the `GitInclude` source, cache GC marking of
  include clones, and the Git parts of the security docs.
- `validate-config` takes the `Store` and holds its lock while resolving
  configs with Git includes. PR 3 is the first PR in which validation writes to
  the store.
- Tests:
  - unit tests 118 to 122,
  - integration tests 123 to 135,
  - `cache_gc_keeps_git_include_clone` and
    `cache_gc_removes_git_include_clone_after_removal`,
  - the Git part of `completion_offers_included_hook_ids`,
  - review follow-ups 168, 170, 172, 173, 176, and 189,
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
  - review follow-ups 166, 167, 171, 175, and 184 to 188.

### PR 6: `prek update` for HTTPS pins

- [`prek update` and HTTPS pins](#prek-update-and-https-pins): the
  `update_include_pins` step, `sha256` site mapping, the `pins` field of
  `ConfigRevisions`, selectors, and output lines.
- Tests: 177 to 183.

### Releases between PRs

If a release ships before every source kind is supported, a config that uses
an unsupported kind fails with the "not supported yet" error from PR 2. The
reference docs name the first `prek` version for each source kind, and
recommend setting `minimum_prek_version` to it, so that older versions fail
early with the version message instead.

## Documentation

- `docs/reference/configuration.md`: a new "`includes`" section under top-level
  keys, with both formats, entry forms, path rules, the included file format,
  hook order and the implicit priority shift, collision rules and how to
  compose includes that share a hook, caching, and Git includes. It opens with
  the warning that `pre-commit` ignores `includes`. `repos` gets a note that
  included hooks come first.
- `docs/reference/environment-variables.md`: `PREK_INCLUDE_CACHE_TTL`.
- `docs/reference/cli.md` or the `prek update` guide: pin updates for HTTPS
  includes and the warning about repositories in local includes.
- `docs/security.md`: the section described in [Security](#security).
- `docs/compatibility.md`: `includes` listed as `prek`-only.
- `docs/monorepos.md` or `docs/cookbook.md`: a short recipe for a shared baseline
  include.
- `docs/reference/cli.md`: regenerate if the `--refresh` help text changes to
  mention includes.

## Future work

- Overrides: letting the main config adjust `args`, `files`, or `stages` of an
  included hook by id, instead of redefining it.
- Dropping included hooks, for example a per-include list of hook ids to leave
  out. This would let two includes that share a hook be combined, and let a
  project opt out of one baseline hook without forking the file.
- Top-level settings scoped to the hooks of one include.
- Private HTTPS includes with a token from the environment. Git includes already
  cover private sources.
- Bumping `rev`s in local included files with `prek update`, instead of only
  warning about them.
- Adding a `sha256` pin to an unpinned HTTPS include with `prek update`.
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
  explicit priorities, and would keep the implicit priorities of main-config
  hooks stable when an include changes, at the cost of shifting the included
  hooks instead.
- **Invalid upstream content.** This spec falls back to the cached copy with a
  warning, which keeps commits working but lets a broken shared file go
  unnoticed by anyone who ignores warnings. The alternative is a hard error,
  which makes an upstream typo an outage for every consumer.
