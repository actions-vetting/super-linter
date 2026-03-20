# GitHub Action Review Report: Super-Linter

**Review Date:** 2026-03-20  
**Action:** [super-linter/super-linter](https://github.com/super-linter/super-linter)  
**Reviewed Version:** v8.5.0  
**Reviewer:** Enterprise Security Team

---

## 1. What Does the Action Do?

Super-linter is a **ready-to-run collection of linters and code analyzers** packaged
as a Docker-based GitHub Action. It validates source code against best practices
and consistent formatting rules across **60+ programming languages** in a single
CI step.

**Key capabilities:**

- Runs 60+ linters/formatters covering languages such as Python, JavaScript/TypeScript,
  Go, Java, Bash, Terraform, Kubernetes manifests, Dockerfiles, and many more.
- Can lint all files in a repository or only changed files in a pull request.
- Supports "fix mode" to auto-fix certain linting issues.
- Posts linting results as GitHub commit statuses and PR comments.
- Generates a GitHub Actions Step Summary with detailed results.
- Supports SSH key setup for private repository access.
- Supports custom SSL certificates for enterprise environments.
- Available in two variants: **standard** (all linters) and **slim** (subset).

**How it works:**

The action runs a pre-built Docker image (`ghcr.io/super-linter/super-linter:v8.5.0`)
from the GitHub Container Registry. The Docker image is based on Alpine Linux and
bundles all linter binaries and their dependencies. The entrypoint is a Bash script
(`lib/linter.sh`) that orchestrates file detection, linter configuration, parallel
execution, and result reporting.

---

## 2. Information About the Creators

- **Organization:** [super-linter](https://github.com/super-linter) (GitHub Organization)
- **Original Copyright:** GitHub, Inc. (2020) — originally created by GitHub staff
- **Current Maintainers:** `@Hanse00`, `@ferrarimarco`, `@zkoppert` (listed in
  Dockerfile labels and issue assignment templates)
- **License:** MIT License
- **Contributors:** [200+ contributors](https://github.com/super-linter/super-linter/graphs/contributors)

The project was originally created by GitHub itself and is now maintained by a
community of open-source contributors under the `super-linter` GitHub organization.
This is a **well-known, high-profile** Action in the GitHub ecosystem.

---

## 3. Repository and Release Information

| Metric | Value |
|--------|-------|
| **GitHub Stars** | ~10,365 |
| **Forks** | ~1,060 |
| **Open Issues** | ~45 |
| **Repository Created** | October 2019 |
| **Last Updated** | March 2026 (actively maintained) |
| **Latest Release** | v8.5.0 (February 7, 2026) |
| **Release Cadence** | Regular (monthly/bi-monthly) |
| **Archived** | No |

**Recent Release History:**

| Version | Date |
|---------|------|
| v8.5.0 | 2026-02-07 |
| v8.4.0 | 2026-01-28 |
| v8.3.2 | 2025-12-24 |
| v8.3.1 | 2025-12-16 |
| v8.3.0 | 2025-11-28 |
| v8.2.1 | 2025-10-16 |
| v8.2.0 | 2025-09-30 |
| v8.1.0 | 2025-08-21 |
| v8.0.0 | 2025-07-18 |

**Assessment:** The project is **actively maintained** with regular releases,
active issue triage (stale bot configured), Dependabot for dependency updates,
and conventional commit enforcement. It uses `release-please` for automated
semantic versioning.

---

## 4. External Systems and Interactions

### 4.1 Systems the Action Interacts With

| System | Direction | Purpose |
|--------|-----------|---------|
| **GitHub REST API** | Import/Export | Posts commit statuses, PR comments, reads event payloads |
| **GitHub Container Registry (ghcr.io)** | Import | Pulls the pre-built Docker image |
| **GitHub SSH (github.com:22)** | Import | Optional SSH key setup for private repo access |
| **GitHub Meta API** | Import | Fetches SSH host key fingerprints for verification |
| **Alpine Linux APK repos** | Import | Installs OS packages at runtime (if configured) |
| **GNU Savannah** | Import (build-time only) | Clones chktex source during Docker image build |

### 4.2 Data Imported

- **GITHUB_TOKEN**: Used for API authentication (commit statuses, PR comments)
- **SSH_KEY** (optional): Private SSH key for repository access
- **SSL_CERT_SECRET** (optional): Custom SSL certificate for enterprise environments
- **GitHub Event JSON**: Webhook payload for determining changed files
- **Repository source code**: The code to be linted (mounted into the container)

### 4.3 Data Exported

- **Commit statuses**: Posted to GitHub API for each language linted
- **PR comments**: Summary of linting results posted as issue comments
- **Step Summary**: Markdown summary appended to `GITHUB_STEP_SUMMARY`
- **Log file** (optional): `super-linter.log` created if `CREATE_LOG_FILE=true`
- **No external telemetry**: Dart and .NET telemetry are explicitly disabled

---

## 5. Code Quality Assessment

**Overall Code Quality: 8 / 10**

### Strengths

- **Well-structured Bash codebase**: Clear separation of concerns with modular
  function files (`lib/functions/*.sh`) and global configuration (`lib/globals/*.sh`).
- **Comprehensive test suite**: 15+ unit test files covering all major functions,
  integration tests, and a test runner (`test/run-super-linter-tests.sh`).
- **Defensive coding practices**: `set -o nounset`, `set -o pipefail` enabled;
  proper quoting throughout; safe array handling with `mapfile`/`readarray`.
- **Input validation**: Boolean variables strictly validated; integer inputs
  checked; path existence verified; conflicting configurations detected.
- **CI/CD hygiene**: Conventional commits enforced, automated releases,
  Dependabot updates, automated stale issue management.
- **Docker multi-stage builds**: Efficient image construction with 20+ build
  stages for dependency isolation.
- **Parallel execution**: Uses GNU `parallel` for concurrent linter execution,
  with proper result collection.

### Areas for Improvement

- **`eval` usage in validation.sh**: While safe due to hardcoded language lists,
  `declare` or `printf -v` would be cleaner alternatives.
- **Workflow Actions not SHA-pinned**: All third-party Actions use tag pinning
  (e.g., `@v6`) rather than full commit SHA pinning.
- **No GPG verification** of cloned source code during image build (chktex from
  GNU Savannah).

---

## 6. Security Risk Assessment

**Overall Risk: 3 / 10 (Low)**

### 6.1 Positive Security Findings

- ✅ **No data exfiltration detected**: No outbound network calls to untrusted hosts.
- ✅ **No `pull_request_target` triggers**: Prevents untrusted fork code execution
  with elevated permissions.
- ✅ **No script injection vulnerabilities**: All `${{ github.event.* }}`
  references in workflow `run:` blocks use safe, non-user-controllable fields
  (event_name, SHA values, commit counts).
- ✅ **Minimal permissions by default**: Workflows declare `permissions: {}` at
  the top level and grant only necessary permissions per job.
- ✅ **Token not exported by default**: `EXPORT_GITHUB_TOKEN` defaults to `false`;
  the GitHub token is only exported to child processes when explicitly enabled.
- ✅ **HTTPS everywhere**: All network requests use HTTPS with TLS verification.
- ✅ **SSH fingerprint verification**: When using SSH, the action verifies GitHub's
  SSH host key fingerprint against the GitHub Meta API (unless explicitly bypassed).
- ✅ **Telemetry disabled**: Dart, .NET, and PowerShell telemetry are all explicitly
  opted out.
- ✅ **Docker image from trusted registry**: Image pulled from GitHub Container
  Registry (`ghcr.io/super-linter/super-linter`), the same organization's package.
- ✅ **Gitleaks integration**: Secret detection is built into the linter suite.

### 6.2 Security Concerns and Risks

#### Medium Risk

1. **SSH Fingerprint Verification Bypass (`SSH_INSECURE_NO_VERIFY_GITHUB_KEY`)**

   The `setupSSH.sh` script allows disabling SSH host key fingerprint verification,
   which could enable man-in-the-middle attacks on SSH connections:

   ```bash
   # lib/functions/setupSSH.sh
   if [[ "${SSH_INSECURE_NO_VERIFY_GITHUB_KEY}" == "true" ]]; then
     warn "Skipping GitHub SSH key verification..."
     ssh-keyscan -t rsa "${GITHUB_DOMAIN}" >> ~/.ssh/known_hosts
   fi
   ```

   **Mitigation:** This is an opt-in flag that defaults to `false`. Users must
   explicitly enable it.

2. **Workflow Actions Use Tag Pinning Instead of SHA Pinning**

   All third-party Actions in the CI/CD workflows use tag references (e.g.,
   `actions/checkout@v6`) rather than full commit SHA pinning. Tags can be
   force-pushed, potentially allowing supply-chain attacks on the CI/CD pipeline.

   Affected Actions:
   - `actions/checkout@v6`
   - `actions/stale@v10`
   - `actions/github-script@v8`
   - `docker/setup-buildx-action@v4`
   - `docker/build-push-action@v7`
   - `docker/login-action@v4.0.0`
   - `googleapis/release-please-action@v4.4.0`
   - `akhilerm/tag-push-action@v2.2.0`
   - `github-community-projects/contributors@v2`
   - `peter-evans/create-issue-from-file@v6`
   - `dependabot/fetch-metadata@v2`
   - `actions/first-interaction@v1`

   **Mitigation:** These are primarily used in the project's own CI/CD
   workflows, not in the action consumed by users. The action itself is a
   Docker image and does not reference these Actions at runtime.

#### Low Risk

3. **SSL Certificate Not Validated (`updateSSL.sh`)**

   When users provide a custom SSL certificate via `SSL_CERT_SECRET`, the
   certificate format is not validated before being added to the system trust
   store:

   ```bash
   # lib/functions/updateSSL.sh
   echo "${SSL_CERT_SECRET}" >> /tmp/cert.crt
   sudo mv /tmp/cert.crt /usr/local/share/ca-certificates/cert.crt
   sudo update-ca-certificates
   ```

   **Mitigation:** The user is providing their own certificate intentionally.
   The risk is limited to the container's runtime environment.

4. **Runtime Package Installation**

   The `runtimeDependencies.sh` script can install Alpine packages at runtime
   from a user-provided JSON configuration file. This could potentially install
   unexpected packages, though they come from Alpine's signed repositories.

5. **`git config --system --add safe.directory "*"`**

   The Docker image globally marks all directories as safe for Git. This is
   necessary for the action's function but could theoretically mask malicious
   repository configurations in edge cases.

---

## 7. Summary of Findings

| Category | Score |
|----------|-------|
| **Code Quality** | **8 / 10** |
| **Risk Level** | **3 / 10** (Low) |

### Key Takeaways

- **Trustworthy project**: Well-established, actively maintained by known
  contributors, originally created by GitHub. 10,000+ stars and regular
  releases indicate a healthy open-source project.
- **No malicious code detected**: Thorough review of all shell scripts, Docker
  configuration, and GitHub workflows revealed no malicious patterns, data
  exfiltration, or backdoors.
- **Good security posture**: Defensive coding practices, proper input validation,
  minimal permissions, and no dangerous workflow triggers.
- **Minor improvements possible**: SHA pinning for Actions, replacing `eval` with
  `declare`, and certificate validation would further harden the project.

---

## 8. Recommendations for Secure Usage

### 8.1 Pin to a Specific Version with SHA

Always pin to a specific commit SHA rather than a mutable tag to prevent
supply-chain attacks:

```yaml
# ❌ Avoid: Tag can be force-pushed
- uses: super-linter/super-linter@v8

# ✅ Recommended: Pin to the SHA of v8.5.0
- uses: super-linter/super-linter@61abc07d755095a68f4987d1c2c3d1d64408f1f9 # v8.5.0
```

### 8.2 Use Minimal Permissions

Grant only the permissions the Action needs:

```yaml
permissions:
  contents: read       # Required: read repository code
  statuses: write      # Optional: post commit statuses
  pull-requests: write # Optional: post PR summary comments
  issues: write        # Optional: post PR summary comments
```

### 8.3 Do Not Export the GitHub Token Unless Necessary

Keep the default `EXPORT_GITHUB_TOKEN: false` unless specific linters (like
`zizmor`) require it for authenticated API access:

```yaml
env:
  EXPORT_GITHUB_TOKEN: false  # Default, keep unless needed
```

### 8.4 Avoid Disabling SSH Verification

Never set `SSH_INSECURE_NO_VERIFY_GITHUB_KEY: true` in production environments:

```yaml
env:
  # ❌ Never do this in production
  SSH_INSECURE_NO_VERIFY_GITHUB_KEY: true
```

### 8.5 Review Linter Configuration Files

When using `LINTER_RULES_PATH` to provide custom linter configurations, ensure
the configuration files are reviewed and do not disable critical checks.

### 8.6 Consider the Slim Variant

If you don't need all 60+ linters, use the slim variant to reduce the attack
surface:

```yaml
- uses: super-linter/super-linter/slim@v8
```

### 8.7 Restrict `FILTER_REGEX_EXCLUDE` Carefully

When excluding files from linting, avoid overly broad patterns that might skip
security-sensitive files.

---

## Appendix: Third-Party Actions Used in Super-Linter's Own Workflows

These are **only used in the project's CI/CD**, not in the action consumed by users:

| Action | Version | Purpose |
|--------|---------|---------|
| `actions/checkout` | v6 | Clone repository |
| `actions/upload-artifact` | v7.0.0 | Upload build artifacts |
| `actions/download-artifact` | v8.0.1 | Download build artifacts |
| `actions/stale` | v10 | Mark stale issues/PRs |
| `actions/github-script` | v8 | Create issues on failure |
| `actions/first-interaction` | v1 | Greet first-time contributors |
| `docker/setup-buildx-action` | v4 | Set up Docker Buildx |
| `docker/build-push-action` | v7 | Build and push Docker images |
| `docker/login-action` | v4.0.0 | Login to GHCR |
| `googleapis/release-please-action` | v4.4.0 | Automated releases |
| `akhilerm/tag-push-action` | v2.2.0 | Retag Docker images |
| `github-community-projects/contributors` | v2 | Monthly contributor report |
| `peter-evans/create-issue-from-file` | v6 | Create issues from files |
| `dependabot/fetch-metadata` | v2 | Dependabot PR metadata |
