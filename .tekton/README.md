# Konflux Platform Infrastructure Tests

This directory contains the Tekton pipeline for running platform infrastructure
tests via Konflux integration test scenarios.

## Overview

The `platform-infra-tests-pipeline.yaml` is triggered automatically by a Konflux
CronJob that runs daily at 6:00 AM UTC (configured in konflux-release-data
repository).

## Pipeline Features

### Multi-Command Support

The pipeline supports running multiple test types in a single execution:

- **Playwright** - Browser automation tests
- **Cypress** - End-to-end tests
- **Jest/Unit** - Unit test suite
- **Custom** - Any arbitrary npm script or command

### Configuration

The pipeline accepts a `TEST_COMMANDS` parameter that specifies which tests to
run. Multiple commands can be specified as comma-separated values.

#### Examples

**Single test type (Playwright only - default):**

```yaml
params:
  - name: TEST_COMMANDS
    value: "playwright"
```

**Multiple test types:**

```yaml
params:
  - name: TEST_COMMANDS
    value: "playwright,cypress"
```

**Custom npm script:**

```yaml
params:
  - name: TEST_COMMANDS
    value: "custom:npm run my-custom-test"
```

**Combination:**

```yaml
params:
  - name: TEST_COMMANDS
    value: "playwright,jest,custom:npm run test:integration"
```

## Supported Test Commands

### Built-in Commands

| Command | Description | npm Script |
|---------|-------------|------------|
| `playwright` | Run Playwright browser tests | `npm run playwright` |
| `cypress` | Run Cypress E2E tests | `npm run cypress:run` |
| `jest` or `unit` | Run Jest unit tests | `npm run test` |

### Custom Commands

Use the `custom:` prefix to run any arbitrary command:

```yaml
TEST_COMMANDS: "custom:npm run test:specific-suite"
TEST_COMMANDS: "custom:./scripts/run-integration-tests.sh"
```

## Modifying Test Configuration

### In konflux-release-data Repository

To change which tests run in the scheduled CronJob, update the
IntegrationTestScenario:

**File:**
`tenants-config/cluster/stone-prd-rh01/tenants/hcc-fr-tenant/chrome-frontend-sc/chrome-platform-infra-tests.yaml`

Add the `TEST_COMMANDS` parameter:

```yaml
spec:
  params:
    - name: TEST_COMMANDS
      value: "playwright,cypress"
  resolverRef:
    params:
      - name: url
        value: https://github.com/RedHatInsights/insights-chrome
      - name: revision
        value: main
      - name: pathInRepo
        value: .tekton/platform-infra-tests-pipeline.yaml
    resolver: git
```

### In This Repository

The pipeline itself (`platform-infra-tests-pipeline.yaml`) can be updated to:

- Add new test command types in the case statement
- Modify the default value of `TEST_COMMANDS`
- Add additional installation steps (e.g., for new browser types)
- Change the Node.js version (update the image references)

## Adding New Test Types

To add support for a new test type:

1. **Add a new case** in the `run-tests` step:

```yaml
newtest)
  echo "Executing new test suite..." | tee -a "$OUTPUT_FILE"
  if npm run test:new; then
    echo "✓ New tests passed" | tee -a "$OUTPUT_FILE"
  else
    echo "✗ New tests failed" | tee -a "$OUTPUT_FILE"
    OVERALL_RESULT="failure"
  fi
  ;;
```

2. **Add installation steps** if needed (e.g., browsers, tools):

Add a new step before `run-tests`:

```yaml
- name: install-custom-tools
  image: registry.access.redhat.com/ubi9/nodejs-20:latest
  workingDir: $(workspaces.source.path)
  script: |
    #!/bin/bash
    set -euo pipefail

    if echo "$(params.TEST_COMMANDS)" | grep -q "newtest"; then
      echo "Installing tools for newtest..."
      # Installation commands here
    fi
```

3. **Update this README** with the new command

## Pipeline Architecture

```text
┌─────────────────────────────────────────────────┐
│ Konflux CronJob (Daily 6 AM UTC)                │
│ (in konflux-release-data repo)                  │
└────────────────┬────────────────────────────────┘
                 │
                 ▼
┌─────────────────────────────────────────────────┐
│ IntegrationTestScenario                         │
│ chrome-platform-infra-tests                     │
│ - Points to this pipeline                       │
│ - Passes TEST_COMMANDS parameter                │
└────────────────┬────────────────────────────────┘
                 │
                 ▼
┌─────────────────────────────────────────────────┐
│ Pipeline: platform-infra-tests-pipeline.yaml    │
│ (this file)                                      │
│                                                  │
│  1. Clone Repository                            │
│  2. Install Dependencies (npm ci)               │
│  3. Install Test Tools (Playwright, etc.)       │
│  4. Run Test Commands                           │
│  5. Report Results                              │
└─────────────────────────────────────────────────┘
```

## Manual Testing

You can trigger this pipeline manually for testing:

### Local Testing (via oc CLI)

```bash
# Create a PipelineRun
oc create -f - <<EOF
apiVersion: tekton.dev/v1
kind: PipelineRun
metadata:
  name: manual-platform-test-$(date +%s)
  namespace: hcc-platex-services-tenant
spec:
  pipelineRef:
    resolver: git
    params:
      - name: url
        value: https://github.com/RedHatInsights/insights-chrome
      - name: revision
        value: main
      - name: pathInRepo
        value: .tekton/platform-infra-tests-pipeline.yaml
  params:
    - name: SNAPSHOT
      value: "manual-test"
    - name: git-url
      value: "https://github.com/RedHatInsights/insights-chrome"
    - name: git-revision
      value: "main"
    - name: TEST_COMMANDS
      value: "playwright"
  workspaces:
    - name: source
      volumeClaimTemplate:
        spec:
          accessModes:
            - ReadWriteOnce
          resources:
            requests:
              storage: 1Gi
EOF

# Watch the logs
oc logs -f <pipelinerun-pod-name>
```

### Via Konflux UI

1. Navigate to the Konflux UI
2. Go to `chrome-frontend-sc` application
3. Find the `chrome-platform-infra-tests` IntegrationTestScenario
4. Trigger manually from the UI

## Troubleshooting

### Tests Failing

1. **Check the logs** in the PipelineRun:
   ```bash
   oc get pipelinerun -n hcc-platex-services-tenant
   oc logs <pipelinerun-pod> -n hcc-platex-services-tenant
   ```

2. **Common issues:**
   - Playwright browser installation failed: Check if browsers are compatible
     with the UBI9 base image
   - npm ci failed: Check if package-lock.json is up to date
   - Test timeout: Increase the pipeline timeout if needed

### Pipeline Not Triggering

1. **Check CronJob status:**
   ```bash
   oc get cronjob cronjob-chrome-platform-infra-tests \
     -n hcc-platex-services-tenant
   ```

2. **Check if there are new commits:**
   - The CronJob only runs if there are commits in the last 24 hours
   - To run regardless of commits, remove `--no-old-commits-run` flag from
     the CronJob

3. **Verify IntegrationTestScenario exists:**
   ```bash
   oc get integrationtestscenario chrome-platform-infra-tests \
     -n hcc-platex-services-tenant
   ```

### Modifying the Schedule

The schedule is configured in the konflux-release-data repository:

**File:**
`tenants-config/cluster/stone-prd-rh01/tenants/hcc-platex-services-tenant/periodics/cronjob-chrome-platform-infra-tests.yaml`

Change the `schedule` field (cron format):

```yaml
spec:
  schedule: '0 6 * * *'  # Daily at 6 AM UTC
```

## References

- [Konflux Integration Tests](https://konflux-ci.dev/docs/how-tos/testing/integration/)
- [Tekton Pipelines](https://tekton.dev/docs/pipelines/)
- [konflux-release-data MR](https://gitlab.cee.redhat.com/releng/konflux-release-data/-/merge_requests/20736)
