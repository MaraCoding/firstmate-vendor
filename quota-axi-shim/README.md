# quota-axi shim (secure track)

quota-axi 0.1.66 audit, 2026-10-25:

- Static: every URL in `dist/` is a vendor quota or auth endpoint. No telemetry,
  analytics, or crash reporting. The only non-vendor host, registry.npmjs.org,
  lives in the `axi-sdk-js` dependency and is reached solely by the explicit
  `quota-axi update` subcommand, never on a quota read.
- Delegated credential refresh may spawn the `claude` CLI; quota-axi sets
  DISABLE_TELEMETRY, DISABLE_ERROR_REPORTING and DISABLE_AUTOUPDATER for it.
- Runtime (strace, fm-egress-audit.sh): a default read connects to
  api.anthropic.com, chatgpt.com AND api.github.com. The GitHub call is the
  Copilot provider probing `copilot_internal/user` with the `gh` OAuth token
  read from ~/.config/gh/hosts.yml, although no Copilot plan exists.

`quota-axi` (this directory) is a PATH shim installed at ~/.local/bin/quota-axi
that pins reads to `--provider claude,codex` unless the caller passes its own
`--provider`, and refuses `update` while ~/firstmate/config/airgap exists.
After the shim the same audit connects only to api.anthropic.com and
chatgpt.com. firstmate's dispatch arrays only ever name claude and pi/codex, so
nothing it needs is hidden.

Evidence: `audit-<timestamp>/{before-stock,after-shim,control-stock}`.
Control track (user fmcontrol) stays stock and unshimmed on purpose.

Reinstall after an npm upgrade of quota-axi is not needed (the shim execs
~/.npm-global/bin/quota-axi), but re-run the audit after any upgrade.
