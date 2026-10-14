# CyberXYZ DepAlert · GitHub Action

Fails the build when a dependency in your manifests is known-malicious, quarantined
by the day-zero signals, or (optionally) matches an advisory. Same verdict as the
CyberXYZ install-time proxy.

```yaml
name: CyberXYZ gate
on: [push, pull_request]
jobs:
  depalert:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: CyberXYZSecurity/depalert-action@v1
        with:
          api-key: ${{ secrets.XYZ_API_KEY }}
          fail-on: block   # block | quarantine | alert
```

## Workflow and AI-agent checks (`code-scan`)

In `scan` mode the action also runs two offline checks of the CLI (no API call,
nothing uploaded):

- `xyz code-scan ci .github`: what GitHub runs. Known-compromised actions
  (`actions-cool/issues-helper`, `tj-actions/changed-files` before 46.0.1,
  `reviewdog/action-setup@v1`), unpinned actions in workflows anyone can trigger
  from an issue or discussion, `${{ github.event... }}` script injection,
  `pull_request_target` checkouts, secrets handed to unpinned actions.
- `xyz code-scan agents .`: what AI agents and editors run when the repository is
  opened. Claude Code / Gemini hooks, VS Code `folderOpen` tasks and agent rules,
  which is how the Mini Shai-Hulud worm spreads (a SessionStart hook and a
  folderOpen task starting an obfuscated `.claude/index.js`).

Each check has its own threshold, so turning this on does not fail builds over
findings a repository already had:

- `ci-fail-on: critical` (v1 default): a known-compromised action fails the
  build; everything else `code-scan ci` finds (an unpinned action that gets
  secrets, say) is printed. Set it to `high` once your workflows are clean.
  **A future v2 will default `ci-fail-on` to `high`.**
- `agents-fail-on: high`: an agent hook or editor task with signs of a worm drop
  fails the build. A medium finding, such as a hook that runs your own script with
  nothing else pointing at a drop, is printed and does not fail it.

Accept a reviewed finding with
`# xyz:ignore[rule-id]` on its line (YAML, JSONC, markdown), or in
`.xyz-ignore.yml` for strict JSON such as `.claude/settings.json` (see the CLI
README). `code-scan: "false"` turns both checks off; a `cli-version` without
`xyz code-scan agents` skips that check with a warning.

## Route the job's own installs through CyberXYZ (`mode: protect`)

`scan` checks lockfiles after the fact. `protect` goes first and points every
package manager in the job at the CyberXYZ proxy, the way a company firewall
would, so a malicious version is refused at install time, including packages
pulled in by build scripts that never reach a lockfile. The recommended setup
is `protect` before your installs and `scan` after them:

```yaml
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4        # without registry-url, see below
        with: { node-version: 20 }

      - uses: CyberXYZSecurity/depalert-action@v1
        with:
          api-key: ${{ secrets.XYZ_API_KEY }}
          mode: protect
          network-lock: true   # optional: blackhole the public registries

      - run: npm ci                  # goes through npm-proxy.cyberxyz.io
      - run: pip install -r requirements.txt

      - uses: CyberXYZSecurity/depalert-action@v1
        with:
          api-key: ${{ secrets.XYZ_API_KEY }}
          # mode: scan is the default
```

What `protect` does (it runs `xyz ci protect --format github`):

- registers the job as machine `ci/github/<owner>/<repo>`, so its installs
  show up on the Environments page and in the proxy install log;
- exports `npm_config_registry`, `YARN_REGISTRY`, `YARN_NPM_REGISTRY_SERVER`,
  `YARN_NPM_AUTH_TOKEN`, `PIP_INDEX_URL`, `UV_INDEX_URL`, `UV_DEFAULT_INDEX` and
  `GOPROXY` (no `,direct`: a blocked module is not one retry away from a bypass)
  to the later steps through `$GITHUB_ENV`, with the proxy token masked;
- writes `~/.npmrc`, the user `pip.conf`, `~/.nuget/NuGet/NuGet.Config`
  (`<clear/>` plus the proxy source; private feeds are kept, nuget.org is
  removed) and the go env file, for tools that ignore the environment;
- with `network-lock: true` (Linux/macOS runners with sudo), maps
  registry.npmjs.org, registry.yarnpkg.com, pypi.org, files.pythonhosted.org
  and proxy.golang.org to `0.0.0.0` in `/etc/hosts`. github.com,
  sum.golang.org, the NuGet hosts (not lockable yet) and CyberXYZ hosts are
  never touched.

If the CyberXYZ API is unreachable or the key is wrong, `protect` prints a
warning and the job continues unprotected; it never breaks your build. Set
`strict: true` to fail the step instead.

Notes:

- `actions/setup-node` with `registry-url` points npm at its own `.npmrc`; run
  `protect` after it (the token is written to whichever file
  `NPM_CONFIG_USERCONFIG` names) or drop `registry-url`.
- A repository `nuget.config` that lists nuget.org without `<clear/>` adds
  nuget.org back as a source; use `network-lock` or remove it.
- `pip install --extra-index-url ...` / `PIP_EXTRA_INDEX_URL` sources are not proxied.
- Concurrent jobs of the same repository share one machine name; the newest
  registration replaces the token, and installs from an older job still in
  flight lose their machine attribution (they are not blocked). Give matrix
  jobs distinct names if that matters: run `xyz ci protect --machine-name ...`
  directly.

## Inputs

| input | default | |
|---|---|---|
| `api-key` | (required) | `sk_xyz_` key, stored as a secret |
| `mode` | `scan` | `scan` or `protect` |
| `network-lock` | `false` | protect: blackhole public registries in /etc/hosts |
| `strict` | `false` | protect: fail the step if the proxy could not be configured |
| `cli-version` | `>=1.4.66,<1.5` | `cyberxyz-scanner` to install: a pip range, an exact version (`1.4.70`), or `latest` |
| `fail-on` | `block` | scan: `block`, `quarantine` or `alert` |
| `manifests` | auto-detect | scan: space-separated manifest paths |
| `working-directory` | `.` | scan: where to auto-detect manifests; also the repository root code-scan reads |
| `code-scan` | `true` | scan: also run `xyz code-scan ci` (on `.github`) and `xyz code-scan agents` |
| `ci-fail-on` | `critical` | scan: lowest `code-scan ci` severity that fails the build (`critical`, `high`, `medium`, `low`, `none`); v2 will default to `high` |
| `agents-fail-on` | `high` | scan: lowest `code-scan agents` severity that fails the build |
| `python-version` | `3.11` | scan: Python used to run the CLI |
| `api-url` | `https://api.cyberxyz.io` | API base URL |

## Which CLI version runs

The action installs the `cyberxyz-scanner` CLI from PyPI on every run. By default
it takes the newest 1.4.x release, at least 1.4.66 (`cli-version: ">=1.4.66,<1.5"`),
so fixes to the gate reach your pipeline without a new action release, while a
future 1.5 or 2.0 (which may change flags or exit codes) never arrives unannounced.
The trade-off is reproducibility: two runs a day apart can use different patch
releases. To make runs byte-for-byte repeatable, pin an exact version and bump it
yourself:

```yaml
      - uses: CyberXYZSecurity/depalert-action@v1
        with:
          api-key: ${{ secrets.XYZ_API_KEY }}
          cli-version: "1.4.70"      # exact; or "latest", or any pip range
```

Any PEP 440 specifier works (`~=1.4.66`, `>=1.4.66,<1.5`); a value that starts
with a digit is taken as an exact version.

Licensed under Apache-2.0 (see LICENSE). The action installs the `cyberxyz-scanner` CLI from PyPI at run time; the CLI itself is proprietary.

Publishing checklist (one-time):
1. Copy this directory (action.yml, README.md, LICENSE, tests/) to the public repository `CyberXYZSecurity/depalert-action`, and move `.github/workflows/test-depalert-action.yml` with it, changing `uses: ./integrations/github-action` to `uses: ./`.
2. The `cli-version` default in action.yml is a range (`>=1.4.66,<1.5`), so a CLI patch release needs no action release. A test in xyz-cli keeps the range admitting the version in pyproject.toml; when the CLI moves to 1.5, raise the ceiling (and the floor) and cut an action release. `mode: protect` needs a CLI that has `xyz ci protect`.
3. Tag `v1.0.0` and a moving `v1` tag.
4. Draft a release and tick "Publish this Action to the GitHub Marketplace".
5. Update `XYZ-Web Content/docs/ci.html` to show the 3-line form first.
