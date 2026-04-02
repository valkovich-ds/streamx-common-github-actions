# streamx-common-github-actions

This repository contains a GitHub actions used in StreamX repositories

## Create JIRA Release GitHub Action

This GitHub action is designed to automate the creation of JIRA releases.
It operates by adding release version details to the `fix versions` field in a JIRA issues,
driven by task numbers derived from commits since the previous git tag.
The action should be set up to run subsequent to code checkouts in the earlier workflow steps.

### Inputs

This action requires the following inputs:

* `atlassianCloudUser`: Your Atlassian cloud user. This is required.
* `atlassianCloudApiKey`: Your Atlassian cloud ApiKey. This is required.
* `atlassianCloudDomain`: Your Atlassian cloud domain. This is required.
* `atlassianCloudJiraProject`: Your JIRA project. This is required.
* `releaseNamePrefix`: Your JIRA release name prefix. This is required.
* `tag`: This is an optional parameter. If provided, it will be used, otherwise, the latest git tag will be used.

### Usage

add following step to your GH workflow.

```yaml
      - name: Release Jira Version
        uses: streamx-hub/streamx-common-github-actions/.github/actions/jira-release@main
        with:
          atlassianCloudUser: ${{ secrets.ATLASSIAN_CLOUD_USER }}
          atlassianCloudApiKey: ${{ secrets.ATLASSIAN_CLOUD_APIKEY }}
          atlassianCloudDomain: ${{ secrets.ATLASSIAN_CLOUD_DOMAIN }}
          atlassianCloudJiraProject: ${{ vars.ATLASSIAN_CLOUD_JIRA_PROJECT }}
          releaseNamePrefix: ${{ github.event.repository.name }}
```

Secrets `ATLASSIAN_CLOUD_USER`, `ATLASSIAN_CLOUD_APIKEY`, and `ATLASSIAN_CLOUD_DOMAIN` are defined at the StreamX-hub organization level
and should be accessible for all repositories without any additional configuration.

The `ATLASSIAN_CLOUD_JIRA_PROJECT` variable is also defined at the organizational level.


## StreamX GitHub Connector Action

StreamX GitHub Connector is a Quarkus GitHub Action project that allows syncing GitHub
changes with StreamX using CloudEvents.

Detail information please check [Quarkus GitHub Action](https://docs.quarkiverse.io/quarkus-github-action/dev/index.html) documentation.

The action sends CloudEvents to StreamX's ingestion API. Use `source-provider` to automatically detect and ingest files from the repository, or provide `subject` to send a single event for a specific resource.

### Usage
```yaml
- uses: streamx-hub/streamx-common-github-actions/.github/actions/connector-github@v1
  with:
    # CloudEvents event type (e.g., com.streamx.blueprints.web-resource.published.v1).
    #
    # Value required.
    event-type:

    # CloudEvents event type used for deletion events (e.g., com.streamx.blueprints.web-resource.unpublished.v1).
    deleted-event-type:

    # StreamX ingestion API endpoint URL (e.g., https://ingestion.streamx.com).
    #
    # Value required.
    streamx-ingestion-url:

    # StreamX ingestion API authorisation token.
    streamx-ingestion-token:

    # Specifies the provider name used to determine the ingestion data source.
    # Check section 'Ingestion data source provider' for details.
    #
    source-provider:

    # CloudEvents subject. Required when `source-provider` is not specified.
    # Also required by ExternalSourceProvider.
    subject:

    # Specifies the root directory for ingestion data lookup. Defaults to github.workspace.
    workspace:

    # Defines Ant-style path patterns for filtering ingestion data (e.g., styles/*.css, scripts/*.js).
    #
    # Value required only with a specific source provider. Check section 'Ingestion data source provider' for details.
    include-patterns:

    # URL property from where resource data is downloaded and used as ingestion data.
    #
    # Value required only with a specific source provider. Check section 'Ingestion data source provider' for details.
    external-resource-url:

    # When set to 'true', this configuration enables DEBUG-level logging during JBang execution.
    # Displays detailed output in the console, useful for troubleshooting and diagnostics.
    #
    # Default value 'false'.
    debug-enabled:
```

#### Ingestion data source provider

* *BatchSourceProvider*

Source provider used for source lookup inside the repository and returns the list of all resources
that matches given `include-patterns` configuration.

```yaml
  with:
    source-provider: BatchSourceProvider
    streamx-ingestion-url:
    event-type:
    # Value required for limiting number of the resources
    include-patterns:
```

* *ExternalSourceProvider*

This data source provider allows to download data from external services.

```yaml
  with:
    source-provider: ExternalSourceProvider
    streamx-ingestion-url:
    event-type:
    # URL property from where resource data is downloaded and used as ingestion data.
    external-resource-url:
    # CloudEvents subject identifying the resource.
    subject:
```

* *PullRequestDiffSourceProvider*

The source provider detects file changes, creations, and deletions associated with a given pull request GitHub event.

The step action must include a condition to ensure the event is a pull request and that the pull request has been merged
(e.g. `if:github.event.pull_request.merged == true`).
For accurate diff detection, all changelog data must be loaded during the checkout stage by setting the option:
`fetch-depth: 0`

```yaml
  with:
    source-provider: PullRequestDiffSourceProvider
    streamx-ingestion-url:
    event-type:
    deleted-event-type:
    # Value required for limiting number of the resources
    include-patterns:
```

### Scenarios

#### Ingest merged pull request changes to StreamX for CSS and JS web resources only

```yaml
on:
  pull_request:
    types:
      - closed
    branches:
      - main
  sync-pr-with-streamx:
    if: github.event.pull_request.merged == true
    runs-on: ubuntu-latest
    steps:
      - name: Checkout code
        uses: actions/checkout@v4
        with:
          fetch-depth: 0
      - name: Run sync with StreamX
        uses: streamx-hub/streamx-common-github-actions/.github/actions/connector-github@v1
        with:
          event-type: com.streamx.blueprints.web-resource.published.v1
          deleted-event-type: com.streamx.blueprints.web-resource.unpublished.v1
          source-provider: PullRequestDiffSourceProvider
          include-patterns: '[\"styles/*.css\", \"scripts/*.js\"]'
          streamx-ingestion-token: ${{ secrets.STREAMX_INGESTION_TOKEN }}
          streamx-ingestion-url: ${{ vars.STREAMX_INGESTION_URL }}

```

#### Publish with StreamX all CSS and JS web resources

```yaml
on:
  workflow_dispatch:
    inputs:
      publish_all_webresources:
        description: "Publish all, pattern included webresources to StreamX"
        required: false
        type: boolean
        default: false
  sync-all-with-streamx:
    if: github.event_name == 'workflow_dispatch' && inputs.publish_all_webresources == true
    runs-on: ubuntu-latest
    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Run full sync with StreamX
        uses: streamx-hub/streamx-common-github-actions/.github/actions/connector-github@v1
        with:
          event-type: com.streamx.blueprints.web-resource.published.v1
          source-provider: BatchSourceProvider
          include-patterns: '[\"styles/*.css\", \"scripts/*.js\"]'
          streamx-ingestion-token: ${{ secrets.STREAMX_INGESTION_TOKEN }}
          streamx-ingestion-url: ${{ vars.STREAMX_INGESTION_URL }}

```

#### Unpublish a specific resource from StreamX

Use `subject` to send a single event without scanning files — for example,
to unpublish a specific resource by its subject. This is useful for manual content removal
triggered via `workflow_dispatch`, where you already know exactly which resource to act on.

```yaml
on:
  workflow_dispatch:
    inputs:
      subject:
        description: "CloudEvents subject to unpublish (e.g., /styles/main.css)"
        required: true
        type: string
  unpublish-resource:
    runs-on: ubuntu-latest
    steps:
      - name: Unpublish resource from StreamX
        uses: streamx-hub/streamx-common-github-actions/.github/actions/connector-github@v1
        with:
          event-type: com.streamx.blueprints.web-resource.unpublished.v1
          subject: ${{ inputs.subject }}
          streamx-ingestion-token: ${{ secrets.STREAMX_INGESTION_TOKEN }}
          streamx-ingestion-url: ${{ vars.STREAMX_INGESTION_URL }}
```

### Enable dependency cache on GitHub workflows

Optimize your GitHub Actions workflow by caching dependencies. This simple step delivers
significant performance gains, reducing execution time from minutes to seconds by avoiding
repeated downloads. This is especially impactful for build processes with numerous or large
dependencies.

Before the JBang execution add these steps.
```yaml
    - name: Setup Java
      uses: actions/setup-java@v3
      with:
        java-version: '21'
        distribution: 'adopt'

    - name: Setup M2 cache
      uses: actions/cache@v4
      with:
        path: ~/.m2
        key: ${{ runner.os }}-maven-${{ hashFiles('.jbang/cache/dependency_cache.json') }}
        restore-keys: |
          ${{ runner.os }}-maven-

    - name: Set up JBang
      uses: jbangdev/setup-jbang@main

    - name: Configure java
      run: |
        jbang jdk install 21 ${{env.JAVA_HOME_21_X64}}
      shell: bash
```
