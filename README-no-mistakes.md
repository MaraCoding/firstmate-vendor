# no-mistakes: NDA-safe build

The binary in this directory is `no-mistakes` built from source with no telemetry and no update checks. It is installed at `~/.no-mistakes/bin/no-mistakes`, and `~/.local/bin/no-mistakes` symlinks to it.

## Provenance

- Upstream: https://github.com/kunchenguid/no-mistakes
- Built from tag **v1.101.0**, commit `21a4e7dc4e16c1f3648ee7c11b8270865c3f36fb`, on 2026-10-25. (The first NDA-safe build was v1.84.0, commit `4822244`; that binary and patch are in `archive/`.)
- Go **1.25.14 linux/amd64**, from the official tarball at `~/.local/go`. Tarball SHA256: `a21ae5633a269bcd7e90cf767e48225633795e99d831742cbf3397064fee7712`, checked against go.dev/dl.
- `no-mistakes-nda-safe` SHA256: `104e8ffec1768297bae026470c92e053b9c0429eb32ab26850404716dbd2e02b`
- Previous NDA-safe build backup: `~/.no-mistakes/bin/no-mistakes.ndasafe-v1.84.0`, SHA256 `147c4f8785c0ae904d8fb07d08ed69f6abd2806fe0fe3e5869f411fa8f28ace4`
- Stock binary backup: `~/.no-mistakes/bin/no-mistakes.stock-v1.84.0`, SHA256 `41fb1c780ef5d204c318eca8fbe73a0a86ae576395b7ae2153e21aacefef0d53`
- `--version` prints: `no-mistakes version v1.101.0-ndasafe (21a4e7d) 2026-10-25T18:58:43Z`

## What the patch changes (`no-mistakes-nda-safe.patch`)

1. **`internal/telemetry/telemetry.go`**: the fallback host `defaultHost` changes from `"https://a.kunchenguid.com"` to `""`. The build also leaves out the `TelemetryHost` and `TelemetryWebsiteID` ldflags. That leaves no website ID and no host, so `Default()` returns the no-op sink.
2. **`internal/update/channels.go`**: `defaultManifestURL()` returns `""` instead of the GitHub release-asset URL. With an empty URL, `fetchReleaseFromManifest` returns `errUpdatesDisabled` and makes no request.
3. **`internal/update/update.go`**:
   - A new `updateCheckDisabledByDefault = true` means `updateCheckDisabled()` is always true. The background check and "new version available" notice are off, as if `NO_MISTAKES_NO_UPDATE_CHECK=1` were always set.
   - `no-mistakes update` prints `self-update is disabled in this build` and exits 0.
4. **`internal/skill/skill.go`** and **`skills/no-mistakes/SKILL.md`**: the docs links to `kunchenguid.github.io` (repo-config protected paths, and since v1.101.0 also CLI intent input) are replaced with plain-text pointers to `docs/reference/repo-config.md` and `docs/reference/cli.md`. This is text only, not a network call. It was changed so no author-hosted domain string is left in the binary.
5. **Tests changed to match the new defaults** (none skipped):
   - `TestDefaultUsesSelfHostedHostWhenHostConfigMissing` is now `TestDefaultIsNoopWhenHostConfigMissing`. It asserts the no-op sink.
   - `TestDefaultManifestURLUsesReleaseAssetCDN` is now `TestDefaultManifestURLIsEmpty`. It asserts an empty URL and `errUpdatesDisabled`.
   - `TestUpdaterMaybeNotifyAndCheck` and `TestUpdaterCachedLatestVersion` call a new `enableUpdateCheckForTest(t)` helper, so the notification logic is still tested.
   - New `TestUpdateCheckDisabledByDefault`.

Still in the binary, as opt-in only: `NO_MISTAKES_UMAMI_HOST` and `NO_MISTAKES_UMAMI_WEBSITE_ID`. Telemetry turns on again only if someone sets **both** env vars on purpose. Nothing sets them by default.

## Build command

```sh
export PATH=$HOME/.local/go/bin:$PATH GOTOOLCHAIN=local GOFLAGS=-mod=mod CGO_ENABLED=0
DATE=$(date -u +%Y-%m-%dT%H:%M:%SZ)
go build -trimpath -ldflags "\
  -X github.com/kunchenguid/no-mistakes/internal/buildinfo.Version=v1.101.0-ndasafe \
  -X github.com/kunchenguid/no-mistakes/internal/buildinfo.Commit=21a4e7d \
  -X github.com/kunchenguid/no-mistakes/internal/buildinfo.Date=$DATE" \
  -o ~/firstmate-vendor/no-mistakes-nda-safe ./cmd/no-mistakes
```

Do not use `make build`. The Makefile adds the umami host and website ID by default.

## Upgrade to v1.101.0 (2026-10-25)

- The v1.84.0 patch conflicted in `internal/update/update.go` and `update_test.go`, where upstream added Nix-install handling. Resolved by keeping both: the Nix guards stay, `updateCheckDisabled()` replaces the env-only check, and `run()` refuses Nix installs first, then refuses with `errUpdatesDisabled` when the manifest URL is empty.
- Every new outbound HTTP call added between v1.84.0 and v1.101.0 was reviewed. They all go to local agent servers (opencode v2 `/api/event`, managed-server health checks) or live in the test-only `cmd/fakeagent`. There are no new third-party hosts.
- `go test ./...` passes in full. `strings` finds 0 matches for `kunchenguid.com`, `a.kunchenguid` and `github.io`. `fm-egress-audit.sh` over `--version`, `doctor`, `status`, `stats` and `update` (with both env opt-outs unset) shows `connects: 0`, `verdict: clean`. `update` prints `self-update is disabled in this build`.
- After install, the daemon's `/proc/<pid>/exe` is the new binary and has no TCP or UDP sockets. `fm-bootstrap.sh` detect reaches only `api.github.com` (forge).
- The state database may be migrated by v1.101.0. Rolling back to v1.84.0 is untested against a migrated database.

## Verification of the first build (v1.84.0, 2026-10-24)

- `go test ./internal/telemetry/... ./internal/update/... ./internal/skill/...` all pass.
- `strings no-mistakes-nda-safe | grep -c` finds 0 matches for `kunchenguid.com`, `a.kunchenguid`, `umami` and `github.io`.
  - A case-insensitive `UMAMI` search matches only the two opt-in env var names.
  - `kunchenguid/no-mistakes` module-path strings remain, as expected.
- `strace -f -e trace=connect` was run with `NO_MISTAKES_TELEMETRY` and `NO_MISTAKES_NO_UPDATE_CHECK` both unset, for `--version`, `doctor`, `stats` and `status`. The output is in `nm-nda-safe.trace`. There was exactly one `connect()` across all four runs:
  `connect(7, {sa_family=AF_UNIX, sun_path="/home/molt_admin/.no-mistakes/socket"}, 39) = 0` (from `doctor`). There were no AF_INET or AF_INET6 connects.
- After installing, `systemctl --user is-active no-mistakes-daemon-0ec9c933.service` returned `active`. The daemon's `/proc/<pid>/exe` is the new binary, and `no-mistakes status` works.
- The systemd drop-in `no-phone-home.conf` (`NO_MISTAKES_TELEMETRY=off`, `NO_MISTAKES_NO_UPDATE_CHECK=1`) is still in place as a second safeguard.

## How to rebuild or upgrade

```sh
git clone --depth 1 --branch vX.Y.Z https://github.com/kunchenguid/no-mistakes nm && cd nm
git apply ~/firstmate-vendor/no-mistakes-nda-safe.patch   # re-check hunks on newer versions
grep -rn 'kunchenguid.com\|github.io\|https://github.com/" + repo' --include=*.go internal cmd | grep -v _test.go
go test ./internal/telemetry/... ./internal/update/... ./internal/skill/...
# build as above, run the strings and strace checks, then:
cp ~/firstmate-vendor/no-mistakes-nda-safe ~/.no-mistakes/bin/no-mistakes.new
mv -f ~/.no-mistakes/bin/no-mistakes.new ~/.no-mistakes/bin/no-mistakes   # atomic; avoids ETXTBSY
systemctl --user restart no-mistakes-daemon-0ec9c933.service
```

Never run `no-mistakes update` to upgrade. In this build it does nothing, and a stock build would replace the patched binary.

To roll back to the previous NDA-safe build: `cp ~/.no-mistakes/bin/no-mistakes.ndasafe-v1.84.0 ~/.no-mistakes/bin/no-mistakes.new && mv -f ~/.no-mistakes/bin/no-mistakes.new ~/.no-mistakes/bin/no-mistakes`, then restart the unit. (`no-mistakes.stock-v1.84.0` is the stock telemetry build; do not roll back to it on the secure track.)
