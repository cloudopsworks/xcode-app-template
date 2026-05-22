# Xcode App Template

This repository is the **CloudOps Works Xcode application template** for bootstrapping a new iOS application with an Xcode project skeleton, GitHub Actions workflows, and CloudOps Works CI/CD delivery wiring already in place.

Use this template when you want a repository that already includes:

- an Xcode project skeleton (rename from the example `I am Groot` app)
- CloudOps Works CI/CD configuration under `.cloudopsworks/`
- GitHub Actions workflows for PR builds, main builds, device testing, and cleanup
- deployment inputs for AWS Device Farm and GCP Firebase Test Lab
- device test scripts under `device_tests/` and a `UITests` target
- Dual GitVersion presets (GitFlow and GitHub Flow)
- optional security scanning via Snyk, Semgrep, and SonarQube

---

## What gets generated from this template

### Application scaffold
- `I am Groot/` — main application source (rename to your app name)
- `I am Groot.xcodeproj/` — Xcode project file (rename)
- `I am GrootUITests/` — UI test target (rename)
- `device_tests/` — device test scripts
- `Makefile` — version helper targets powered by Tronador

### Delivery scaffold
- `.cloudopsworks/cloudopsworks-ci.yaml` — repository governance and deployment routing
- `.cloudopsworks/vars/inputs-global.yaml` — global Xcode config, cloud target, Apple Developer settings
- `.cloudopsworks/vars/inputs-XCODE-ENV.yaml` — per-environment Xcode build and device configuration
- `.cloudopsworks/vars/preview/` — preview environment configuration fragments
- `.cloudopsworks/_VERSION` — template version tracked by release automation
- `.cloudopsworks/gitversion_gitflow.yaml` — GitFlow reference configuration
- `.cloudopsworks/gitversion_githubflow.yaml` — GitHub Flow reference configuration
- `.github/workflows/` — reusable CI/CD orchestration

---

## Recommended bootstrap flow

### 1. Create a repository from this template
Create a new repository from `cloudopsworks/xcode-app-template`, then clone it locally.

### 2. Rename the Xcode project
Rename the example project and all related targets to match your application name:

- Rename `I am Groot.xcodeproj` → `<YourApp>.xcodeproj`
- Rename the main app folder and UITests folder to match
- Update `CFBundleIdentifier` in project settings
- Update the Apple Developer Team ID and product bundle in `.cloudopsworks/vars/inputs-global.yaml`

### 3. Bootstrap the version metadata
```bash
make version
```
Writes the computed version to `VERSION` using GitVersion semantics.

### 4. Set the base application metadata
Update `.cloudopsworks/vars/inputs-global.yaml`:
- `organization_name`, `organization_unit`, `environment_name`, `repository_owner`
- `xcode.version` — Xcode version to build with (default: `16.4`)
- `xcode.dev_team` — Apple Developer Team ID
- `xcode.product_bundle` — product bundle identifier matching the Xcode project
- `xcode.schemes` — list of Xcode schemes to build
- `xcode.sdks` — target SDKs (e.g., `iphoneos`)
- `xcode.destinations` — build destinations (e.g., `generic/platform=iOS`)
- `xcode.export_options` — code signing and distribution export options
- `cloud` — `aws`, `azure`, or `gcp`
- `cloud_type` — `device-farm`

### 5. Choose a branching model
```bash
# For GitFlow (develop, feature/*, release/*, hotfix/*)
cp .cloudopsworks/gitversion_gitflow.yaml .cloudopsworks/gitversion.yaml

# For GitHub Flow (main/master, short-lived feature branches)
cp .cloudopsworks/gitversion_githubflow.yaml .cloudopsworks/gitversion.yaml
```

### 6. Configure per-environment inputs
Fill in `.cloudopsworks/vars/inputs-XCODE-ENV.yaml` for each environment:
- `environment` — target environment label (`dev`, `uat`, `prod`, `demo`)
- `xcode.configuration` — `Debug` or `Release`
- Uncomment and fill the `aws`, `gcp`, or `azure` block for the device testing target

---

## Deployment targets

### AWS Device Farm
Use the `aws` block in `inputs-XCODE-ENV.yaml`.

Key fields:
- `aws.region` — must be `us-west-2`
- `aws.device_farm_name` — Device Farm project name
- `aws.device_farm_pool` — device pool (or specify `aws.device` individually)
- `aws.test_script_path` — path to test artifacts (e.g., `device_tests`)
- `aws.test_script_type` — test type (e.g., `XCTEST_UI_TEST_PACKAGE`)

To discover available iOS devices:
```bash
aws devicefarm list-devices --region us-west-2 \
  --filters attribute=PLATFORM,operator=EQUALS,values=IOS \
  --query 'devices[].[model,formFactor,os]' --output table
```

### GCP Firebase Test Lab
Use the `gcp` block in `inputs-XCODE-ENV.yaml`.

Key fields:
- `gcp.project_id` — GCP project ID
- `gcp.region` — GCP region
- `gcp.devices` — list of `model_id`, `version_id`, `locale_id`, `orientation`
- `gcp.test_lab_bucket` — GCS bucket for test results
- `gcp.test_script_type` — `robo`, `xctest`, or `game-loop`

To discover available iOS device models:
```bash
gcloud firebase test ios models list
```

### Azure
Use the `azure` block in `inputs-XCODE-ENV.yaml` for Azure Key Vault secret injection alongside the build.

Key fields:
- `azure.resource_group` — Azure resource group
- `azure.keyvault_name` — Azure Key Vault name
- `azure.keyvault_secret_filter` — secret name prefix to import

---

## GitHub Actions workflow model

Key workflows in this template:

- `main-build.yml` — full Xcode build, IPA signing, artifact upload, GitVersion tagging
- `pr-build.yml` — PR validation: build, optional linting, security scan
- `deploy.yml` — dispatched per environment; builds IPA and runs device tests
- `scan.yml` — periodic security and dependency scanning
- `automerge.yml` — auto-merge approved dependency-update PRs
- `pr-close.yaml` — preview environment cleanup when a PR closes
- `jira-integration.yml` — Jira ticket transitions on release events
- `environment-destroy.yml` / `environment-unlock.yml` — environment lifecycle operations

---

## Minimum checklist before first release

- [ ] Xcode project, schemes, and targets are renamed from the example app
- [ ] Apple Developer Team ID and bundle identifier are set in `inputs-global.yaml`
- [ ] Code signing export options are configured for distribution
- [ ] Choose a GitVersion preset and copy it to `.cloudopsworks/gitversion.yaml`
- [ ] Replace all placeholders in `inputs-global.yaml` (`ORG_NAME`, `ORG_UNIT`, `ENV_NAME`, `REPO_OWNER`)
- [ ] Set real `cloud` and `cloud_type` values
- [ ] Fill in `inputs-XCODE-ENV.yaml` with real environment, device, and cloud values
- [ ] Remove `disable_deploy` or set it correctly per environment
- [ ] Configure GitHub environment secrets for each deployment environment
- [ ] Update `README.md` to describe the actual iOS application
- [ ] CI passes on a pull request

---

## Notes

- `.omx/`, `.claude/`, `.opencode/`, and similar agent/tooling directories are intentionally ignored and are not part of the application template contract.
- The template is designed for CloudOps Works blueprint-backed automation; if you remove that integration, also prune the related workflows and `.cloudopsworks/` configuration.
- `runner_set` defaults to `macos-latest`; override per environment in `inputs-XCODE-ENV.yaml` when using self-hosted macOS runners.

---

## AI-assisted upgrade of `.cloudopsworks/vars` configuration files

This section is a machine-readable protocol for AI agents performing a seamless, non-destructive upgrade of all configuration files under `.cloudopsworks/vars/` when a new template version is released. Follow the steps below in order.

### Upgrade overview

The template version locked into this repository is recorded in `.cloudopsworks/_VERSION`. The canonical upstream source is the GitHub repository `cloudopsworks/xcode-app-template`, pinned to the tag that matches the content of `_VERSION`.

An upgrade merges new keys, updated comments, and structural changes from the upstream template into local files **without overwriting values the operator has already set**.

---

### Step 1 — determine current and target versions

1. Read `.cloudopsworks/_VERSION` to get the **current locked version** (e.g., `v1.4.15`).
2. The **target version** is either supplied by the operator or is the latest release tag on `cloudopsworks/xcode-app-template`.
3. Fetch any upstream file from GitHub using the pattern:
   ```
   https://raw.githubusercontent.com/cloudopsworks/xcode-app-template/<version>/<path>
   ```
   Example:
   ```
   https://raw.githubusercontent.com/cloudopsworks/xcode-app-template/v1.4.15/.cloudopsworks/vars/inputs-global.yaml
   ```

---

### Step 2 — identify the deployment type for each environment file

Each `inputs-<name>.yaml` file under `.cloudopsworks/vars/` maps to a specific upstream template. Determine the type using the following priority order:

**Priority 1 — `Agents:` header comment**

If the file contains an `# Agents:` line in its header block, read `cloud` and `cloud_type` directly from it:

```yaml
# Agents: cloud=aws ; cloud_type=device-farm
```

**Priority 2 — fallback to `inputs-global.yaml`**

If no `# Agents:` line is present, read the active `cloud` and `cloud_type` values from `.cloudopsworks/vars/inputs-global.yaml` and apply the mapping table below.

**`cloud` / `cloud_type` → upstream template file:**

| `cloud`             | `cloud_type`      | Upstream template file      |
|---------------------|-------------------|-----------------------------|
| `aws`               | `device-farm`     | `inputs-XCODE-ENV.yaml`     |
| `gcp`               | `device-farm`     | `inputs-XCODE-ENV.yaml`     |
| `azure`             | `device-farm`     | `inputs-XCODE-ENV.yaml`     |

`inputs-global.yaml` always maps to the upstream `inputs-global.yaml` regardless of cloud type.

---

### Step 3 — merge environment-specific files

For each `inputs-<name>.yaml`, apply all of the following rules.

#### Keys and values

- **Preserve operator-set values** — any key whose local value differs from the upstream template's placeholder or default must be kept exactly as-is.
- **Add missing keys** — keys present in the upstream template but absent locally must be inserted at the correct structural position using the upstream default value and comment.
- **Flag removed keys** — keys present locally but deleted from the upstream template must be reported to the operator before removal; do not silently delete them.

#### Comments

- **Template comments are authoritative for unchanged sections** — section-level and field-level comments from the upstream template replace their local equivalents when the operator has made no additions to that comment block.
- **Preserve operator-added comments** — any comment not present in the upstream template must be retained verbatim.
- **Update the `Agents:` header line** — if the upstream template added or changed the `# Agents:` metadata line, update it in the local file without altering the first description line (`# This file contains...`).

#### Formatting

- **Match upstream indentation and quoting** — indentation, block vs. flow style, and quoted vs. unquoted strings must match the upstream template for any unchanged or newly added sections.
- **Commented-out blocks** — blocks that are commented out in the upstream template must remain commented out unless the operator has explicitly uncommented them locally.
- **Multiline scalars** — preserve the operator's choice of `|` vs. `>` for any multiline value the operator has set.

---

### Step 4 — merge `inputs-global.yaml`

`inputs-global.yaml` requires special handling because it contains mandatory operator identity fields alongside a large body of optional commented-out sections.

Merge procedure:

1. **Retain the four mandatory identity fields** verbatim at the top of the file:
   ```yaml
   organization_name: "..."
   organization_unit: "..."
   environment_name: "..."
   repository_owner: "..."
   ```
2. **Retain `cloud` and `cloud_type`** exactly as the operator set them.
3. **For every optional commented-out section** in the upstream template, check the local file:
   - If the operator **has uncommented and configured it** — keep the operator's values; update only surrounding comment text if it changed upstream.
   - If the section **is still fully commented out locally** — replace the entire commented block with the upstream version, capturing any new fields or updated documentation within it.
4. **Append new optional sections** that appear in the upstream template but are entirely absent locally, in fully commented-out form, preserving their upstream position and comments.

---

### Step 5 — upgrade subdirectory files

Apply the same merge rules (Steps 3 and 4) to every file in the following subdirectories, matching each local file to its corresponding upstream file at the same relative path:

- `.cloudopsworks/vars/preview/inputs.yaml`
- `.cloudopsworks/vars/preview/values.yaml`

---

### Step 6 — update `_VERSION`

After all merges are verified correct, write the target version string (e.g., `v1.4.16`) to `.cloudopsworks/_VERSION`. This is the final step.

---

### Upgrade invariants

An agent performing this upgrade must **never**:

- Overwrite a field the operator has explicitly set to a non-placeholder value.
- Remove a commented-out operator value without first reporting it.
- Change the YAML structure of any active (uncommented) operator section.
- Alter a file's opening description comment (`# This file contains...`) unless the upstream version changed it.
- Modify `.cloudopsworks/cloudopsworks-ci.yaml`, `gitversion_*.yaml`, or any file under `.github/workflows/` as part of a vars upgrade — those follow their own upgrade path.
- Update `_VERSION` before all file merges are complete.

---

### Conflict resolution

When a merge cannot be resolved automatically (for example, the upstream template restructured a section that the operator has customized):

1. Emit a diff showing both the upstream template block and the local operator block side by side.
2. Pause and present the conflict to the operator, asking which version to keep or whether a manual merge is needed.
3. Never silently choose one side.
