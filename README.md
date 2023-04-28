<p style="text-align: center;"><img src=".github/assets/nx.png"
width="100%" alt="Nx - Smart, Extensible Build Framework"></p>

<h1 align="center">Nx Cloud Github Workflows</h1>

✨ Github Workflows for easily configuring distributed CI pipelines powered by the speed and intelligence of Nx Cloud.

- [Disclaimer: Github Workflow Limitations](#disclaimer-github-workflow-limitations)
- [Example Usage](#example-usage)
- [Limited Secrets Support](#limited-secrets-support)
- [Configuration Options for the Main Job (nx-cloud-main.yml)](#configuration-options-for-the-main-job-nx-cloud-mainyml)
- [Configuration Options for Agent Jobs (nx-cloud-agents.yml)](#configuration-options-for-agent-jobs-nx-cloud-agentsyml)

## Disclaimer: Github Workflow Limitations

These workflows are intended to save folks time and complexity in getting up and running with Nx Cloud and Github Actions. They are _NOT_ intended to be a one-size-fits-all CI solution.

They will scale to workspaces of essentially any size thanks to the way they can distribute and parallelize tasks, however, that does not mean that they are infinitely configurable.

The workflows found here leverage a relatively new feature of Github called reusable workflows: https://docs.github.com/en/actions/using-workflows/reusing-workflows

The extensibility of these Nx Cloud workflows is therefore strictly bound by the capabilities of the Github feature.

This means that you cannot do things such as embed additional Github actions within the workflow, or majorly customize the steps we have set up for you.

If you find yourself needing to customize things beyond what is supported by Github reusable workflows, then the best way is to simply take a look at the source of the workflows within this repo [./.github/workflows](./.github/workflows) and use that as your starting point directly within your configs.

> NOTE: Another really important limitation of these reusable Github workflows is that you are not allowed to pass data from the special `${{ secrets }}` namespace into the workflow you are invoking (i.e. `nrwl/ci`). If you find yourself needing to use secret values beyond the small number we have explicitly built support for, you will again need to use this project as inspiration, rather than using it directly.

## Example Usage

The following will configure a CI workflow that runs on the `main` branch, and on all pull requests into the `main` branch. The CI workflow will do the following:

- Checkout your repo at the appropriate depth for determining affected projects
- Determine the `head` and `base` SHAs to use for `nx affected`
- Install nodejs
- Use an appropriate node_module caching strategy for either `yarn`, `npm` or `pnpm` (depending on which one is configured in your repo)
- Install locked dependencies with either `yarn`, `npm` or `pnpm` (depending on which one is configured in your repo)
- Spawn 3 agents ready to receive tasks/targets to run
- Initiate all the provided commands in parallel, with those defined in `parallel-commands` being executed on the main job, and those in `parallel-commands-on-agents` being intelligently distributed across the 3 agents by Nx Cloud
- Shut down all agents when all parallel tasks have been completed

**.github/workflows/ci.yml**

<!-- start example-usage -->

```yaml
name: CI

on:
  push:
    branches:
      - main
  pull_request:

concurrency:
  group: ${{ github.workflow }}-${{ github.event.number || github.ref }}
  cancel-in-progress: true

jobs:
  main:
    name: Nx Cloud - Main Job
    uses: nrwl/ci/.github/workflows/nx-cloud-main.yml@v0.13.0
    with:
      # NOTE: Here we are using the special `nx-cloud record` command to ensure that any commands we run that do not go through the cloud task runner natively
      # (i.e. anything that starts with `nx run`/`nx run-many`/`nx affected --target`), are still captured in the Nx Cloud UI and Github App comment for
      # easier troubleshooting. See more here: https://nx.dev/nx-cloud/set-up/record-commands#recording-non-nx-commands
      parallel-commands: |
        npx nx-cloud record -- npx nx format:check
      parallel-commands-on-agents: |
        npx nx affected --target=lint --parallel=3
        npx nx affected --target=test --parallel=3 --ci --code-coverage
        npx nx affected --target=build --parallel=3

  agents:
    name: Nx Cloud - Agents
    uses: nrwl/ci/.github/workflows/nx-cloud-agents.yml@v0.13.0
    with:
      number-of-agents: 3
```

<!-- end example-usage -->

## Limited Secrets Support

As noted above, it is not possible to generically forward values from the special `${{ secrets }}` namespace into this workflow, because of the limitations of Github reusable workflows.

Nevertheless, we have set up some explicit handling for the most common secret values that folks would need in the simple CI setups that this workflow is intended to support.

For example, the main and agent workflows both support passing `NX_CLOUD_AUTH_TOKEN` and `NX_CLOUD_ACCESS_TOKEN` from the parent workflow. These secrets are still kept encrypted and the `main` workflow will only use the `NX_CLOUD_AUTH_TOKEN` and `NX_CLOUD_ACCESS_TOKEN` if those are defined.

**.github/workflows/ci.yml**

<!-- start example-usage -->

```yaml
name: CI

on:
  push:
    branches:
      - main
  pull_request:

concurrency:
  group: ${{ github.workflow }}-${{ github.event.number || github.ref }}
  cancel-in-progress: true

jobs:
  main:
    name: Nx Cloud - Main Job
    uses: nrwl/ci/.github/workflows/nx-cloud-main.yml@v0.13.0
    secrets:
      NX_CLOUD_ACCESS_TOKEN: ${{ secrets.NX_CLOUD_ACCESS_TOKEN }}
      NX_CLOUD_AUTH_TOKEN: ${{ secrets.NX_CLOUD_AUTH_TOKEN }}
    with: ...

  agents:
    name: Nx Cloud - Agents
    uses: nrwl/ci/.github/workflows/nx-cloud-agents.yml@v0.13.0
    secrets:
      NX_CLOUD_ACCESS_TOKEN: ${{ secrets.NX_CLOUD_ACCESS_TOKEN }}
      NX_CLOUD_AUTH_TOKEN: ${{ secrets.NX_CLOUD_AUTH_TOKEN }}
    with: ...
```

See the annotated configuration below for all explicitly supported secret values.

<!-- end example-usage -->

## Configuration Options for the Main Job (nx-cloud-main.yml)

<!-- start configuration-options-for-the-main-job -->

```yaml
- uses: nrwl/ci/.github/workflows/nx-cloud-main.yml@v0.13.0
  # [OPTIONAL] Explicitly supported secret values which can be passed into the workflow from your outer workflow run.
  #
  # NOTE: You cannot access values from ${{ secrets }} beyond what is explicitly specified here because of the limitations of reusable Github workflows
  secrets:
    NPM_TOKEN: ""
    NX_CLOUD_ACCESS_TOKEN: ""
    NX_CLOUD_AUTH_TOKEN: ""
  with:
    # [OPTIONAL] The available number of agents used by the Nx Cloud to distribute tasks in parallel.
    # By default, NxCloud tries to infer dynamically how many agents you have available. Some agents
    # can have delayed start leading to incorrect count when distributing tasks.
    #
    # If you know exactly how many agents you have available, it is recommended to set this so we can more
    # reliably distribute the tasks.
    number-of-agents: 3

    # [OPTIONAL] A multi-line string containing non-secret environment variables which need to be passed from the parent workflow
    # to the reusable workflow. The variables are defined in form `VARIABLE_NAME=value`
    #
    # NOTE: Environment variables cannot contain values derived from ${{ secrets }}
    # because of how reusable workflows work
    environment-variables: |
      ""

    # [OPTIONAL] A multi-line string representing any bash commands (separated by new lines) which should
    # run sequentially, directly on the main job BEFORE executing any of the parallel commands which
    # may be specified via parallel-commands and/or parallel-commands-on-agents
    init-commands: |
      ""

    # [OPTIONAL] A multi-line string representing any bash commands (separated by new lines) which should
    # run in parallel, directly on the main job at the same time as any commands which may have beeen
    # specified via parallel-commands-on-agents
    #
    # NOTE: Due to how each stringified command gets interpreted in order to parallelize it, there may be
    # some limitations in terms of quote escaping and variable assignments within the command itself.
    parallel-commands: |
      ""

    # [OPTIONAL] A multi-line string representing any bash commands (separated by new lines) which should
    # run in parallel, preferentially distributed across any available agents at the same time as any
    # commands which may have been specified via parallel-commands
    #
    # NOTE: Due to how each stringified command gets interpreted in order to parallelize it, there may be
    # some limitations in terms of quote escaping and variable assignments within the command itself.
    parallel-commands-on-agents: |
      ""

    # [OPTIONAL] A multi-line string representing any bash commands (separated by new lines) which should
    # run sequentially, directly on the main job AFTER executing any of the parallel commands which
    # may be specified via parallel-commands and/or parallel-commands-on-agents
    final-commands: |
      ""

    # [OPTIONAL] The "main" branch of your repository (the base branch which you target with PRs).
    # Common names for this branch include main and master.
    #
    # Default: main
    main-branch-name: ""

    # [OPTIONAL] If you want to provide a specific node-version to use you can do that here.
    # If you do not specify one, it will respect an optional volta config you might have in
    # your package.json, otherwise it will simply install the latest LTS version of node.
    node-version: ""

    # [OPTIONAL] If you want to provide a specific npm-version to use you can do that here.
    # If you do not specify one, it will respect an optional volta config you might have in
    # your package.json, otherwise it will simply install the version of npm which is bundled
    # with the latest LTS version of node.
    npm-version: ""

    # [OPTIONAL] If you want to provide a specific yarn v1 version to use you can do that here.
    # If you do not specify one, it will respect an optional volta config you might have in
    # your package.json, otherwise it will simply install the latest v1 version of yarn available.
    yarn-version: ""

    # [OPTIONAL] If you want to provide a specific pnpm version to use you can do that here.
    # If you do not specify one, it will install the latest version of pnpm available.
    # Pnpm gets installed only if pnpm-lock.yaml is present in your repo.
    pnpm-version: ""

    # [OPTIONAL] If you want to provide specific install commands to use when installing dependencies
    # you can do that here. The default install step is not executed when this input is given.
    install-commands: ""

    # [OPTIONAL] Provides override for type of the machine to run the workflow on
    # The machine can be either a GitHub-hosted runner or a self-hosted runner
    #
    # NOTE: If you change this option, make sure it matches the agent configuration
    # Default: ubuntu-latest
    runs-on: ""

    # [OPTIONAL] If you want to upload artifacts, please provide the paths as are required by the
    # [upload-artifact](https://github.com/actions/upload-artifact) action. The name of the artifacts
    # will be the value of the `artifacts-name` input.
    #
    # NOTE: To download your artifact in another job you need to use the [download-artifact](https://github.com/actions/download-artifact) action
    # Default: ""
    artifacts-path: ""

    # [OPTIONAL] Provide the name of uploaded artifacts.
    #
    # NOTE: This input only has an effect if used with the `artifacts-path` input.
    # Default: "nx-main-artifacts"
    artifacts-name: ""
    
    # [OPTIONAL] If you want to override GitHub's default timeout provide a timeout expressed in minutes.
    #
    # NOTE: Timeout should be expressed in minutes
    # Default: 360
    timeout: ""
```

<!-- end configuration-options-for-the-main-job -->

## Configuration Options for Agent Jobs (nx-cloud-agents.yml)

<!-- start configuration-options-for-agent-jobs -->

```yaml
- uses: nrwl/ci/.github/workflows/nx-cloud-agents.yml@v0.13.0
  # [OPTIONAL] Explicitly supported secret values which can be passed into the workflow from your outer workflow run.
  #
  # NOTE: You cannot access values from ${{ secrets }} beyond what is explicitly specified here because of the limitations of reusable Github workflows
  secrets:
    NPM_TOKEN: ""
    NX_CLOUD_ACCESS_TOKEN: ""
    NX_CLOUD_AUTH_TOKEN: ""
  with:
    # [REQUIRED] The number of agents which should be created as part of the workflow in order to
    # allow Nx Cloud to intelligently distribute tasks in parallel.
    number-of-agents: 3

    # [OPTIONAL] A multi-line string containing non-secret environment variables which need to be passed from the parent workflow
    # to the reusable workflow. The variables are defined in form `VARIABLE_NAME=value`
    #
    # NOTE: Environment variables cannot contain values derived from ${{ secrets }}
    # because of how reusable workflows work
    environment-variables: |
      ""

    # [OPTIONAL] If you want to provide a specific node-version to use you can do that here.
    # If you do not specify one, it will respect an optional volta config you might have in
    # your package.json, otherwise it will simply install the latest LTS version of node.
    node-version: ""

    # [OPTIONAL] If you want to provide a specific npm-version to use you can do that here.
    # If you do not specify one, it will respect an optional volta config you might have in
    # your package.json, otherwise it will simply install the version of npm which is bundled
    # with the latest LTS version of node.
    npm-version: ""

    # [OPTIONAL] If you want to provide a specific yarn v1 version to use you can do that here.
    # If you do not specify one, it will respect an optional volta config you might have in
    # your package.json, otherwise it will simply install the latest v1 version of yarn available.
    yarn-version: ""

    # [OPTIONAL] If you want to provide a specific pnpm version to use you can do that here.
    # If you do not specify one, it will install the latest version of pnpm available.
    # Pnpm gets installed only if pnpm-lock.yaml is present in your repo.
    pnpm-version: ""

    # [OPTIONAL] If you want to provide specific install commands to use when installing dependencies
    # you can do that here. The default install step is not executed when this input is given.
    install-commands: ""

    # [OPTIONAL] Provides override for type of the machine to run the workflow on
    # The machine can be either a GitHub-hosted runner or a self-hosted runner
    #
    # NOTE: If you change this option, make sure it matches the main configuration
    # Default: ubuntu-latest
    runs-on: ""

    # [OPTIONAL] If you want to override GitHub's default timeout provide a timeout expressed in minutes.
    #
    # NOTE: Timeout should be expressed in minutes
    # Default: 360
    timeout: ""
```

<!-- end configuration-options-for-agent-jobs -->
