# Aztec Benchmark
[![npm version](https://badge.fury.io/js/%40aztec-foundation%2Faztec-benchmark.svg)](https://www.npmjs.com/package/@aztec-foundation/aztec-benchmark)

**CLI tool and reusable CI workflows for running Aztec contract benchmarks.**

Use the CLI to execute benchmark files written in TypeScript. For CI integration, this repository provides **reusable GitHub workflows** that handle the full benchmark-and-compare cycle — including environment setup, baseline management, and PR commenting — so consumer repos can integrate with a single `uses:` line.

## Table of Contents

- [Installation](#installation)
- [CLI Usage](#cli-usage)
  - [Configuration (`Nargo.toml`)](#configuration-nargotoml)
  - [Options](#options)
  - [Examples](#examples)
- [Writing Benchmarks](#writing-benchmarks)
- [Benchmark Output](#benchmark-output)
- [Reusable Workflows](#reusable-workflows)
  - [Pinning a version](#pinning-a-version)
  - [Security notes](#security-notes)
  - [PR Benchmark (`pr-benchmark.yml`)](#pr-benchmark-pr-benchmarkyml)
  - [Update Baseline (`update-baseline.yml`)](#update-baseline-update-baselineyml)
  - [Block duration](#block-duration)
  - [How Baselines Work](#how-baselines-work)
- [Action Usage (Advanced)](#action-usage-advanced)
  - [Inputs](#inputs)
  - [Outputs](#outputs)

---

## Installation

```sh
yarn add --dev @aztec-foundation/aztec-benchmark
# or
npm install --save-dev @aztec-foundation/aztec-benchmark
```

Pick the release line that matches your Aztec packages:

| Your Aztec packages | aztec-benchmark |
|---|---|
| v6 (`@aztec-labs/*`) | `6.x` (currently `@rc`: `yarn add --dev @aztec-foundation/aztec-benchmark@rc`) |
| v5 (`@aztec/*`) | `5.0.1` |

`@aztec-labs/aztec.js` and `@aztec-labs/wallets` are peer dependencies. Any `6.x` release or `6.0.0` pre-release (nightly or rc) satisfies them. With npm 7+, if you haven't installed them yourself, npm installs them for you.

---

## CLI Usage

After installing, run the CLI using `npx aztec-benchmark`. By default, it looks for a `Nargo.toml` file in the current directory and runs benchmarks defined within it.

```sh
npx aztec-benchmark [options]
```

### Configuration (`Nargo.toml`)

Define which contracts have associated benchmark files in your `Nargo.toml` under the `[benchmark]` section:

```toml
[benchmark]
token = "benchmarks/token_contract.benchmark.ts"
another_contract = "path/to/another.benchmark.ts"
```

The paths to the `.benchmark.ts` files are relative to the `Nargo.toml` file.

### Options

- `-c, --contracts <names...>`: Specify which contracts (keys from the `[benchmark]` section) to run. If omitted, runs all defined benchmarks.
- `--config <path>`: Path to your `Nargo.toml` file (default: `./Nargo.toml`).
- `-o, --output-dir <path>`: Directory to save benchmark JSON reports (default: `./benchmarks`).
- `-s, --suffix <suffix>`: Optional suffix to append to report filenames (e.g., `_pr` results in `token_pr.benchmark.json`).
- `--skip-proving`: Skip proving transactions. Only measures gate counts and gas; proving time will be `0` in reports. When enabled, the `wallet` is not required in the benchmark context.

### Examples

Run all benchmarks defined in `./Nargo.toml`:
```sh
npx aztec-benchmark 
```

Run only the `token` benchmark:
```sh
npx aztec-benchmark --contracts token
```

Run `token` and `another_contract` benchmarks, saving reports with a suffix:
```sh
npx aztec-benchmark --contracts token another_contract --output-dir ./benchmark_results --suffix _v2
```

---

## Writing Benchmarks

Benchmarks are TypeScript classes extending `BenchmarkBase` from this package.
Each entry in the array returned by `getMethods` can either be a plain `ContractFunctionInteractionCallIntent` 
(in which case the benchmark name is auto-derived) or a `NamedBenchmarkedInteraction` object 
(which includes the `interaction` and a custom `name` for reporting).

### Fee Payment

By default, every benchmarked account must hold Fee Juice (FJ) to pay for transaction fees. If your accounts don't have pre-existing FJ (e.g. freshly-created accounts on sandbox), you can return a `feePaymentMethod` from `setup()` inside the `BenchmarkContext`. The profiler will pass it to every `send()` and `proveInteraction()` call automatically.

The sandbox ships with a canonical `SponsoredFPC` contract that has FJ and can sponsor fees for any account — making it the easiest way to get benchmarks running without bridging from L1.

```ts
import {
  Benchmark, // Alias for BenchmarkBase
  type BenchmarkContext,
  type NamedBenchmarkedInteraction
} from '@aztec-foundation/aztec-benchmark';
import type { PXE } from '@aztec-labs/pxe/server';
import type { Contract } from '@aztec-labs/aztec.js/contracts'; // Generic Contract type from Aztec.js
import type { AztecAddress } from '@aztec-labs/aztec.js/addresses';
import type { ContractFunctionInteractionCallIntent } from '@aztec-labs/aztec.js/authorization';
import type { FeePaymentMethod } from '@aztec-labs/aztec.js/fee';
import { createStore } from '@aztec-labs/kv-store/lmdb-v2';
import { createPXE, getPXEConfig } from '@aztec-labs/pxe/server';
import { createAztecNodeClient, waitForNode } from '@aztec-labs/aztec.js/node';
import { EmbeddedWallet } from '@aztec-labs/wallets/embedded';
import { registerInitialLocalNetworkAccountsInWallet } from '@aztec-labs/wallets/testing';
// import { YourSpecificContract } from '../artifacts/YourSpecificContract.js'; // Replace with your actual contract artifact

// 1. Define a specific context for your benchmark (optional but good practice)
interface MyBenchmarkContext extends BenchmarkContext {
  pxe: PXE;
  wallet: EmbeddedWallet;
  deployer: AztecAddress;
  contract: Contract; // Use the generic Contract type or your specific contract type
  feePaymentMethod?: FeePaymentMethod;
}

export default class MyContractBenchmark extends Benchmark {
  // Runs once before all benchmark methods.
  async setup(): Promise<MyBenchmarkContext> {
    console.log('Setting up benchmark environment...');

    const { NODE_URL = 'http://localhost:8080' } = process.env;
    const node = createAztecNodeClient(NODE_URL);
    await waitForNode(node);
    const l1Contracts = await node.getL1ContractAddresses();
    const config = getPXEConfig();
    const fullConfig = { ...config, l1Contracts };
    // IMPORTANT: true enables proof generation for the benchmark, set it to false when using --skip-proving
    fullConfig.proverEnabled = true;
    const pxeVersion = 2;
    const store = await createStore('pxe', pxeVersion, {
      dataDirectory: 'store',
      dataStoreMapSizeKb: 1e6,
    });

    const pxe: PXE = await createPXE(node, fullConfig, { store });
    // `EmbeddedWalletOptions` uses a unified `pxe` field for PXE config and dependency overrides.
    const wallet: EmbeddedWallet = await EmbeddedWallet.create(node, { pxe: fullConfig });
    const accounts: AztecAddress[] = await registerInitialLocalNetworkAccountsInWallet(wallet);
    const [deployer] = accounts;
    
    //  Deploy your contract (replace YourSpecificContract with your actual contract class).
    //  `DeployMethod.send()` now always returns `{ contract, receipt, instance }`.
    const { contract } = await YourSpecificContract
      .deploy(wallet, /* constructor args */)
      .send({ from: deployer });
    console.log('Contract deployed at:', contract.address.toString());

    // Optional: use SponsoredFPC so accounts don't need pre-existing Fee Juice.
    // The sandbox ships with a canonical SponsoredFPC pre-deployed at a deterministic address.
    //
    // import { SponsoredFeePaymentMethod } from '@aztec-labs/aztec.js/fee/testing';
    // import { SponsoredFPCContract } from '@aztec-labs/noir-contracts.js/SponsoredFPC';
    // import { getContractInstanceFromInstantiationParams } from '@aztec-labs/aztec.js/contracts';
    //
    // const instance = await getContractInstanceFromInstantiationParams(
    //   SponsoredFPCContract.artifact,
    //   { salt: new Fr(0n) },
    // );
    // await wallet.registerContract(instance, SponsoredFPCContract.artifact);
    // const feePaymentMethod = new SponsoredFeePaymentMethod(instance.address);

    return { pxe, wallet, deployer, contract /*, feePaymentMethod */ }; 
  }

  // Returns an array of interactions to benchmark. 
  getMethods(context: MyBenchmarkContext): Promise<Array<ContractFunctionInteractionCallIntent | NamedBenchmarkedInteraction>> {
    // Ensure context is available (it should be if setup ran correctly)
    if (!context || !context.contract) {
      // In a real scenario, setup() must initialize the context properly.
      // Throwing an error or returning an empty array might be appropriate here if setup failed.
      console.error("Benchmark context or contract not initialized in setup(). Skipping getMethods.");
      return [];
    }
    
    const { contract, deployer } = context;
    const recipient = deployer; // Example recipient

    // Replace `contract.methods.someMethodName` with actual methods from your contract.
    const interactionPlain = { caller: deployer, action: contract.methods.transfer(recipient, 100n) }
    const interactionNamed1 = { caller: deployer, action: contract.methods.someOtherMethod("test_value_1") };
    const interactionNamed2 = { caller: deployer, action: contract.methods.someOtherMethod("test_value_2") };

    return [
      // Example of a plain interaction - name will be auto-derived
      interactionPlain,
      // Example of a named interaction
      { interaction: interactionNamed1, name: "Some Other Method (value 1)" }, 
      // Another named interaction
      { interaction: interactionNamed2, name: "Some Other Method (value 2)" }, 
    ];
  }

  // Optional cleanup phase
  async teardown(context: MyBenchmarkContext): Promise<void> {
    console.log('Cleaning up benchmark environment...');
    if (context && context.pxe) { 
      await context.pxe.stop(); 
    }
  }
}
```

**Note:** Your benchmark code needs a valid Aztec project setup to interact with contracts.
Your `BenchmarkBase` implementation is responsible for constructing the `ContractFunctionInteractionCallIntent` objects.
If you provide a `NamedBenchmarkedInteraction` object, its `name` field will be used in reports. 
If you provide a plain `ContractFunctionInteractionCallIntent`, the tool will attempt to derive a name from the interaction (e.g., the method name).
If you return a `feePaymentMethod` in the `BenchmarkContext`, it is automatically passed to every transaction the profiler sends — no changes to `getMethods` are needed.

### Aztec's Usage Example

You can find how we use this tool for benchmarking our Aztec contracts in [`aztec-standards`](https://github.com/AztecProtocol/aztec-standards/tree/main/benchmarks).

---

## Benchmark Output

Your `BenchmarkBase` implementation is responsible for measuring and outputting performance data (e.g., as JSON). The comparison action uses this output.
Each entry in the output will be identified by the custom `name` you provided (if any) or the auto-derived name.

---

## Reusable Workflows

This repository ships two **reusable GitHub workflows** (`workflow_call`) that handle the full CI benchmark cycle. Consumer repos call them with a single `uses:` line, with no workflow YAML to copy and no artifact management to wire up by hand.

### Pinning a version

Reference the workflows by the full commit SHA of a release, with the tag as a comment. A tag can be moved, but a SHA always names the same code. To resolve a release tag to its commit:

```sh
gh api repos/AztecProtocol/aztec-benchmark/commits/v6.0.0-rc.1 --jq .sha
```

The examples below use `<commit-sha>` as a placeholder for that value.

### Security notes

- Trigger `pr-benchmark.yml` from `pull_request`, never `pull_request_target`. It checks out and runs the PR's code, including its benchmarks and dependency install scripts, in the same job that holds the token used to comment.
- The checkout does not persist the GitHub token (`persist-credentials: false`). If your dependency install needs authenticated git access, for example private git dependencies, authenticate that step separately.
- PRs from forks get a read-only token. The benchmarks still run, but the comment step then fails the job, so no baseline artifact is uploaded for the fork branch.

### PR Benchmark (`pr-benchmark.yml`)

Runs benchmarks on the PR head, downloads the baseline from the base branch, generates a comparison report, comments it on the PR (hiding any previous benchmark comments as outdated), and uploads the new results as a baseline artifact for the PR branch.

**Usage:**

```yaml
# .github/workflows/pr-checks.yml
name: PR Checks

on:
  pull_request:
    branches: [main]

jobs:
  benchmark:
    uses: AztecProtocol/aztec-benchmark/.github/workflows/pr-benchmark.yml@<commit-sha> # v6.0.0-rc.1
    permissions:
      contents: read
      pull-requests: write
      issues: write
      actions: read
```

**Inputs:**

| Input | Type | Default | Description |
|---|---|---|---|
| `runner` | `string` | `ubuntu-latest-m` | GitHub runner label |
| `timeout` | `number` | `120` | Job timeout in minutes |
| `bench-dir` | `string` | `./benchmarks` | Directory for benchmark files |
| `current-suffix` | `string` | `_latest` | Suffix for baseline benchmark files |
| `pr-suffix` | `string` | `_new` | Suffix for PR benchmark files |
| `baseline-workflow` | `string` | `update-baseline.yml` | Workflow file that produces baselines for `main` |
| `pr-workflow` | `string` | `pr-checks.yml` | Workflow file that produces baselines for other base branches |
| `circuit-details` | `boolean` | `false` | Include a per-circuit gate breakdown in the report |
| `block-duration-ms` | `string` | `""` | Local-network block duration in ms; see [Block duration](#block-duration) |

**With custom inputs:**

```yaml
jobs:
  benchmark:
    uses: AztecProtocol/aztec-benchmark/.github/workflows/pr-benchmark.yml@<commit-sha> # v6.0.0-rc.1
    permissions:
      contents: read
      pull-requests: write
      issues: write
      actions: read
    with:
      runner: ubuntu-latest-l
      timeout: 180
      bench-dir: ./my-benchmarks
      block-duration-ms: "6000"
```

### Update Baseline (`update-baseline.yml`)

Runs benchmarks on the current branch and uploads the results as a baseline artifact. Trigger it on pushes to your default branches, so that PR benchmarks have a baseline to compare against.

**Usage:**

```yaml
# .github/workflows/update-baseline.yml
name: Update Baseline

on:
  push:
    branches: [main]

jobs:
  update-baseline:
    uses: AztecProtocol/aztec-benchmark/.github/workflows/update-baseline.yml@<commit-sha> # v6.0.0-rc.1
    permissions:
      contents: read
      actions: write
```

**Inputs:**

| Input | Type | Default | Description |
|---|---|---|---|
| `runner` | `string` | `ubuntu-latest-m` | GitHub runner label |
| `timeout` | `number` | `120` | Job timeout in minutes |
| `bench-dir` | `string` | `./benchmarks` | Directory for benchmark files |
| `current-suffix` | `string` | `_latest` | Suffix for benchmark report files |
| `block-duration-ms` | `string` | `""` | Local-network block duration in ms; see [Block duration](#block-duration) |

### Block duration

`setup-aztec` starts a local network with Aztec's default 3-second blocks. Each transaction's DA gas cap is a share of the checkpoint's blob space, so more blocks per checkpoint means a lower cap:

| Block duration | Blocks per 72 s checkpoint | Per-tx DA gas cap |
|---|---|---|
| 3000 ms (default) | 21 | 55,836 |
| 6000 ms (mainnet) | 10 | 117,624 |

A benchmark that publishes a large contract class can exceed the default cap, and then fails with `Transaction consumes N DA gas but the network only admits transactions declaring up to 55836 DA gas`. Set the same value on both workflows:

```yaml
    with:
      block-duration-ms: "6000"
```

The value must be a positive integer number of milliseconds; anything else fails the job's first step. The node refuses to start with values it can't schedule, which is anything above about 33000 with the local network's 72 s slots. Leave the input empty to keep Aztec's default.

### How Baselines Work

The workflows use GitHub Actions artifacts to store and retrieve baseline benchmark results:

1. **`update-baseline.yml`** runs benchmarks with the `_latest` suffix and uploads the results as `benchmark-baseline-<branch>`.
2. **`pr-benchmark.yml`** runs benchmarks with the `_new` suffix on the PR head, then downloads the `benchmark-baseline-<base-branch>` artifact to get the `_latest` files. It compares `_latest` (baseline) with `_new` (PR) and comments a Markdown diff table on the PR.
3. Before posting the new comment, the workflow finds all previous benchmark comments on the PR (identified by a unique marker in the comment body) and hides them as **Outdated** via the GitHub GraphQL API, so the PR timeline stays clean.
4. After comparison, the PR workflow renames `_new` files to `_latest` and uploads them as `benchmark-baseline-<head-branch>`, so stacked PRs can also compare against each other.

Artifacts are retained for **90 days** by default.

---

## Action Usage (Advanced)

> **Note:** For most projects, the [reusable workflows](#reusable-workflows) above are the recommended approach. The action below is a lower-level building block for projects that need a custom CI setup.

This repository also includes a GitHub Action (defined in `action/action.yml`) that runs `aztec-benchmark` and compares results. It automatically finds benchmark reports (named with `_base` and `_latest` suffixes) and produces a Markdown comparison report.

### Inputs

- `threshold`: Regression threshold percentage (default: `2.5`).
- `output_markdown_path`: Path to save the generated Markdown comparison report (default: `benchmark-comparison.md`).

### Outputs

- `comparison_markdown`: The generated Markdown report content.
- `markdown_file_path`: Path to the saved Markdown file.

Refer to the `action/action.yml` file for the definitive inputs and description.
