# CyberXYZ DepAlert

Fail the build when a dependency in your manifests is known-malicious, quarantined
by the CyberXYZ day-zero signals, or, optionally, matches an advisory. Same verdict
as the CyberXYZ install-time proxy, in about the time it takes to install the CLI.

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
```

Create the key in the CyberXYZ dashboard under Settings, API keys, and add it as a
repository secret named `XYZ_API_KEY`.

## Inputs

| Input | Default | Meaning |
|---|---|---|
| `api-key` | required | CyberXYZ API key, from a repository secret |
| `fail-on` | `block` | Narrowest verdict that fails the build: `block`, `quarantine`, `alert` |
| `manifests` | auto-detect | Space-separated manifest paths |
| `working-directory` | `.` | Where to auto-detect `package-lock.json`, `requirements*.txt`, `Pipfile.lock`, `poetry.lock`, `go.sum` |
| `api-url` | `https://api.cyberxyz.io` | API base URL |
| `python-version` | `3.11` | Python used to run the CLI |

## Outputs

| Output | Values |
|---|---|
| `verdict` | `allow`, `alert`, `quarantine`, `block`, `error` |
| `exit-code` | `0` clean, `1` block, `2` quarantine, `3` alert, `4` backend unreachable |

The verdict is also exported as `XYZ_VERDICT` for later steps in the same job, and
written to the job summary. The gate fails closed: if the backend cannot be reached
the job fails with exit 4 rather than letting an unchecked package through.

## What it does not do

It scans the manifests present at scan time. It does not watch network traffic or
processes on the runner, and it does not block on lockfile drift. To gate what is
actually fetched during `npm install`, `pip install` or `go get`, put the CyberXYZ
proxy on the runner: see [the CI/CD guide](https://cyberxyz.io/docs/ci.html).

## Pinning

Pin to a full commit SHA if your policy requires it:

```yaml
- uses: CyberXYZSecurity/depalert-action@<sha>  # v1.0.0
```

Licensed under Apache-2.0. The action installs the `cyberxyz-scanner` CLI from PyPI
at run time; the CLI itself is proprietary.
