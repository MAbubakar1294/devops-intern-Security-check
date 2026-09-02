# CI/CD Security Checks Integration Guide

**Document Type:** Technical Integration Guide  
**CI Platform:** GitHub Actions  
**Security Scope:** Secrets, SAST, Dependencies, Container Images, SBOM  
**Status:** Reference Implementation  

---

## Document Purpose

This guide defines a structured approach for integrating security checks into an existing CI/CD pipeline using GitHub Actions.

The implementation follows a layered security model covering:

- Secrets scanning with **GitLeaks**
- Static Application Security Testing (SAST) with **Semgrep**
- Dependency vulnerability scanning with **OSV-Scanner**
- Container image scanning with **Trivy**
- SBOM generation with **Syft**
- SBOM-based vulnerability scanning with **Grype**

Each control is positioned as a security gate so that identified security issues can be detected and addressed before deployment.

---

# 1. Introduction to CI/CD

CI/CD (Continuous Integration and Continuous Delivery/Deployment) automates the process of building, validating, testing, and delivering software.

### Continuous Integration (CI)

CI ensures that code changes are automatically validated when developers push changes or create pull requests.

### Continuous Delivery/Deployment (CD)

CD ensures that validated changes can be safely delivered or deployed to target environments.

Although CI/CD improves development speed and consistency, it can also accelerate the propagation of vulnerabilities if security controls are absent. Integrating security checks directly into the pipeline supports **shift-left security**, allowing issues to be identified before they reach production.

> **Security Principle:** Prevention in CI is always cheaper and safer than remediation in production.

---

# 2. Why Security in CI/CD Is Critical

Modern applications depend heavily on:

- Open-source dependencies
- Containerized runtimes
- Cloud-native infrastructure
- Rapid development cycles

These introduce risks including:

| Risk | Example |
|---|---|
| Secret exposure | API keys, tokens, cloud credentials |
| Vulnerable dependencies | Known CVEs in direct/transitive packages |
| Insecure code | Injection or insecure cryptographic patterns |
| Container vulnerabilities | Vulnerable OS/application packages |
| Supply-chain risk | Unknown or vulnerable software components |

Integrating security controls into CI/CD provides earlier visibility and automated enforcement.

---

# 3. Security Architecture

The reference security pipeline follows a defense-in-depth model:

```text
                         SOURCE CODE
                              │
                              ▼
                    ┌──────────────────┐
                    │  GitLeaks        │
                    │ Secrets Scanning │
                    └────────┬─────────┘
                             │
                             ▼
                    ┌──────────────────┐
                    │    Semgrep       │
                    │       SAST       │
                    └────────┬─────────┘
                             │
                             ▼
                    ┌──────────────────┐
                    │  OSV-Scanner     │
                    │ Dependency Scan  │
                    └────────┬─────────┘
                             │
                             ▼
                    ┌──────────────────┐
                    │   Docker Build   │
                    │  Image Creation  │
                    └────────┬─────────┘
                             │
                             ▼
                    ┌──────────────────┐
                    │     Trivy        │
                    │  Image Scanning  │
                    └────────┬─────────┘
                             │
                             ▼
                    ┌──────────────────┐
                    │      Syft        │
                    │   SBOM Generate  │
                    └────────┬─────────┘
                             │
                             ▼
                    ┌──────────────────┐
                    │     Grype        │
                    │   SBOM Scanning  │
                    └────────┬─────────┘
                             │
                             ▼
                         DEPLOYMENT
```

The pipeline should progress toward deployment only after the required security gates pass.

---

# 4. Security Tooling and Responsibilities

| Security Area | Tool | Primary Responsibility |
|---|---|---|
| Secrets Scanning | GitLeaks | Detect hard-coded credentials |
| SAST | Semgrep | Detect insecure source-code patterns |
| Dependency Scanning | OSV-Scanner | Identify known dependency vulnerabilities |
| Image Scanning | Trivy | Detect OS/application vulnerabilities in images |
| SBOM Generation | Syft | Generate software component inventory |
| SBOM Scanning | Grype | Scan SBOM components for vulnerabilities |

The tools in this reference implementation are selected for their CI suitability, open-source availability, industry adoption, and community support.

---

# 5. Prerequisites

Before integrating the controls into an existing GitHub Actions pipeline, verify the following:

### Repository Requirements

- GitHub repository is available.
- GitHub Actions is enabled.
- Workflow files are stored under `.github/workflows/`.
- Required source and dependency manifests are present.
- A Dockerfile exists when container scanning is required.

### GitHub Actions Requirements

The workflow should have appropriate permissions:

```yaml
permissions:
  contents: read
  security-events: write
```

For Git-history-based secrets scanning, use:

```yaml
- name: Checkout repository
  uses: actions/checkout@v4
  with:
    fetch-depth: 0
```

### Credential Management

Do not hard-code credentials in workflow files or source code. Use GitHub Secrets for credentials required by external services.

---

# 6. Phase 1 — Secrets Scanning with GitLeaks

## 6.1 Objective

Secrets scanning identifies credentials accidentally committed to source code or Git history.

Typical targets include:

- API keys
- Access tokens
- Passwords
- Cloud credentials

## 6.2 Security Impact

Exposed secrets may result in:

- Account takeover
- Data breaches
- Unauthorized cloud-resource usage

## 6.3 Integration

```yaml
- name: Secrets Scan (GitLeaks)
  uses: gitleaks/gitleaks-action@v2
  env:
    GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
```

## 6.4 Recommended Placement

GitLeaks should execute early in the pipeline because secret exposure is a high-impact issue and should be identified before later build stages.

## 6.5 Recommended Controls

- Scan source code and Git history.
- Block pull requests when secrets are detected.
- Remove exposed credentials immediately.
- Rotate/revoke compromised credentials.
- Investigate historical exposure where required.

---

# 7. Phase 2 — Static Application Security Testing with Semgrep

## 7.1 Objective

SAST analyzes application source code without executing the application and identifies potentially insecure coding patterns.

Examples include:

- Command injection
- SQL injection
- Cross-site scripting (XSS)
- Insecure cryptography
- Hard-coded credentials

## 7.2 Integration

```yaml
- name: Run SAST (Semgrep)
  uses: returntocorp/semgrep-action@v1
  with:
    config: p/security-audit
```

## 7.3 Recommended Controls

- Run SAST during pull requests.
- Use severity-based gating.
- Tune rules to reduce false positives.
- Review security findings before deployment.

---

# 8. Phase 3 — Dependency Vulnerability Scanning with OSV-Scanner

## 8.1 Objective

Dependency scanning identifies vulnerabilities in direct and transitive dependencies.

The scanner compares dependency information against public vulnerability databases.

## 8.2 Integration

```yaml
- name: Run OSV-Scanner
  uses: google/osv-scanner-action/osv-scanner-action@ffff457756fc02fd3b933aabf3705406f57a2e19
  with:
    scan-args: |-
      --recursive
      ./
```

## 8.3 Recommended Controls

- Scan dependencies before building deployment artifacts.
- Fail builds on high-severity vulnerabilities according to policy.
- Keep dependencies regularly updated.
- Review transitive dependencies.

---

# 9. Phase 4 — Container Image Scanning with Trivy

## 9.1 Objective

Container image scanning analyzes images for:

- OS-level vulnerabilities
- Application-library vulnerabilities
- Vulnerable base-image components

## 9.2 Build and Scan

```yaml
- name: Set up Docker
  uses: docker/setup-buildx-action@v3

- name: Build Docker image
  run: |
    docker build -t $IMAGE_NAME:$IMAGE_TAG .

- name: Image Scan (Trivy)
  uses: aquasecurity/trivy-action@0.22.0
  with:
    image-ref: ${{ env.IMAGE_NAME }}:${{ env.IMAGE_TAG }}
    severity: HIGH,CRITICAL
    vuln-type: os,library
    ignore-unfixed: true
    exit-code: "1"
```

## 9.3 Security Gate

The reference configuration treats **HIGH** and **CRITICAL** findings as pipeline-blocking conditions.

## 9.4 Recommended Controls

- Prefer minimal and updated base images.
- Enforce HIGH/CRITICAL severity gates.
- Review unfixed vulnerabilities.
- Document justified exceptions.

---

# 10. Phase 5 — SBOM Generation with Syft

## 10.1 Objective

A Software Bill of Materials (SBOM) provides an inventory of software components contained in an application or container.

It can include:

- Packages
- Libraries
- Versions
- Dependencies

## 10.2 Benefits

SBOMs improve:

- Supply-chain visibility
- Compliance and auditing
- Vulnerability analysis
- Component traceability

## 10.3 Integration

```yaml
- name: Install Syft
  run: |
    curl -sSfL https://raw.githubusercontent.com/anchore/syft/main/install.sh | sh -s -- -b /usr/local/bin

- name: Generate SBOM
  run: |
    syft $IMAGE_NAME:$IMAGE_TAG -o json > sbom.json
```

---

# 11. Phase 6 — SBOM Vulnerability Scanning with Grype

## 11.1 Objective

Grype scans the generated SBOM for known vulnerabilities.

## 11.2 Integration

```yaml
- name: Install Grype
  run: |
    curl -sSfL https://raw.githubusercontent.com/anchore/grype/main/install.sh | sh -s -- -b /usr/local/bin

- name: Vulnerability Scan (Grype)
  run: |
    grype sbom:sbom.json --fail-on high --only-fixed
```

## 11.3 Security Gate

The example fails when applicable high-severity vulnerabilities are identified.

---

# 12. Production-Grade GitHub Actions Reference Workflow

The following reference workflow combines all security phases:

```yaml
name: CI Security Pipeline

on:
  push:
    branches: ["main"]
  pull_request:
    branches: ["main"]

permissions:
  contents: read
  security-events: write

env:
  IMAGE_NAME: myapp
  IMAGE_TAG: ${{ github.sha }}

jobs:
  security-checks:
    runs-on: ubuntu-latest

    steps:
      - name: Checkout repository
        uses: actions/checkout@v4
        with:
          fetch-depth: 0

      - name: Secrets Scan (GitLeaks)
        uses: gitleaks/gitleaks-action@v2
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}

      - name: Run SAST (Semgrep)
        uses: returntocorp/semgrep-action@v1
        with:
          config: p/security-audit

      - name: Run OSV-Scanner
        uses: google/osv-scanner-action/osv-scanner-action@ffff457756fc02fd3b933aabf3705406f57a2e19
        with:
          scan-args: |-
            --recursive
            ./

      - name: Set up Docker
        uses: docker/setup-buildx-action@v3

      - name: Build Docker image
        run: |
          docker build -t $IMAGE_NAME:$IMAGE_TAG .

      - name: Image Scan (Trivy)
        uses: aquasecurity/trivy-action@0.22.0
        with:
          image-ref: ${{ env.IMAGE_NAME }}:${{ env.IMAGE_TAG }}
          severity: HIGH,CRITICAL
          vuln-type: os,library
          ignore-unfixed: true
          exit-code: "1"

      - name: Install Syft
        run: |
          curl -sSfL https://raw.githubusercontent.com/anchore/syft/main/install.sh | sh -s -- -b /usr/local/bin

      - name: Generate SBOM
        run: |
          syft $IMAGE_NAME:$IMAGE_TAG -o json > sbom.json

      - name: Install Grype
        run: |
          curl -sSfL https://raw.githubusercontent.com/anchore/grype/main/install.sh | sh -s -- -b /usr/local/bin

      - name: Vulnerability Scan (Grype)
        run: |
          grype sbom:sbom.json --fail-on high --only-fixed
```

---

# 13. Integrating into an Existing CI/CD Pipeline

When adding security checks to an existing pipeline, avoid creating a completely separate delivery process unless required.

A practical integration sequence is:

### Step 1 — Identify the Existing Pipeline

Review:

- Trigger events
- Build stages
- Test stages
- Artifact creation
- Container build process
- Deployment stages

### Step 2 — Add Early Security Controls

Add GitLeaks and Semgrep after source checkout.

### Step 3 — Scan Dependencies

Run OSV-Scanner before creating deployment artifacts.

### Step 4 — Build the Container

Build the image using the existing Docker build process.

### Step 5 — Scan the Image

Run Trivy against the exact image that will proceed toward deployment.

### Step 6 — Generate the SBOM

Generate `sbom.json` using Syft.

### Step 7 — Scan the SBOM

Use Grype to identify known vulnerabilities in the generated component inventory.

### Step 8 — Enforce Gates

Configure the pipeline so required security failures stop progression toward deployment.

---

# 14. Security Gate Policy

Security tools should provide both visibility and enforcement.

| Control | Gate Behavior |
|---|---|
| GitLeaks | Fail when secrets are detected |
| Semgrep | Apply severity-based gating |
| OSV-Scanner | Fail on high-severity vulnerabilities according to policy |
| Trivy | Fail on HIGH/CRITICAL vulnerabilities |
| Grype | Fail on high-severity findings according to policy |

Exceptions should be explicitly reviewed, documented, and justified.

---

# 15. Troubleshooting

## GitLeaks — Secret Detected

**Symptom:** Workflow fails during secrets scanning.

**Resolution:**
1. Remove the secret from source code.
2. Revoke or rotate the exposed credential.
3. Review Git history for exposure.
4. Re-run the pipeline after remediation.

## Semgrep — Security Finding

**Symptom:** SAST identifies an insecure coding pattern.

**Resolution:**
1. Review the affected code.
2. Confirm whether the finding is valid.
3. Remediate the insecure pattern.
4. Tune the rule only when a finding is confirmed as a false positive.

## OSV-Scanner — Vulnerable Dependency

**Symptom:** Dependency scanning reports a vulnerability.

**Resolution:**
1. Identify the affected package and version.
2. Upgrade to a safe version where available.
3. Rebuild the dependency tree.
4. Re-run the scan.

## Trivy — Image Vulnerability

**Symptom:** HIGH/CRITICAL vulnerability blocks the image stage.

**Resolution:**
1. Identify the affected OS package or application library.
2. Update the dependency or base image.
3. Rebuild the image.
4. Run Trivy again.

## Grype — SBOM Finding

**Symptom:** Grype blocks the pipeline because of a vulnerability in the SBOM.

**Resolution:**
1. Identify the vulnerable component.
2. Update or replace the affected component.
3. Regenerate the SBOM.
4. Re-run the Grype scan.

---

# 16. Validation — Break and Fix Approach

A security pipeline should be validated using both successful and intentionally failing scenarios.

### Test Scenarios

Controlled test cases can introduce:

- Hard-coded secrets
- Vulnerable dependencies
- Insecure code patterns
- Vulnerable container images

### Expected Behavior

For each applicable test:

1. The corresponding tool detects the issue.
2. The security gate fails.
3. The pipeline is prevented from progressing.
4. The issue is remediated.
5. The pipeline passes after remediation.

This demonstrates that the controls are actively enforced rather than simply configured.

---

# 17. Operational Best Practices

### Secrets

- Never commit credentials.
- Use managed secrets.
- Rotate exposed credentials immediately.

### Source Code

- Run SAST on pull requests.
- Keep security rules maintained.
- Review high-severity findings before merging.

### Dependencies

- Keep packages updated.
- Scan both direct and transitive dependencies.
- Establish severity thresholds.

### Containers

- Use minimal, maintained base images.
- Scan the exact image intended for deployment.
- Review HIGH/CRITICAL findings before release.

### SBOM

- Generate SBOMs during the build process.
- Preserve SBOM artifacts when required for traceability.
- Re-scan SBOMs when vulnerability intelligence changes.

---

# 18. Implementation Checklist

- [ ] GitHub Actions enabled
- [ ] Security workflow stored under `.github/workflows/`
- [ ] GitLeaks integrated
- [ ] Full Git history enabled where required
- [ ] Semgrep integrated
- [ ] OSV-Scanner integrated
- [ ] Docker image build integrated
- [ ] Trivy image scanning integrated
- [ ] HIGH/CRITICAL image gate configured
- [ ] Syft SBOM generation integrated
- [ ] Grype SBOM scanning integrated
- [ ] GitHub Actions permissions reviewed
- [ ] Security failure conditions tested
- [ ] Break-and-fix validation completed
- [ ] Troubleshooting procedures documented

---

# 19. Reference Security Flow

```text
Developer Commit / Pull Request
              │
              ▼
       GitHub Actions CI
              │
      ┌───────┴────────┐
      ▼                ▼
   GitLeaks          Semgrep
   Secrets             SAST
      │                │
      └───────┬────────┘
              ▼
         OSV-Scanner
       Dependencies
              │
              ▼
        Docker Build
              │
              ▼
           Trivy
        Image Scan
              │
              ▼
            Syft
         Generate SBOM
              │
              ▼
           Grype
         SBOM Scan
              │
              ▼
      Security Gates Pass
              │
              ▼
          Deployment
```

---

# 20. Conclusion

Integrating layered security controls into CI/CD provides automated protection across source code, credentials, dependencies, container images, and software supply-chain components.

The GitHub Actions reference implementation establishes security gates using GitLeaks, Semgrep, OSV-Scanner, Trivy, Syft, and Grype. The controls are positioned throughout the pipeline so security findings can be detected and remediated before deployment.

The result is a structured **shift-left security approach** that combines early detection, automated enforcement, and supply-chain visibility.

---

**Document Status:** Ready for CI/CD Integration Reference  
**Primary Platform:** GitHub Actions  
**Security Model:** Layered / Defense-in-Depth
