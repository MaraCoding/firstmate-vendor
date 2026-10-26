# firstmate-vendor

Everything the NDA-safe firstmate install needs, fetched once and verified, so
a work machine installs from here and never from `curl | sh` or the public npm
registry. Install with `~/firstmate/bin/fm-vendor-install.sh --vendor <this dir> --airgap`.

| Path | What | Provenance |
| --- | --- | --- |
| `npm/*.tgz`, `npm/SHA256SUMS` | tasks-axi 0.2.6, quota-axi 0.1.66, gh-axi 0.1.43, chrome-devtools-axi 0.1.44, axi-sdk-js 0.1.13 | `npm pack <name>@<version>` from registry.npmjs.org, 2026-10-24 |
| `npm/lavish-local-1.0.0.tgz` | lavish-local: offline replica of lavish-axi 0.1.90 (same `lavish-axi` bin name, no telemetry, no CDN, no publishing); source in the private `lavish-local` repo, contract and tests in its README | `npm pack` of `~/lavish-local`, 2026-10-25 |
| `archive/lavish-axi-0.1.90.tgz` | stock lavish-axi, reference only: reports every CLI run to `a.kunchenguid.com` (Umami) unless `LAVISH_AXI_TELEMETRY=off`; the stock control track installs it from the registry | `npm pack lavish-axi@0.1.90`, 2026-10-24 |
| `bin/treehouse-v3.1.5-linux-amd64.tar.gz` | treehouse release | github.com/kunchenguid/treehouse/releases/download/v3.1.5/ |
| `bin/no-mistakes-v1.84.0-linux-amd64.tar.gz` | stock no-mistakes release, kept for reference only (ships Umami telemetry) | github.com/kunchenguid/no-mistakes/releases/download/v1.84.0/ |
| `bin/no-mistakes-nda-safe-v1.101.0-linux-amd64` | no-mistakes rebuilt from the v1.101.0 tag without telemetry and with the update check off | built here; see `README-no-mistakes.md` |
| `bin/SHA256SUMS` | hashes for everything under `bin/` | |
| `no-mistakes-nda-safe.patch` | the source patch applied before the rebuild (same as `no-mistakes-nda-safe-v1.101.0.patch`) | |
| `archive/` | the previous v1.84.0 NDA-safe binary and its patch, with `SHA256SUMS`, for rollback | |
| `nm-nda-safe.trace` | strace of the rebuilt binary: the only connect() is its own UNIX socket | |
| `README-no-mistakes.md` | commit, Go version, exact build command, verification, how to rebuild and roll back | |
| `quota-axi-shim/README.md` | the quota-axi audit summary. The shim itself now lives in the firstmate fork as `bin/fm-quota-axi-shim.sh` and `bin/fm-vendor-install.sh` installs it; the copy here is kept for reference only | 2026-10-25 |
| `tools/` | ShellCheck 0.11.0 (firstmate's pinned lint version), hash in `tools/SHA256SUMS` | github.com/koalaman/shellcheck releases |
| `compare/run-compare.sh` (local only, not in the repo) | the secure-versus-control comparison scenario (phases A-E; E = the `/bearings lavish` board via `compare/bench/fm-bench-lavish.sh`) | |
| `compare/bench/` (local only) | task benchmarks (worker path, captain-driven) and the lavish board benchmark | |
| `compare/results-*/` (local only) | one directory per comparison run: timings, test logs, egress audits, `summary.txt` | |
| `TRACKS.md` (local only, not in the repo) | the map of both tracks: users, roots, ports, binaries, how to run, compare, and update each | |

To refresh: re-run the `npm pack` commands and the two release downloads on a
machine that may fetch, regenerate `SHA256SUMS`, and rebuild no-mistakes per
`README-no-mistakes.md`. Then run the egress allowlist test and the egress audit
from the fork (`docs/airgap.md`) before adopting.
