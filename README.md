# follia-ci-security

Reusable GitHub Actions workflow that runs security checks on Follia microservices. It is
**report-only**: findings never fail a job, they are listed in the run summary.

| Tool | What it checks |
|---|---|
| [Semgrep](https://semgrep.dev) CE | the code, with the public rulesets of each language, OWASP Top 10, the shared rules in `rules/` and the service's own rules in `.semgrep/` |
| [Trivy](https://trivy.dev) | known CVEs in the libraries (`pom.xml`, `requirements.txt`, `package-lock.json`, ...) and in the base image of the Dockerfile |
| [gitleaks](https://github.com/gitleaks/gitleaks) | secrets committed to git, always redacted in the output |

On a pull request Semgrep and gitleaks report only what the PR introduces (Semgrep also
re-reports existing lines the PR modifies). On any other event (weekly schedule, manual run)
the whole repository is scanned, so new CVEs in unchanged code show up too.

## Usage

Copy [`examples/on-pr_security-scan.yml`](examples/on-pr_security-scan.yml) to
`.github/workflows/on-pr_security-scan.yml` in the service repo and set the inputs:

| Input | Default | Description |
|---|---|---|
| `languages` | (required) | comma-separated: `java`, `python`, `typescript` (`javascript` is an alias) |
| `dockerfile` | `.build/dockerfiles/Dockerfile` | Dockerfile used to find the base image |
| `target-stage` | `dist` | stage that runs in production; if no stage has this name, the last `FROM` is used. A stage built `FROM` another stage is followed to the real image |
| `rules-ref` | `v1` | ref of this repo the shared rules are taken from; keep it equal to the ref after `@` in `uses:` |

## Python services with a generated `requirements.txt`

Where `requirements.txt` is generated from `requirements.base.txt` and not committed, the
workflow scans `requirements.base.txt` instead. Trivy can only match CVEs against **pinned**
versions (`package==1.2.3`): unpinned dependencies are not checked.

## Exceptions

- Semgrep: `// nosemgrep: <rule-id>` (or `# nosemgrep: <rule-id>`) on the line, with a comment
  explaining why.
- gitleaks: the finding's fingerprint in `.gitleaksignore`.
- Trivy: the CVE id in `.trivyignore`, with a comment explaining why.

## Shared rules

`rules/<language>/*.yml` are Semgrep rules for patterns specific to our code, applied to every
service of that language. Rules useful to a single service belong in that service's `.semgrep/`.

## Versions

Callers use the moving major tag `v1`. Changes that keep the inputs compatible are released
by tagging `v1.x.y` and moving `v1`; a breaking change to the inputs gets `v2`.
