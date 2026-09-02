# Security Checks CI/CD Workflow

## DEV-383 — GitHub Actions Security Checks

**Repository:** `devops-intern-Security-check`  
**Workflow:** `.github/workflows/security-checks.yml`  
**Status:** Implemented & Validated  
**Platform:** GitHub Actions

---

## 1. Purpose

This repository demonstrates a practical DevSecOps workflow that integrates multiple security checks into a GitHub Actions CI/CD pipeline.

The workflow is designed to identify common security risks early in the software development lifecycle before changes progress toward build, release, or deployment stages.

### Security controls implemented

| Security Control | Tool | Purpose |
|---|---|---|
| Secret Scanning | Gitleaks | Detect exposed secrets and credentials |
| Dependency Scanning | Trivy | Identify vulnerable software dependencies |
| Static Code Analysis | Semgrep | Detect security and code-quality issues |

---

## 2. Workflow Architecture

```text
                    Developer
                        |
                        v
              Commit / Pull Request
                        |
                        v
                 GitHub Actions
                        |
          +-------------+-------------+
          |             |             |
          v             v             v
      Gitleaks        Trivy        Semgrep
       Secrets      Dependencies     SAST
          |             |             |
          +-------------+-------------+
                        |
                        v
                Security Validation
                        |
                +-------+-------+
                |               |
               PASS            FAIL
                |               |
                v               v
          Continue CI/CD    Remediation
```

---

## 3. Workflow Location

The workflow is stored at:

```text
.github/workflows/security-checks.yml
```

The workflow name displayed in GitHub Actions is:

```text
Security Checks
```

---

## 4. Workflow Triggers

The workflow is configured to execute on the following events.

### Push

The workflow runs when code is pushed to:

```yaml
main
develop
```

### Pull Request

The workflow also executes for pull-request activity.

This provides security feedback before changes are merged.

### Manual Execution

The workflow supports manual execution through:

```yaml
workflow_dispatch:
```

This is useful for manually re-running security validation when required.

---

# 5. Security Checks

## 5.1 Gitleaks — Secret Scanning

Gitleaks is integrated to detect potentially exposed secrets in the repository.

Typical examples include:

- API keys
- Access tokens
- Passwords
- Private keys
- Cloud credentials
- Authentication tokens

The workflow uses:

```yaml
uses: gitleaks/gitleaks-action@v3
```

The repository is checked out with full history:

```yaml
with:
  fetch-depth: 0
```

This allows the scanning process to inspect repository history where applicable.

### Security objective

Prevent credentials and sensitive authentication material from being accidentally committed to source control.

---

# 6. Trivy — Dependency Vulnerability Scanning

Trivy is used to identify known vulnerabilities in application dependencies.

The workflow performs a filesystem vulnerability scan:

```yaml
scan-type: fs
```

The security policy currently focuses on:

```text
HIGH
CRITICAL
```

The relevant configuration is:

```yaml
scanners: vuln
severity: HIGH,CRITICAL
ignore-unfixed: true
exit-code: 1
```

### Pipeline behavior

When a configured HIGH or CRITICAL vulnerability is detected, the Trivy job can return a non-zero exit code.

```text
Vulnerability Found
        |
        v
Severity Check
        |
   +----+----+
   |         |
 HIGH/      Below
CRITICAL   Threshold
   |         |
   v         v
 FAIL      Continue
```

### Security objective

Prevent vulnerable third-party dependencies from progressing through the CI/CD pipeline without review or remediation.

---

# 7. Semgrep — Static Code Analysis

Semgrep is integrated to perform static analysis of the source code.

The workflow executes:

```bash
semgrep scan --config auto
```

Static analysis examines source code without executing the application.

It can identify patterns associated with:

- Security weaknesses
- Unsafe coding practices
- Potential bugs
- Code-quality problems
- Organization-specific security rules

### Security objective

Identify security and code-quality issues as early as possible during development.

---

# 8. GitHub Actions Job Structure

The workflow contains three independent security jobs.

```text
Security Checks
│
├── Secret Scanning - Gitleaks
│
├── Dependency Scan - Trivy
│
└── Static Analysis - Semgrep
```

Each job runs on:

```text
ubuntu-latest
```

This separation makes the workflow easier to troubleshoot and allows each security control to report its own status.

---

# 9. Workflow Permissions

The workflow explicitly defines GitHub Actions permissions:

```yaml
permissions:
  contents: read
  security-events: write
```

### `contents: read`

Allows the workflow to read repository contents required for scanning.

### `security-events: write`

Provides permission for security-event integrations where required.

Explicit permissions help follow the principle of least privilege instead of granting unnecessary repository access.

---

# 10. Complete Workflow

The implemented workflow is maintained in:

```text
.github/workflows/security-checks.yml
```

Core structure:

```yaml
name: Security Checks

on:
  push:
    branches:
      - main
      - develop

  pull_request:

  workflow_dispatch:

permissions:
  contents: read
  security-events: write
```

The workflow then executes Gitleaks, Trivy, and Semgrep as separate jobs.

---

# 11. Security Gate Policy

The workflow establishes a baseline for CI/CD security enforcement.

| Severity / Finding | Recommended Action |
|---|---|
| Critical vulnerability | Block pipeline and remediate immediately |
| High vulnerability | Block protected branch/release pipeline |
| Medium vulnerability | Report and track |
| Low vulnerability | Address during maintenance |
| Exposed secret | Treat as security incident and rotate credential |

### Important

A security finding should not simply be suppressed to make the pipeline pass.

If remediation is temporarily impossible, use a documented security exception containing:

- Business justification
- Finding details
- Risk assessment
- Compensating controls
- Responsible owner
- Review/expiry date

---

# 12. Validation Results

The implemented workflow was successfully executed through GitHub Actions.

### Validation status

| Security Check | Result | Execution Time |
|---|---:|---:|
| Gitleaks Secret Scanning | ✅ PASS | 9 seconds |
| Trivy Dependency Scan | ✅ PASS | 17 seconds |
| Semgrep Static Analysis | ✅ PASS | 19 seconds |

### Overall result

```text
Security Checks
      |
      +-- Gitleaks    ✅ PASS
      |
      +-- Trivy       ✅ PASS
      |
      +-- Semgrep     ✅ PASS
      |
      v
   Workflow PASS
```

The successful run confirms that the three implemented security checks execute correctly in the GitHub Actions environment.

---

# 13. Testing Strategy

The following test cases should be used when extending or maintaining the workflow.

| Test ID | Test Case | Expected Result |
|---|---|---|
| SEC-01 | Normal code change | Workflow executes successfully |
| SEC-02 | Pull request created | Security checks execute |
| SEC-03 | Exposed secret introduced | Gitleaks reports the finding |
| SEC-04 | HIGH/CRITICAL dependency introduced | Trivy can fail the job |
| SEC-05 | Static security issue introduced | Semgrep reports the finding |
| SEC-06 | Clean repository | Security checks pass |
| SEC-07 | Manual workflow execution | Workflow runs successfully |

---

# 14. Recommended CI/CD Security Flow

For a production DevSecOps implementation, the security workflow should be positioned before deployment.

```text
Source Code
    |
    v
Pull Request
    |
    v
Security Checks
    |
    +---- Gitleaks
    |
    +---- Trivy
    |
    +---- Semgrep
    |
    v
Security Gate
    |
    +---- PASS ---> Build ---> Test ---> Deploy
    |
    +---- FAIL ---> Remediation
```

---

# 15. Best Practices

## Secret Management

- Never store passwords or API keys directly in source code.
- Use GitHub Secrets for sensitive configuration.
- Rotate credentials immediately if a real secret is exposed.
- Do not print secrets in workflow logs.

## Dependency Security

- Keep application dependencies updated.
- Maintain dependency lock files where appropriate.
- Prioritize HIGH and CRITICAL vulnerabilities.
- Upgrade vulnerable packages to supported fixed versions.
- Re-run scanning after dependency remediation.

## Static Analysis

- Run static analysis during pull requests.
- Review security findings before merging.
- Maintain appropriate project-specific rules.
- Avoid disabling security rules without documented justification.

## GitHub Actions Security

- Use minimum required workflow permissions.
- Review third-party actions before adoption.
- Use approved/pinned action versions.
- Avoid unnecessary secrets.
- Keep workflows under source control and review changes through pull requests.

---

# 16. Troubleshooting

## Gitleaks Failure

If Gitleaks reports a finding:

1. Review the affected file and commit.
2. Determine whether the value is a real credential.
3. Revoke or rotate the credential if exposed.
4. Remove the secret from source control.
5. Re-run the workflow.

Do not treat credential exposure as a normal code-quality warning.

---

## Trivy Failure

If Trivy reports a vulnerability:

1. Identify the affected dependency.
2. Check the installed version.
3. Review the vulnerability identifier and severity.
4. Identify the available fixed version.
5. Upgrade the dependency.
6. Run application tests.
7. Re-run the security workflow.

---

## Semgrep Finding

If Semgrep reports an issue:

1. Review the finding and affected source-code location.
2. Understand the security rule that triggered.
3. Correct the vulnerable code pattern.
4. Run tests.
5. Re-run the workflow.

---

# 17. Future Improvements

The current implementation provides a baseline for CI/CD security checks.

Potential future improvements include:

- Container image scanning
- SARIF security reports
- GitHub Security integration
- Dependency update automation
- License compliance scanning
- Code coverage validation
- Build and deployment stages
- Branch protection requiring successful security checks
- Centralized security reporting
- Security notification/alerting

---

# 18. Implementation Checklist

### Workflow

- [x] GitHub Actions workflow created
- [x] Workflow stored under `.github/workflows`
- [x] Push trigger configured
- [x] Pull-request trigger configured
- [x] Manual trigger configured

### Security Controls

- [x] Gitleaks integrated
- [x] Trivy dependency scanning integrated
- [x] Semgrep static analysis integrated
- [x] HIGH/CRITICAL dependency policy configured
- [x] Workflow permissions explicitly defined

### Validation

- [x] Workflow executed successfully
- [x] Gitleaks passed
- [x] Trivy passed
- [x] Semgrep passed
- [x] Execution results verified

### Future

- [ ] Container image scanning
- [ ] SARIF reporting
- [ ] Branch protection/security gate
- [ ] Centralized security reporting

---

# 19. Jira Acceptance Criteria

The DEV-383 task is considered complete when:

- [x] A sample GitHub Actions security workflow has been created.
- [x] Multiple security checks are integrated into the workflow.
- [x] Secret scanning is implemented using Gitleaks.
- [x] Dependency vulnerability scanning is implemented using Trivy.
- [x] Static code analysis is implemented using Semgrep.
- [x] CI/CD triggers are configured.
- [x] Security permissions are defined.
- [x] Security gate behavior is documented.
- [x] Workflow execution has been successfully validated.
- [x] Technical documentation is available in the repository.

---

# 20. Conclusion

The DEV-383 implementation demonstrates a practical multi-layer security approach for GitHub Actions.

The workflow combines:

**Gitleaks**  
→ Secret detection

**Trivy**  
→ Dependency vulnerability detection

**Semgrep**  
→ Static source-code analysis

Together, these controls provide early security feedback and establish a foundation for integrating additional DevSecOps controls into the CI/CD lifecycle.

The workflow has been successfully executed, with all three implemented security checks completing successfully.

---

## Repository Structure

```text
devops-intern-Security-check/
│
├── .github/
│   └── workflows/
│       └── security-checks.yml
│
└── SECURITY_CHECKS_DOCUMENTATION.md
```

**Workflow:** `.github/workflows/security-checks.yml`  
**Documentation:** `SECURITY_CHECKS_DOCUMENTATION.md`
