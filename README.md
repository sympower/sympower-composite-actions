# Sympower Common Composite Actions

This repository contains Sympower's reusable GitHub composed actions to be called/reused by the workflows of each 
individual repository.

## Actions in this repository

### setup-build-environment

All jobs that want to use composite actions from this repository need to run this action first, unless explicitly
documented here that the action is self-contained and does not need this action. 

This action:
* checks out the repository,
* configures Gradle and credentials for it,
* configures AWS credentials and logs in to ECR,
* populates `env.REPOSITORY_NAME` environment variable.

Required inputs for this GitHub actions:
* `secrets` - JSON string of the GitHub secrets.

Example of usage:
```yaml
jobs:
  build:
    runs-on: ubuntu-latest
    env:
      secrets: ${{ toJSON(secrets) }}
    steps:
      - id: setup-build-environment
        name: "Setup build environment"
        uses: sympower/sympower-composite-actions/setup-build-environment@{LATEST_VERSION}
        with:
          secrets: ${{ env.secrets }}
```

### format-version

**This action can be used without `setup-build-environment` action running first.**

Based on git repository information, `format-version` action outputs a formatted version. This action has two modes of 
operation depending on `style-as-release` flag:

* `true` - Outputs a version in the format `yyyy.MM.dd.HH.mm.ss-commit_hash`.
* `false` - Outputs first 20 characters of the branch name as the version.

Required inputs for this GitHub actions:
* `style-as-release` - `true`/`false`to determine the style of the version as explained above.

Example of usage:
```yaml
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - id: format-version
        name: "Format version"
        uses: sympower/sympower-composite-actions/format-version@{LATEST_VERSION}
        with:
          style-as-release: true
```

### run-tests

`run-tests` action executes Gradle `test`, `behaviourTest` and `integrationTest` tasks in this order. It will execute 
[phoenix-actions/test-reporting](https://github.com/phoenix-actions/test-reporting) action for those tests that have
test output files to report.

Example of usage:
```yaml
jobs:
  build:
    runs-on: ubuntu-latest
    env:
      secrets: ${{ toJSON(secrets) }}
    steps:
      - id: setup-build-environment
        name: "Setup build environment"
        uses: sympower/sympower-composite-actions/setup-build-environment@{LATEST_VERSION}
        with:
          secrets: ${{ env.secrets }}
      - id: run-tests
        name: "Run tests"
        uses: sympower/sympower-composite-actions/run-tests@{LATEST_VERSION}
```

### code-analysis

`code-analysis` action runs Sonar code analysis Gradle task. Preferably this action should run after `run-tests` action 
to ensure code coverage is included in the analysis.

Required inputs for this GitHub actions:
* `secrets` - JSON string of the GitHub secrets.

Example of usage:
```yaml
jobs:
  build:
    runs-on: ubuntu-latest
    env:
      secrets: ${{ toJSON(secrets) }}
    steps:
      - id: setup-build-environment
        name: "Setup build environment"
        uses: sympower/sympower-composite-actions/setup-build-environment@{LATEST_VERSION}
        with:
          secrets: ${{ env.secrets }}
      - id: run-tests
        name: "Run tests"
        uses: sympower/sympower-composite-actions/run-tests@{LATEST_VERSION}
      - id: code-analysis
        name: "Code analysis"
        uses: sympower/sympower-composite-actions/code-analysis@v2023.10.30.15.12-299ab4a
        with:
          secrets: ${{ env.secrets }}
```

### build-and-upload-docker-image

`build-and-upload-docker-image` action uses jib Gradle plugin to build and upload a multi-arch Docker image of the 
Java application(s) present int the repo to ECR. 

`format-version` action can be used to format the repo version for this action, but it is not required as you can choose 
to use any version you want as the input. 

Required inputs for this GitHub actions:
* `version` - Version of the Docker image to be published.

Example of usage:
```yaml
jobs:
  build:
    runs-on: ubuntu-latest
    env:
      secrets: ${{ toJSON(secrets) }}
    steps:
      - id: setup-build-environment
        name: "Setup build environment"
        uses: sympower/sympower-composite-actions/setup-build-environment@{LATEST_VERSION}
        with:
          secrets: ${{ env.secrets }}
      - id: format-version
        name: "Format version"
        uses: sympower/sympower-composite-actions/format-version@{LATEST_VERSION}
        with:
          style-as-release: true
      - id: build-and-upload-docker-image
        name: "Build and upload Docker Image"
        uses: sympower/sympower-composite-actions/build-and-upload-docker-image@{LATEST_VERSION}
        with:
          version: ${{ steps.format-version.outputs.version }}
```

### upload-schema

`upload-schema` action detects if there are any file changes in the schema module. Directory named `schema` is checked
by default, however this can be overridden by providing a different directory name as `schema-module` input for this
action. If changes are detected then runs Gradle publish command to ensure schema artifact is published to Nexus.

`format-version` action can be used to format the repo version for this action, but it is not required as you can choose 
to use any version you want as the input.

Required inputs for this GitHub actions:
* `version` - Version of the artifact to be published.
* `gistID` - ID of the gist with the version badge to update.
* `secrets` - JSON string of the GitHub secrets.

Example of usage:
```yaml
jobs:
  build:
    runs-on: ubuntu-latest
    env:
      secrets: ${{ toJSON(secrets) }}
    steps:
      - id: setup-build-environment
        name: "Setup build environment"
        uses: sympower/sympower-composite-actions/setup-build-environment@{LATEST_VERSION}
        with:
          secrets: ${{ env.secrets }}
      - id: format-version
        name: "Format version"
        uses: sympower/sympower-composite-actions/format-version@{LATEST_VERSION}
        with:
          style-as-release: true
      - id: upload-schema
        name: "Upload schema"
        uses: sympower/sympower-composite-actions/upload-schema@{LATEST_VERSION}
        with:
          version: ${{ steps.format-version.outputs.version }}
          secrets: ${{ env.secrets }}
          gistID: {some-valid-gist-id}
```

### upload-pacts

`upload-pacts` action detects if there are any Pact contracts that have been produced by the build process. If so, then
it will upload them to Pact Broker.

`format-version` action can be used to format the repo version for this action, but it is not required as you can choose
to use any version you want as the input.

Required inputs for this GitHub actions:
* `version` - Version of the contract to be published.

Example of usage:
```yaml
jobs:
  build:
    runs-on: ubuntu-latest
    env:
      secrets: ${{ toJSON(secrets) }}
    steps:
      - id: setup-build-environment
        name: "Setup build environment"
        uses: sympower/sympower-composite-actions/setup-build-environment@{LATEST_VERSION}
        with:
          secrets: ${{ env.secrets }}
      - id: format-version
        name: "Format version"
        uses: sympower/sympower-composite-actions/format-version@{LATEST_VERSION}
        with:
          style-as-release: true
          
      # Example of an action that might produce Pact contracts     
      - id: run-tests
        name: "Run tests"
        uses: sympower/sympower-composite-actions/run-tests@{LATEST_VERSION}
      # end-of-pact-source-example
        
      - id: upload-pacts
        name: "Upload pacts"
        if: env.IS_DEFAULT_BRANCH == 'true'
        uses: sympower/sympower-composite-actions/upload-pacts@{LATEST_VERSION}
        with:
          version: ${{ steps.format-version.outputs.version }}
```

### deploy-to-environment

`deploy-to-environment` action searches for JSON files with file extension `.env.json` from `auto-deploy` directory. 
Based on these files and inputs configurations for the action, it will call deployment GitHub events in
[environments](https://github.com/sympower/environments) repository.

Auto-deploy environment JSON file fields:
* `environment` **required** - Declares the environment to deploy. Usual values: `staging`, `platform`.
* `deployGroup` **optional** - Optional field to create deployment groups if there is a need to have multiple
  `deploy-to-environment` action calls for services in the same environment. If not declared then `environment` value is
  automatically used as the deployment group name. This value is used to match `deploy-group` action input to
  determine if the action should include this `auto-deploy` file in the deployment call.
* `component` **optional** - Declares the component to deploy into. This declares the EC2 instance where the service is 
  running in, and it corresponds to the folder name where `main.yml` declaration containing the service in the
  [environments](https://github.com/sympower/environments) repository. If not declared then value from `default-name` 
  input is used. In most cases repository name is matches the component name and is also inputted as `default-name`.
* `service` **optional** - Declares the name of service to deploy. Corresponds to the service name in `main.yml` 
  declaration chosen by `component` field in this file. If not declared then value from `default-name` input is used. 
  In most cases repository name is matches the service name and is also inputted as `default-name`.

In most cases `auto-deploy/*.env.json` files can be only include environment:
* Staging:
```json
{
  "environment": "staging"
}
```
* Platform (production):
```json
{
  "environment": "platform"
}
```
* However, here is a full example of an `auto-deploy/*.env.json` file:
```json
{
  "environment": "platform",
  "deployGroup": "platform-fi",
  "component": "msa-some-adapter-fi",
  "service": "msa-some-adapter"
}
```

`format-version` action can be used to format the repo version for this action, but it is not required as you can choose
to use any version you want as the input.

Required inputs for this GitHub actions:
* `version` - Version to use when calling deployment.
* `secrets` - JSON string of the GitHub secrets.
* `default-name` - Default name to use for `component` and `service` fields in `auto-deploy` JSON files if either of 
  those are missing.
* `deploy-group` - Deployment group name to use when calling deployment to choose which 

Example of usage:
```yaml
jobs:
  build:
    runs-on: ubuntu-latest
    env:
      secrets: ${{ toJSON(secrets) }}
    steps:
      - id: setup-build-environment
        name: "Setup build environment"
        uses: sympower/sympower-composite-actions/setup-build-environment@{LATEST_VERSION}
        with:
          secrets: ${{ env.secrets }}
      - id: format-version
        name: "Format version"
        uses: sympower/sympower-composite-actions/format-version@{LATEST_VERSION}
        with:
          style-as-release: true
      - id: build-and-upload-docker-image
        name: "Build and upload Docker Image"
        uses: sympower/sympower-composite-actions/build-and-upload-docker-image@{LATEST_VERSION}
        with:
          version: ${{ steps.format-version.outputs.version }}
      - id: deploy-to-platform
        name: "Deploy Platform (Production)"
        uses: sympower/sympower-composite-actions/deploy-to-environment@{LATEST_VERSION}
        with:
          secrets: ${{ env.secrets }}
          version: ${{ steps.format-version.outputs.version }}
          default-name: ${{ env.REPOSITORY_NAME }}
          deploy-group: platform
```

### upload-build-artifacts

**This action can be used without `setup-build-environment` action running first.**

`upload-build-artifacts` uploads build reports (`**/build/reports/**`) as a zip file if there are any. It is recommended 
to include `if: always()` clause with this action to ensure it is always executed even if previous steps fail.

Example of usage:
```yaml
jobs:
  build:
    runs-on: ubuntu-latest
    env:
      secrets: ${{ toJSON(secrets) }}
    steps:
      # Example of an action that might produce build reports
      - id: setup-build-environment
        name: "Setup build environment"
        uses: sympower/sympower-composite-actions/setup-build-environment@{LATEST_VERSION}
        with:
          secrets: ${{ env.secrets }}
      - id: run-tests
        name: "Run tests"
        uses: sympower/sympower-composite-actions/run-tests@{LATEST_VERSION}
      # end-of-report-source-example
      
      - id: upload-build-artifacts
        name: "Upload build artifacts"
        if: always()
        uses: sympower/sympower-composite-actions/upload-build-artifacts@{LATEST_VERSION}
```

## Releasing new actions

Merging your changes to main will automatically create a new release and tag. Renovate is able to pick this up for most
repositories. However, for [sympower-actions](https://github.com/sympower/sympower-actions) Renovate does not work, 
so it will need manual intervention.
