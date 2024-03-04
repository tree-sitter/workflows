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
    default: ${{vars.PYTHON_VERSION || 'stable'}}
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

- [parser-setup](https://github.com/tree-sitter/parser-setup-action)
- [parser-test](https://github.com/tree-sitter/parser-test-action)
  (includes [parse](https://github.com/tree-sitter/parse-action))
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
      - name: Set up the repo
        uses: tree-sitter/parser-setup-action@v1.1
      - name: Run tests
        uses: tree-sitter/parser-test-action@v1.1
        with:
          test-library: ${{runner.os == 'Linux'}}
          examples: examples/**
      - name: Check for scanner changes
        id: scanner-check
        shell: sh
        run: |-
          {
            test -f src/scanner.c && ! git diff --quiet HEAD^ -- "$_" \
              && printf 'changed=true\n' || printf 'changed=false\n'
          } >> "$GITHUB_OUTPUT"
      - name: Fuzz scanner
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
        id: update
        uses: tree-sitter/parser-update-action@v1.1
        with:
          parent-name: c
          language-name: cpp
```
