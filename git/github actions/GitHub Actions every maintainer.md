# GitHub Actions every maintainer

## 1. stale

- [actions/stale](https://github.com/actions/stale)

## 2. labeler

- [actions/labeler](https://github.com/actions/labeler)

## 3. pull request template

- 템플릿은 다음 위치 중 한 곳에 넣으면 인식합니다.
  - 루트 디렉토리 : pull_request_template.md
  - docs 디렉토리 : docs/pull_request_template.md
  - .github 디렉토리 : .github/pull_request_template.md

## 4. super linter

- [super-linter/super-linter](https://github.com/super-linter/super-linter)

## 5. create or update comment

- [peter-evans/create-or-update-comment](https://github.com/peter-evans/create-or-update-comment)

## 6. release drafter

- [release-drafter/release-drafter](https://github.com/release-drafter/release-drafter)

## 7. find secrets

- [trufflesecurity/trufflehog](https://github.com/trufflesecurity/trufflehog)

## 8. codeql action

- [github/codeql-action](https://github.com/github/codeql-action)

```yaml
name: "Golang Security Scan"

# Run workflow each time code is pushed to your repository and on a schedule.
# The scheduled workflow runs every at 00:00 on Sunday UTC time.
on:
  push:
  schedule:
  - cron: '0 0 * * 0'

jobs:
  tests:
    runs-on: ubuntu-latest
    env:
      GO111MODULE: on
    steps:
      - name: Checkout Source
        uses: actions/checkout@v5
        if: ${{ github.actor != 'dependabot[bot]' }}
      - name: Run Gosec Security Scanner
        if: ${{ github.actor != 'dependabot[bot]' }}
        uses: securego/gosec@v2.22.9
        with:
          # we let the report trigger content trigger a failure using the GitHub Security features.
          args: '-no-fail -fmt sarif -out results.sarif ./...'
      - name: Upload SARIF file
        if: ${{ github.actor != 'dependabot[bot]' }}
        uses: github/codeql-action/upload-sarif@v3
        with:
          # Path to SARIF file relative to the root of the repository
          sarif_file: results.sarif
```

```yaml
name: "Java/Kotlin Security Scan"

on:
  push:
  pull_request:
  schedule:
    - cron: '0 0 * * 0' # 매주 일요일 00:00 UTC

jobs:
  codeql:
    name: Analyze
    runs-on: ubuntu-latest
    permissions:
      actions: read
      contents: read
      security-events: write
    steps:
      - name: Checkout repository
        uses: actions/checkout@v4
      - name: Initialize CodeQL
        uses: github/codeql-action/init@v3
        with:
          languages: java # Kotlin도 자동으로 Java 분석기에 포함됩니다
      - name: Autobuild
        uses: github/codeql-action/autobuild@v3
      - name: Perform CodeQL Analysis
        uses: github/codeql-action/analyze@v3
```


## 9. pluto

> Detect Deprecated API

- [FairwindsOps/pluto](https://github.com/FairwindsOps/pluto)

## 10. trivy action

- [aquasecurity/trivy-action](https://github.com/aquasecurity/trivy-action)

```yaml
name: Trivy Nightly Scan
on:
  schedule:
    - cron: "0 2 * * 5" # Run at 2AM UTC on every Friday

permissions: read-all
jobs:
  nightly-scan:
    name: Trivy Scan nightly
    strategy:
      fail-fast: false
      matrix:
        # It will test for only the latest version as older version is not maintained
        versions: [latest]
    permissions:
      security-events: write # for github/codeql-action/upload-sarif to upload SARIF results

    runs-on: ubuntu-latest
    steps:
      - name: Checkout code
        uses: actions/checkout@11bd71901bbe5b1630ceea73d27597364c9af683 # v4.2.2
      - name: Run Trivy vulnerability scanner
        uses: aquasecurity/trivy-action@6c175e9c4083a92bbca2f9724c8a5e33bc2d97a5 # master
        with:
          image-ref: "docker.io/coredns/coredns:${{ matrix.versions }}"
          severity: "CRITICAL,HIGH"
          format: "sarif"
          output: "trivy-results.sarif"

      - name: Upload Trivy scan results to GitHub Security tab
        uses: github/codeql-action/upload-sarif@ff0a06e83cb2de871e5a09832bc6a81e7276941f # v3.28.18
        with:
          sarif_file: "trivy-results.sarif"
```

참고 문서

- [5 GitHub Actions every maintainer needs to know](https://github.blog/open-source/maintainers/5-github-actions-every-maintainer-needs-to-know/)
