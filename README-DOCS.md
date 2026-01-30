# CloudOpsWorks XCode Application Template Documentation

This document provides a comprehensive technical review and step-by-step documentation of the XCode Application Template module. This module is designed for automated CI/CD using CloudOpsWorks Blueprints and is optimized for XCode development and deployment to mobile testing clouds.

## Table of Contents
1. [Project Overview](#project-overview)
2. [Workflows and Automation Blueprints](#workflows-and-automation-blueprints)
    - [Workflow Descriptions](#workflow-descriptions)
3. [Module Configuration (.cloudopsworks/vars)](#module-configuration-cloudopsworksvars)
    - [cloudopsworks-ci.yaml](#cloudopsworks-ciyaml)
    - [inputs-global.yaml (Global Configuration)](#inputs-globalyaml-global-configuration)
    - [inputs-***.yaml (Environment Specific)](#inputs-yaml-environment-specific)
4. [Cloud Provider Integrations](#cloud-provider-integrations)
    - [AWS Device Farm](#aws-device-farm)
    - [Google Firebase Test Lab](#google-firebase-test-lab)
    - [Azure](#azure)
5. [Swift Application Template](#swift-application-template)

---

## Project Overview

The XCode Application Template provides a standardized structure for iOS/macOS applications, integrating seamlessly with CloudOpsWorks automation blueprints. It automates the entire lifecycle from building and testing to security scanning and cloud-based device testing.

---

## Workflows and Automation Blueprints

Automation is driven by GitHub Actions located in `.github/workflows`. These workflows reference CloudOpsWorks blueprints using the `./bp` prefix, which points to a library of standardized CI/CD actions.

### Workflow Descriptions

#### 1. Main Build (`main-build.yml`)
*   **Trigger**: Triggered on pushes to `develop`, `release/**`, `support/**` branches, and version tags (`v*.*.*`).
*   **Blueprint Actions Used**:
    *   `./bp/ci/config`: Initializes the pipeline configuration.
    *   `./bp/ci/xcode/config`: Loads XCode-specific settings for the target environment.
    *   `./bp/ci/xcode/build`: Performs the actual XCode build, handling certificates, provisioning profiles, and IPA export.
    *   `./bp/ci/xcode/artifacts`: Packages and saves build artifacts (.ipa, .app, etc.).
    *   `./bp/cd/release`: Handles GitHub Release creation and artifact attachment.

#### 2. PR Build (`pr-build.yml`)
*   **Trigger**: Triggered on Pull Requests targeting major branches.
*   **Function**: Validates the PR by performing a full build and security scan.
*   **Blueprint Actions Used**:
    *   `./bp/cd/tasks/repo/checkpr`: Validates PR compliance.
    *   `./bp/ci/xcode/build`: Ensures the code compiles and builds correctly.
    *   `./bp/ci/xcode/artifacts`: Saves build artifacts for inspection.
*   *Note: Preview environments are currently not supported for XCode targets.*

#### 3. Continuous Deployment (`deploy.yml`)
*   **Trigger**: Called as a reusable workflow by `main-build.yml`.
*   **Blueprint Actions Used**:
    *   `./bp/cd/deploy/mobile/aws`: Deploys to AWS Device Farm.
    *   `./bp/cd/deploy/mobile/azure`: Deploys to Azure mobile testing services.
    *   `./bp/cd/deploy/mobile/gcp`: Deploys to Firebase Test Lab.

#### 4. Code Scan (`scan.yml`)
*   **Function**: Orchestrates security and quality analysis.
*   **Blueprint Actions Used**:
    *   `./bp/ci/scan/config`: Initializes scanning configuration.
    *   `./bp/ci/xcode/scan/semgrep`: Performs Static Analysis Security Testing (SAST).
    *   `./bp/ci/xcode/scan/snyk`: Performs Software Composition Analysis (SCA).
    *   `./bp/ci/xcode/scan/sonarqube`: Analyzes code quality and coverage.
    *   `./bp/ci/scan/sonarqube/quality-gate`: Enforces SonarQube quality gates.
    *   `./bp/ci/xcode/scan/dtrack`: Integrates with DependencyTrack for BOM management.

---

## Module Configuration (.cloudopsworks/vars)

The configuration is centralized in the `.cloudopsworks/vars` directory. These files define how the automation blueprints behave.

### cloudopsworks-ci.yaml

This file is the entry point for CI/CD behavior and environment mapping. It ties the execution of corresponding inputs files for each environment.

| Parameter | Description |
| :--- | :--- |
| `zipGlobs` | List of glob patterns for files to be included in the build artifact zip. |
| `excludeGlobs` | List of glob patterns for files to be excluded from processing. |
| `config.branchProtection` | Boolean to enable/disable automated branch protection. |
| `config.gitFlow.enabled` | Boolean to enable GitFlow workflow support. |
| `config.requiredReviewers` | Number of required reviewers for PRs. |
| `config.owners` / `contributors` | RBAC configuration for repository access. |
| `cd.deployments` | Maps Git branches to deployment environments (`env`). |
| `cd.deployments.<branch>.env` | The environment name (e.g., `dev`, `uat`, `prod`) that corresponds to a branch push. This name is used to select the `inputs-<ENV>.yaml` file. |

### inputs-global.yaml (Global Configuration)

This file contains the base configuration used across all environments. It is the most detailed part of the configuration.

#### Core Settings
*   **`organization_name`**: Name of the organization.
*   **`organization_unit`**: Department or unit within the organization.
*   **`environment_name`**: Default environment name.
*   **`repository_owner`**: GitHub owner/organization of the repository.

#### XCode Settings (`xcode:`)
*   **`version`**: The XCode version to be used by the runner (e.g., `16.4`).
*   **`extra_args`**: Additional command-line arguments for `xcodebuild` (e.g., `-allowProvisioningUpdates`).
*   **`extra_targets`**: (Optional) Additional build targets, often required for cloud testing.
*   **`build_for_testing`**: Boolean. Set to `true` if building for UI/Unit tests.
*   **`unsigned`**: Boolean. If `true`, skips the signing process.
*   **`dev_team`**: (Required) Your Apple Developer Team ID.
*   **`product_bundle`**: (Required) The bundle identifier of your application.
*   **`export_options`**: Configuration for the IPA export process.
    *   `type`: Export type (e.g., `ad_hoc`, `app-store`, `enterprise`, `development`).
    *   `method`: Export method.
    *   `profile`: Name of the provisioning profile.
    *   `app_url` / `image_url`: URLs for manifest generation (Enterprise/Ad-Hoc).
*   **`schemes`**: List of XCode schemes to build.
*   **`sdks`**: List of SDKs to target (e.g., `iphoneos`).
*   **`destinations`**: Build destinations (e.g., `generic/platform=iOS`).

#### Security & Quality Tools
*   **`snyk.enabled`**: Enables Snyk SCA scanning.
*   **`semgrep.enabled`**: Enables Semgrep SAST scanning.
*   **`sonarqube`**:
    *   `enabled`: Enables SonarQube analysis.
    *   `fail_on_quality_gate`: If true, fails the pipeline if the quality gate is not met.
    *   `sources_path` / `tests_path`: Paths for code and test analysis.
    *   `exclusions`: Patterns to exclude from scanning.
*   **`dependencyTrack`**:
    *   `enabled`: Enables DependencyTrack integration.
    *   `type`: Component type (e.g., `Application`, `Library`).

#### Cloud and Runner Settings
*   **`cloud`**: Target cloud provider (`aws`, `azure`, or `gcp`).
*   **`cloud_type`**: Deployment target type (e.g., `device-farm` for AWS/GCP).
*   **`runner_set`**: The GitHub Runner label to use (e.g., `macos-latest`).

### inputs-***.yaml (Environment Specific)

These files (e.g., `inputs-XCODE-ENV.yaml`) provide configuration for specific deployment targets. **Only one environment-specific file is active per deployment**, determined by the `env` mapping in `cloudopsworks-ci.yaml`.

*   **`environment`**: The environment name (e.g., `dev`, `uat`, `prod`).
*   **`disable_deploy`**: Set to `true` to skip the deployment step (useful if only building/scanning).
*   **`xcode.configuration`**: Build configuration (e.g., `Debug`, `Release`).
*   **`runner_set`**: Environment-specific runner override.

---

## Cloud Provider Integrations

### AWS Device Farm

Configuration under the `aws:` key in environment-specific files:

*   **`region`**: AWS Region (e.g., `us-west-2`).
*   **`sts_role_arn`**: IAM Role ARN for deployment permissions.
*   **`device_farm_name`**: Name of the Device Farm project.
*   **`device_farm_pool`**: Name of the device pool to use.
*   **`device`**: (Used if `device_farm_pool` is not defined)
    *   `form_factor`: `PHONE` or `TABLET`.
    *   `model`: Specific device model (e.g., `Apple iPhone 14 Pro`).
    *   `os_version`: Target OS version.
*   **`test_script_path`**: Path to the test scripts within the project.
*   **`test_script_type`**: Type of test package (e.g., `XCTEST_UI_TEST_SPEC`, `APPIUM_JAVA_JUNIT_TEST_SPEC`).

### Google Firebase Test Lab

Configuration under the `gcp:` key in environment-specific files:

*   **`project_id`**: Google Cloud Project ID.
*   **`region`**: GCP Region.
*   **`impersonate_sa`**: Service Account email for impersonation.
*   **`devices`**: List of target devices.
    *   `model_id`: Device model (e.g., `iphone14pro`).
    *   `version_id`: iOS version.
    *   `locale_id`: Locale (e.g., `en_US`).
    *   `orientation`: `portrait` or `landscape`.
*   **`test_lab_bucket`**: GCS bucket for test results.
*   **`test_script_type`**: Test type (e.g., `xctest`, `robo`, `game-loop`).

### Azure

Configuration under the `azure:` key in environment-specific files:

*   **`resource_group`**: Target Azure Resource Group.
*   **`keyvault_name`**: Azure Key Vault for retrieving secrets.
*   **`keyvault_secret_filter`**: Filter for specific secrets to be injected.
*   **`pod_identity`**: Configuration for managed identities.

---

## Swift Application Template

The project includes a boilerplate Swift application located in the `I am Groot` directory.

*   **Architecture**: Standard iOS application structure using UIKit.
*   **Components**:
    *   `AppDelegate.swift` & `SceneDelegate.swift`: Standard lifecycle management.
    *   `ViewController.swift`: Main entry point for the application's UI.
    *   `Main.storyboard` & `LaunchScreen.storyboard`: UI layout and launch screen.
    *   `Assets.xcassets`: Management of app icons and images.
*   **Tests**:
    *   `I am GrootUITests`: Template for UI testing using XCTest.

The application serves as a foundation for developing iOS apps that are fully integrated with the CloudOpsWorks CI/CD ecosystem.
