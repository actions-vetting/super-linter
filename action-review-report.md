# GitHub Action Security Review: super-linter/super-linter

**Review Date:** 2026-03-20
**Action Version Reviewed:** v8.5.0
**Repository:** [super-linter/super-linter](https://github.com/super-linter/super-linter)
**License:** MIT

---

## 1. What Does This Action Do?

Super-Linter is a Docker-based GitHub Action that bundles **60+ linters and code analyzers** into a single, ready-to-run container image. When triggered (typically on pull requests or pushes), it:

1. Detects changed files (via `git diff` or full codebase scan).
2. Classifies files by language/format (Bash, Python, JavaScript, YAML, Dockerfile, Terraform, etc.).
3. Runs the appropriate linter(s) for each detected language **in parallel** using GNU Parallel.
4. Reports results as GitHub commit statuses and/or pull request comments.
5. Optionally operates in **fix mode**, auto-correcting issues in-place.

It supports two image variants:
- **Standard**: All 60+ linters including .NET, Rust, PowerShell, and ARM template tools.
- **Slim**: A lighter image excluding .NET, Rust, PowerShell, and ARM.

The action runs entirely inside a Docker container (`ghcr.io/super-linter/super-linter:v8.5.0`) and does not install anything on the runner host.

---

## 2. Creators and Maintainers

| Attribute | Details |
|-----------|---------|
| **Organization** | [super-linter](https://github.com/super-linter) (GitHub Organization) |
| **Origin** | Originally created by GitHub (copyright 2020), now community-maintained |
| **CODEOWNERS** | `@zkoppert`, `@Hanse00`, `@ferrarimarco` |
| **Contributors** | 250+ contributors (based on GitHub contributor graph) |

The project was originally created under the `github` organization and later transferred to the dedicated `super-linter` organization. The three CODEOWNERS are long-time open-source contributors with significant GitHub presence.

---

## 3. Repository Health and Maintenance

| Metric | Value |
|--------|-------|
| **Stars** | 10,365+ |
| **Forks** | 1,060+ |
| **Open Issues** | ~42 |
| **License** | MIT |
| **Default Branch** | `main` |
| **Primary Language** | Shell (Bash) |
| **Archived/Disabled** | No |
| **Last Push** | 2026-03-20 (same day as review) |

### Release Cadence (Last 10 Releases)

| Version | Date | Gap |
|---------|------|-----|
| v8.5.0 | 2026-02-07 | 10 days |
| v8.4.0 | 2026-01-28 | 35 days |
| v8.3.2 | 2025-12-24 | 8 days |
| v8.3.1 | 2025-12-16 | 18 days |
| v8.3.0 | 2025-11-28 | 43 days |
| v8.2.1 | 2025-10-16 | 16 days |
| v8.2.0 | 2025-09-30 | 40 days |
| v8.1.0 | 2025-08-21 | 34 days |
| v8.0.0 | 2025-07-18 | 66 days |
| v7.4.0 | 2025-05-13 | — |

**Assessment:** The project is **actively maintained** with roughly monthly releases, consistent dependency updates via Dependabot, and active issue triage. It uses `release-please` for automated semantic versioning.

---

## 4. External Systems and Data Interactions

### 4.1 Systems Accessed at Runtime

| System | Purpose | Auth Required | Data Direction |
|--------|---------|---------------|----------------|
| **GitHub REST API** | Report commit statuses, post PR comments | `GITHUB_TOKEN` (Bearer) | **Export**: Linter results (pass/fail per language) |
| **GitHub SSH** (optional) | Clone private repos for linting | `SSH_KEY` env var | **Import**: Repository content |
| **Custom SSL CA** (optional) | Trust internal CAs for GHES | `SSL_CERT_SECRET` env var | **Import**: CA certificate |

### 4.2 Systems Accessed at Build Time (Docker Image)

| System | Purpose | Auth Required |
|--------|---------|---------------|
| **GitHub API** | Download release assets (ktlint, checkstyle, ARM TTK, etc.) | `GITHUB_TOKEN` (Docker build secret) |
| **PyPI** | Install Python linters (pylint, ruff, black, etc.) | No |
| **npm Registry** | Install Node linters (eslint, prettier, etc.) | No |
| **RubyGems** | Install Ruby linters (rubocop, etc.) | No |
| **Packagist** | Install PHP linters (phpcs, phpstan, psalm) | No |
| **CPAN** | Install Perl linters (perl-critic) | No |
| **GHCR (ghcr.io)** | Pull base images and publish final images | `GITHUB_TOKEN` |
| **Docker Hub** | Pull upstream tool images (golang, dotnet, etc.) | No |
| **Alpine APK repos** | Install system packages | No |

### 4.3 Data Exported

- **Linter results** → GitHub Commit Status API (per-language pass/fail)
- **Summary comments** → GitHub Pull Request / Issues API (markdown summary)
- **Log output** → GitHub Actions log stream (stdout/stderr)
- **Fix-mode changes** → Modified files in the workspace (checked out repo)

### 4.4 Data Imported

- **Source code** → Read from `GITHUB_WORKSPACE` (the checked-out repository)
- **GitHub event payload** → Read from `GITHUB_EVENT_PATH` (PR metadata, commit info)
- **Linter configuration** → Read from repository (`.eslintrc`, `.pylintrc`, etc.) or bundled defaults

**No data is sent to any third-party service at runtime.** All network communication is exclusively with the GitHub API of the instance the action runs on.

---

## 5. Code Quality Assessment

**Overall Score: 8 / 10**

### Strengths
- **Strict Bash practices**: All scripts use `set -o nounset` and `set -o pipefail`. Variables are consistently double-quoted (`"${VAR}"`).
- **Modular architecture**: Clean separation into `lib/functions/`, `lib/globals/`, and `scripts/` with single-responsibility modules.
- **Comprehensive error handling**: Fatal errors exit with proper codes; trap handlers ensure cleanup on any exit signal.
- **Extensive test suite**: 50+ Makefile test targets, InSpec compliance tests, and language-specific test cases.
- **Multi-stage Docker build**: Reduces final image size and attack surface by discarding build tools.
- **Parallel execution**: GNU Parallel with proper isolation per linter invocation.
- **Active dependency management**: Dependabot configured for all ecosystems (npm, pip, Ruby, PHP, Docker, GitHub Actions) with daily checks.

### Weaknesses
- **Use of `eval`**: Found in `lib/functions/linterRules.sh` and `lib/functions/validation.sh` for dynamic variable assignment. While inputs are controlled (hardcoded language names), `declare -n` (nameref) would be a safer alternative, and is already used elsewhere in the codebase.
- **CPAN piped install**: The Dockerfile pipes a remote script to `perl` for CPAN module installation (`curl ... | perl -`), which is a supply-chain risk mitigated by retries and `--no-wget`.
- **Large codebase**: ~5,500 lines of shell code across 24 files. Shell is harder to audit for security than higher-level languages.

---

## 6. Risk Assessment

**Overall Risk Score: 3 / 10 (Low)**

| Risk Category | Level | Rationale |
|---------------|-------|-----------|
| **Malicious code** | Very Low | No evidence of data exfiltration, backdoors, or obfuscated code. All network calls target GitHub API only. |
| **Supply chain** | Low | Docker image from GHCR with version tags. Dependencies managed via Dependabot. No unpinned `latest` tags in Dockerfile. |
| **Credential exposure** | Low | `GITHUB_TOKEN` is conditionally exported, never logged. SSH keys passed via stdin. SSL certs written to `/tmp` with default permissions (minor). |
| **Command injection** | Low | Proper quoting throughout. `eval` usage is limited and uses controlled inputs. No user-supplied strings in command construction. |
| **Privilege escalation** | Very Low | Runs in a container. No `docker` commands, no `sudo`, no privileged operations. |
| **Data exfiltration** | Very Low | All outbound calls are to the GitHub API. No calls to external services at runtime. |
| **Availability impact** | Very Low | Action only affects CI pipeline. No writes to production systems. Fix mode only modifies workspace files. |

---

## 7. Detailed Security Findings

### 7.1 Use of `eval` for Dynamic Variable Assignment (Low Risk)

**Location:** `lib/functions/linterRules.sh` (lines 25, 42, 49, 66, 73), `lib/functions/validation.sh`

```bash
# linterRules.sh - Dynamic variable assignment via eval
eval "${LANGUAGE_LINTER_RULES}="
eval "${LANGUAGE_LINTER_RULES}=${LANGUAGE_FILE_PATH}"
eval "export ${LANGUAGE_LINTER_RULES}"
```

**Risk:** If `LANGUAGE_LINTER_RULES` were ever derived from untrusted input, this could lead to command injection. Currently, these values come from a hardcoded language array, so the risk is **low**.

**Recommendation:** Replace with `declare -g` or `declare -n` (nameref), which is already used in other parts of the codebase (e.g., `lib/linter.sh`).

---

### 7.2 SSH Key Verification Bypass Option (Informational)

**Location:** `lib/functions/setupSSH.sh`

The action supports an `SSH_INSECURE_NO_VERIFY_GITHUB_KEY` flag that disables SSH host key verification for GitHub. While this is documented as an insecure option, it exists as a configuration lever.

**Recommendation:** If your organization uses this action, ensure `SSH_INSECURE_NO_VERIFY_GITHUB_KEY` is never set to `true` in your workflows.

---

### 7.3 SSL Certificate Written to `/tmp` (Low Risk)

**Location:** `lib/functions/updateSSL.sh`

```bash
echo "${SSL_CERT_SECRET}" >>"${CERT_FILE}"
```

The SSL certificate secret is written to a file in `/tmp` with default permissions. Within a container, this is generally safe, but any process in the container can read it.

**Recommendation:** Acceptable for the containerized context. No action needed unless running on a shared runner without container isolation.

---

### 7.4 CPAN Install via Piped Remote Script (Medium Risk)

**Location:** `Dockerfile` (line ~213)

```dockerfile
RUN curl --retry 5 --retry-delay 5 -sL https://cpanmin.us/ | perl - -nq --no-wget \
    Perl::Critic@1.156 ...
```

**Risk:** Downloads and executes a script from `cpanmin.us` at build time. If this domain were compromised, arbitrary code could execute during image build.

**Mitigation:** This only affects the **build** of the Docker image, not runtime. The published image on GHCR is the artifact users consume. Pinning by SHA or caching the installer would further reduce risk.

---

### 7.5 GitHub Actions in Workflows Not SHA-Pinned (Medium Risk)

All third-party GitHub Actions used in the project's own CI/CD workflows use **tag-based references** rather than SHA pinning:

```yaml
# cd.yml, ci.yml examples:
- uses: actions/checkout@v6              # Tag-based
- uses: docker/setup-buildx-action@v4   # Tag-based
- uses: docker/login-action@v4.0.0      # Tag-based (specific version)
- uses: docker/build-push-action@v7     # Tag-based
- uses: googleapis/release-please-action@v4.4.0
- uses: akhilerm/tag-push-action@v2.2.0
- uses: dependabot/fetch-metadata@v2
- uses: actions/stale@v10
- uses: github-community-projects/contributors@v2
- uses: peter-evans/create-issue-from-file@v6
```

**Risk:** Tags can be force-pushed or retagged. A compromised upstream action could inject malicious code into the super-linter CI/CD pipeline, potentially affecting published images.

**Note:** This affects the super-linter project's own CI/CD, not the consumers of the action. Consumers reference the Docker image by tag (e.g., `v8.5.0`), not these workflow actions.

---

### 7.6 No Evidence of Malicious Code

After thorough review of all shell scripts, the Dockerfile, and CI/CD workflows:

- **No data exfiltration**: No outbound calls to non-GitHub services at runtime.
- **No backdoors**: No hidden network listeners, reverse shells, or encoded payloads.
- **No obfuscated code**: All code is readable Bash with clear intent.
- **No cryptocurrency miners**: No compute-intensive background processes.
- **No credential harvesting**: Secrets are used only for their intended purpose.

---

## 8. Secure Usage Recommendations

### 8.1 Pin to a Specific Docker Image SHA

Instead of using a mutable version tag, pin to the immutable image digest:

```yaml
# ❌ Less secure: tag can be retagged
- uses: super-linter/super-linter@v8.5.0

# ✅ More secure: immutable image digest
- uses: super-linter/super-linter@<commit-sha>
# The action.yml references: docker://ghcr.io/super-linter/super-linter:v8.5.0
# You can also pull the image digest and reference it directly.
```

To find the commit SHA for a release, visit the [releases page](https://github.com/super-linter/super-linter/releases) and note the commit associated with each tag.

### 8.2 Minimize Token Permissions

```yaml
permissions:
  contents: read          # Read repository content
  statuses: write         # Write commit statuses (if MULTI_STATUS enabled)
  pull-requests: write    # Write PR comments (if summary comments enabled)
```

Do **not** grant broader permissions than needed. Super-linter only needs `contents: read` for basic linting.

### 8.3 Do Not Enable Unnecessary Features

```yaml
env:
  EXPORT_GITHUB_TOKEN: false                    # Default; do not change unless needed
  SSH_INSECURE_NO_VERIFY_GITHUB_KEY: false      # Never set to true
  MULTI_STATUS: false                           # Disable if you don't need per-language statuses
```

### 8.4 Use in a Controlled Environment

- Run on **GitHub-hosted runners** or hardened self-hosted runners.
- The action runs in a Docker container, providing isolation from the host.
- If using self-hosted runners, ensure Docker is configured with appropriate security policies.

### 8.5 Review Updates Before Upgrading

Super-linter releases frequently (monthly). Before upgrading:
1. Review the [CHANGELOG](https://github.com/super-linter/super-linter/blob/main/CHANGELOG.md).
2. Check for new environment variables or changed defaults.
3. Test in a non-production workflow first.

### 8.6 Example Secure Workflow

```yaml
name: Lint Code
on:
  pull_request:
    branches: [main]

permissions:
  contents: read
  statuses: write

jobs:
  lint:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v6
        with:
          fetch-depth: 0

      - uses: super-linter/super-linter@v8.5.0
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
          VALIDATE_ALL_CODEBASE: false
          DEFAULT_BRANCH: main
```

---

## 9. Summary

| Category | Rating |
|----------|--------|
| **Code Quality** | **8 / 10** — Well-structured Bash with strict error handling, modular architecture, and comprehensive tests. Minor deductions for `eval` usage and shell-based complexity. |
| **Risk Level** | **3 / 10 (Low)** — No malicious code found. Minimal attack surface at runtime. All network communication is GitHub API only. Runs in a Docker container with no privilege escalation. |
| **Maintenance** | **Excellent** — Actively maintained with monthly releases, 10k+ stars, 3 dedicated CODEOWNERS, Dependabot for all ecosystems, and automated release management. |
| **Recommendation** | **Approved for use** with standard precautions: pin to specific versions, minimize token permissions, and review changelogs before upgrading. |
