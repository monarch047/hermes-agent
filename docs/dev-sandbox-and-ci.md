# Hermes Agent: Dev Sandbox Architecture & Cloud CI Pipeline

## Overview

The Hermes Agent Dev Sandbox provides an unprivileged, isolated execution environment for testing `scripts/install.sh` and `hermes update` without mutating the developer's host machine. The test suite is fully wired into GitHub Actions to execute parallel matrix jobs on cloud runners.

## Architecture

1. **Isolation Layer (`scripts/sandbox/stage2-run.sh`):**
   - Built on `bwrap` (Bubblewrap) unprivileged user namespaces.
   - Read-only mounts for host runtime (`/usr`, `/lib`, `/bin`) and ephemeral writable tmpfs layers for `$HOME` and `$INSTALL_DIR`.
   - Dedicated network namespace powered by `slirp4netns`.

2. **Network Interception & Proxying (`scripts/sandbox/proxy.py`):**
   - Intercepts outbound HTTP/HTTPS on `127.0.0.1:8080`.
   - Serves local test fixtures from `/work/http/` (e.g. `hermes-agent.nousresearch.com`).
   - Mints ephemeral certificates using a throwaway sandbox CA.
   - Merges throwaway CA with host root CAs into `/work/certs/combined-ca.pem`.
   - Pass-through streaming for external endpoints (PyPI, npm registry, GitHub) with `settimeout(None)` for non-blocking downloads.

3. **Installer & Updater Integration (`scripts/install.sh` & `tests/install/install-update-e2e.sh`):**
   - Headless CLI testing uses `--skip-browser` to skip Playwright, Chromium, and Node.js dependencies.
   - `scripts/dev-sandbox.sh` supports `--install-ref` to test clean installs and in-place updates from older release tags.

## Cloud Execution via GitHub Actions

To run the full E2E matrix in the cloud:

```bash
gh workflow run install-e2e.yml --repo monarch047/hermes-agent --ref main -f route=both -f tag-count=2
```

To monitor the run:

```bash
gh run list --workflow="install-e2e.yml"
gh run view <run-id>
```
