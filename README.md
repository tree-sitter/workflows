# Reusable workflows

This repository contains reusable and reference workflows for parsers.

## Packaging workflows

These workflows can be used to publish parser packages to registries.<br>
Remove the jobs you don't need and create an environment for each job you do.

```yaml
name: Publish package

on:
  push:
    tags: ["*"]

concurrency:
  group: ${{github.workflow}}-${{github.ref}}
  cancel-in-progress: true

jobs:
  npm:
    uses: tree-sitter/workflows/.github/workflows/package-npm.yml@main
    secrets:
      NODE_AUTH_TOKEN: ${{secrets.NPM_TOKEN}}
  crates:
    uses: tree-sitter/workflows/.github/workflows/package-crates.yml@main
    secrets:
      CARGO_REGISTRY_TOKEN: ${{secrets.CARGO_TOKEN}}
  pypi:
    uses: tree-sitter/workflows/.github/workflows/package-pypi.yml@main
    secrets:
      PYPI_API_TOKEN: ${{secrets.PYPI_TOKEN}}
```

### npm options

```yaml
inputs:
  package-name:
    description: The name of the package
    default: ${{github.event.repository.name}}
    type: string
  environment-name:
    description: The name of the environment
    default: npm
    type: string
  node-version:
    description: The NodeJS version
    default: ${{vars.NODE_VERSION || 'latest'}}
    type: string
secrets:
  NODE_AUTH_TOKEN:
    description: An authentication token for npm
    required: true
```

### crates options

```yaml
inputs:
  package-name:
    description: The name of the package
    default: ${{github.event.repository.name}}
    type: string
  environment-name:
    description: The name of the environment
    default: crates
    type: string
  rust-toolchain:
    description: The Rust toolchain
    default: ${{vars.RUST_TOOLCHAIN || 'stable'}}
    type: string
secrets:
  CARGO_REGISTRY_TOKEN:
    description: An authentication token for crates.io
    required: true
```

### pypi options

```yaml
inputs:
  package-name:
    description: The name of the package
    default: ${{github.event.repository.name}}
    type: string
  environment-name:
    description: The name of the environment
    default: pypi
    type: string
  python-version:
    description: The Python version
    default: ${{vars.PYTHON_VERSION || '3.x'}}
    type: string
secrets:
  PYPI_API_TOKEN:
    description: An authentication token for pypi
    required: true
```

## Reference workflows

These workflows make use of our parser actions.

### CI workflow

This workflow uses all the following actions:

- [setup](https://github.com/tree-sitter/setup-action)
- [parser-test](https://github.com/tree-sitter/parser-test-action)
- [parse](https://github.com/tree-sitter/parse-action)
- [fuzz](https://github.com/tree-sitter/fuzz-action)

```yaml
name: CI

on:
  push:
    branches: [master]
    paths:
      - grammar.js
      - src/**
      - test/**
      - bindings/**
      - binding.gyp
  pull_request:
    paths:
      - grammar.js
      - src/**
      - test/**
      - bindings/**
      - binding.gyp

concurrency:
  group: ${{github.workflow}}-${{github.ref}}
  cancel-in-progress: true

jobs:
  test:
    name: Test parser
    runs-on: ${{matrix.os}}
    strategy:
      fail-fast: false
      matrix:
        os: [ubuntu-latest, windows-latest, macos-14]
    steps:
      - name: Checkout repository
        uses: actions/checkout@v4
      - name: Set up tree-sitter
        uses: tree-sitter/setup-action/cli@v1
      - name: Run parser and binding tests
        uses: tree-sitter/parser-test-action@v2
        with:
          test-rust: ${{runner.os == 'Linux'}}
          test-swift: ${{runner.os == 'macOS'}}
      - name: Parse sample files
        uses: tree-sitter/parse-action@v4
        id: parse-files
        with:
          files: examples/**
      - name: Upload failures artifact
        uses: actions/upload-artifact@v4
        if: "!cancelled() && steps.parse-files.outcome == 'failure'"
        with:
          name: failures-${{runner.os}}
          path: ${{steps.parse-files.outputs.failures}}
  fuzz:
    name: Fuzz scanner
    runs-on: ubuntu-latest
    steps:
      - name: Checkout repository
        uses: actions/checkout@v4
        with:
          fetch-depth: 2
      - name: Check for scanner changes
        id: scanner-check
        run: |-
          if git diff --quiet HEAD^ -- src/scanner.c; then
            printf 'changed=false\n' >> "$GITHUB_OUTPUT"
          else
            printf 'changed=true\n' >> "$GITHUB_OUTPUT"
          fi
      - name: Run the fuzzer
        uses: tree-sitter/fuzz-action@v4
        if: steps.scanner-check.outputs.changed == 'true'
```

### Dependency update workflow

This workflow uses the [parser-update](https://github.com/tree-sitter/parser-update-action) action.

```yaml
name: Update dependencies

on:
  schedule:
    - cron: "0 0 * * 0" # every week
  workflow_dispatch:

permissions:
  contents: write
  pull-requests: write

jobs:
  update:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout repository
        uses: actions/checkout@v4
      - name: Set up NodeJS
        uses: actions/setup-node@v4
        with:
          cache: npm
      - name: Update dependencies
        uses: tree-sitter/parser-update-action@v1.1
        with:
          parent-name: c
          language-name: cpp
```
