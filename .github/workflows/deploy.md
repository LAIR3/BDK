xplanation of the deploy.yml file used for deploying the Kurtosis CDK environment. It outlines the purpose of each job and step in the workflow.
Overview

The deploy.yml GitHub Actions workflow is designed to automate the deployment of the Kurtosis CDK environment. It triggers on pull requests and pushes to the main branch. The workflow supports three deployment modes:

    Monolithic: Deploys the entire environment in a single step.
    Incremental: Deploys the environment in stages.
    Configless: Deploys the environment without a specified parameter file.

Workflow Trigger

The workflow triggers on:

    pull_request
    push to the main branch

on:
  pull_request:
  push:
    branches: [main]


Concurrency Control

The workflow uses a concurrency group to prevent multiple deployments from running simultaneously on the same pull request or branch.

```bash
concurrency:
  group: deploy-${{ github.event.pull_request.number || github.ref }}
  cancel-in-progress: true
```

The workflow sets the following environment variables:

    KURTOSIS_VERSION: Version of the Kurtosis CLI to install.
    GO_VERSION: Version of Go to use.
    BATCH_VERIFICATION_MONITOR_TARGET: Target for batch verification monitoring.
    BATCH_VERIFICATION_MONITOR_TIMEOUT: Timeout for batch verification monitoring.

'''yaml
env:
  KURTOSIS_VERSION: 0.89.3
  GO_VERSION: 1.21
  BATCH_VERIFICATION_MONITOR_TARGET: 30
  BATCH_VERIFICATION_MONITOR_TIMEOUT: 600 # 10 minutes
```

Monolithic Deployment Job

This job deploys the entire Kurtosis CDK environment in one step, with the gas token feature enabled.

'''bash
jobs:
  monolithic:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-go@v5
        with:
          go-version: ${{ env.GO_VERSION }}
          cache-dependency-path: scripts/zkevm-config-diff/go.sum
```

    Kurtosis CLI: Installs the Kurtosis CLI.
    yq: Installs yq for YAML processing.
    Foundry: Installs Foundry for Ethereum development.

'''bash
      - name: Install kurtosis
        run: |
          echo "deb [trusted=yes] https://apt.fury.io/kurtosis-tech/ /" | sudo tee /etc/apt/sources.list.d/kurtosis.list
          sudo apt update
          sudo apt install kurtosis-cli=${{ env.KURTOSIS_VERSION }}
          kurtosis analytics disable

      - name: Install yq
        run: pip3 install yq

      - name: Install foundry
        uses: foundry-rs/foundry-toolchain@v1
```

    Enable Gas Token Feature: Enables the gas token feature in the parameter file.
    Deploy Kurtosis CDK Package: Deploys the CDK package.
    Check Batch Verification: Checks that batches are being verified.


'''bash
      - name: Enable gas token feature
        run: yq -Y --in-place '.args.zkevm_use_gas_token_contract = true' params.yml

      - name: Deploy Kurtosis CDK package
        run: kurtosis run --enclave cdk-v1 --args-file params.yml --image-download always .

      - name: Check that batches are being verified
        working-directory: ./scripts
        run: ./batch_verification_monitor.sh ${{ env.BATCH_VERIFICATION_MONITOR_TARGET }} ${{ env.BATCH_VERIFICATION_MONITOR_TIMEOUT }}
```

    Dump Configs: Dumps the default and Kurtosis CDK configurations.
    Compare Configs: Compares the dumped configurations.

'''bash
  incremental:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
```

    Kurtosis CLI: Installs the Kurtosis CLI.
    yq: Installs yq for YAML processing.
    Foundry: Installs Foundry for Ethereum development.

'''bash
      - name: Install kurtosis
        run: |
          echo "deb [trusted=yes] https://apt.fury.io/kurtosis-tech/ /" | sudo tee /etc/apt/sources.list.d/kurtosis.list
          sudo apt update
          sudo apt install kurtosis-cli=${{ env.KURTOSIS_VERSION }}
          kurtosis analytics disable

      - name: Install yq
        run: pip3 install yq

      - name: Install foundry
        uses: foundry-rs/foundry-toolchain@v1
```

Each deployment step is enabled in the parameter file, run, and then reset.

    Disable All Deployment Steps: Disables all deployment steps in the parameter file.
    Deploy L1: Deploys L1 components.
    Deploy zkEVM Contracts on L1: Deploys zkEVM contracts on L1.
    Deploy zkEVM Node and CDK Peripheral Databases: Deploys the zkEVM node and peripheral databases.
    Deploy CDK Central Environment: Deploys the CDK central environment.
    Deploy CDK Bridge Infrastructure: Deploys the CDK bridge infrastructure.
    Deploy zkEVM Permissionless Node: Deploys the zkEVM permissionless node.
    Deploy Observability Stack: Deploys the observability stack.
    Deploy ETH Load Balancer: Deploys the ETH load balancer.
    Apply Workload: Applies workload.
    Check Batch Verification: Checks that batches are being verified.

'''bash
      - name: Disable all deployment steps
        run: yq -Y --in-place 'with_entries(if .value | type == "boolean" then .value = false else . end)' params.yml

      - name: Deploy L1
        run: |
          yq -Y --in-place '.deploy_l1 = true' params.yml
          kurtosis run --enclave cdk-v1 --args-file params.yml .
          yq -Y --in-place '.deploy_l1 = false' params.yml # reset

      - name: Deploy zkEVM contracts on L1
        run: |
          yq -Y --in-place '.deploy_zkevm_contracts_on_l1 = true' params.yml
          kurtosis run --enclave cdk-v1 --args-file params.yml --image-download always .
          yq -Y --in-place '.deploy_zkevm_contracts_on_l1 = false' params.yml # reset

      - name: Deploy zkEVM node and cdk peripheral databases
        run: |
          yq -Y --in-place '.deploy_databases = true' params.yml
          kurtosis run --enclave cdk-v1 --args-file params.yml .
          yq -Y --in-place '.deploy_databases = false' params.yml # reset

      - name: Deploy CDK central environment
        run: |
          yq -Y --in-place '.deploy_cdk_central_environment = true' params.yml
          kurtosis run --enclave cdk-v1 --args-file params.yml .
          yq -Y --in-place '.deploy_cdk_central_environment = false' params.yml # reset

      - name: Deploy CDK bridge infrastructure
        run: |
          yq -Y --in-place '.deploy_cdk_bridge_infra = true' params.yml
          kurtosis run --enclave cdk-v1 --args-file params.yml .
          yq -Y --in-place '.deploy_cdk_bridge_infra = false' params.yml # reset

      - name: Deploy zkEVM permissionless node
        run: |
          yq -Y --in-place '.deploy_zkevm_permissionless_node = true' params.yml
          kurtosis run --enclave cdk-v1 --args-file params.yml .
          yq -Y --in-place '.deploy_zkevm_permissionless_node = false' params.yml # reset

      - name: Deploy observability stack
        run: |
          yq -Y --in-place '.deploy_observability = true' params.yml
          kurtosis run --enclave cdk-v1 --args-file params.yml .
          yq -Y --in-place '.deploy_observability = false' params.yml # reset

      - name: Deploy ETH load balancer
        run: |
          yq -Y --in-place '.deploy_blutgang = true' params.yml
          kurtosis run --enclave cdk-v1 --args-file params.yml .
          yq -Y --in-place '.deploy_blutgang = false' params.yml # reset

      - name: Apply workload
        run: |
          yq -Y --in-place '.apply_workload = true' params.yml
          kurtosis run --enclave cdk-v1 --args-file params.yml .
          yq -Y --in-place '.apply_workload = false' params.yml # reset

      - name: Check that batches are being verified
        working-directory: ./scripts
        run: ./batch_verification_monitor.sh ${{ env.BATCH_VERIFICATION_MONITOR_TARGET }} ${{ env.BATCH_VERIFICATION_MONITOR_TIMEOUT }}

```

This job deploys the Kurtosis CDK environment without specifying any parameter file.

'''bash
  configless:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
```

    Kurtosis CLI: Installs the Kurtosis CLI.
    Foundry: Installs Foundry for Ethereum development.

```      - name: Install kurtosis
        run: |
          echo "deb [trusted=yes] https://apt.fury.io/kurtosis-tech/ /" | sudo tee /etc/apt/sources.list.d/kurtosis.list
          sudo apt update
          sudo apt install kurtosis-cli=${{ env.KURTOSIS_VERSION }}
          kurtosis analytics disable

      - name: Install foundry
        uses: foundry-rs/foundry-toolchain@v1
```

    Deploy Kurtosis CDK Package: Deploys the CDK package.
    Check Batch Verification: Checks that batches are being verified.

```bash
      - name: Deploy Kurtosis CDK package
        run: kurtosis run --enclave cdk-v1 --image-download always .

      - name: Check that batches are being verified
        working-directory: ./scripts
        run: ./batch_verification_monitor.sh ${{ env.BATCH_VERIFICATION_MONITOR_TARGET }} ${{ env.BATCH_VERIFICATION_MONITOR_TIMEOUT }}
```

This detailed documentation provides a clear understanding of the deploy.yml file, outlining each step, job, and the purpose of the workflow. This should help in comprehending the deployment process of the Kurtosis CDK environment using GitHub Actions.


