# Bazel Build System Documentation

This repository uses [Bazel](https://bazel.build/) as its build system with [Bzlmod](https://bazel.build/external/overview#bzlmod).

## Quick Start

### Prerequisites

- Bazel 8.4.2 or later (automatically managed via `.bazelversion`)
- If you don't have Bazel installed, install [Bazelisk](https://github.com/bazelbuild/bazelisk) which will automatically use the correct version

### Building

```bash
# Build all targets
bazel build //...

# Build specific targets
bazel build //io/prometheus/client:metrics_proto
bazel build //io/prometheus/client:metrics_cc_proto
bazel build //io/prometheus/client:metrics_go_proto
bazel build //io/prometheus/client:metrics_java_proto
```

### Testing

```bash
# Run all tests
bazel test //...

# Run tests with detailed output
bazel test //... --test_output=all
```

## Repository Structure

- `MODULE.bazel` - Bazel module definition with dependencies
- `.bazelrc` - Bazel configuration options
- `.bazelversion` - Pinned Bazel version
- `.bazelci/presubmit.yml` - BazelCI configuration for continuous integration
- `BUILD.bazel` - Build definitions for each package

## Available Targets

### Protocol Buffer Definitions

- `//io/prometheus/client:metrics_proto` - Core proto_library
- `//io/prometheus/client:metrics_cc_proto` - C++ library
- `//io/prometheus/client:metrics_py_proto` - Python library  
- `//io/prometheus/client:metrics_java_proto` - Java library
- `//io/prometheus/client:metrics_go_proto` - Go library

## Usage

In your MODULE.bazel:

```starlark
bazel_dep(name = "prometheus_client_model", version = "0.6.1")
```

**C++ example:**
```starlark
cc_binary(
    name = "my_app",
    srcs = ["main.cc"],
    deps = [
        "@prometheus_client_model//io/prometheus/client:metrics_cc_proto",
    ],
)
```

**Go example:**
```starlark
go_binary(
    name = "my_app",
    srcs = ["main.go"],
    deps = [
        "@prometheus_client_model//io/prometheus/client:metrics_go_proto",
    ],
)
```

**Java example:**
```starlark
java_binary(
    name = "my_app",
    srcs = ["Main.java"],
    deps = [
        "@prometheus_client_model//io/prometheus/client:metrics_java_proto",
    ],
)
```

The build file content is already included in this repository, so no additional
`build_file_content` parameter is needed.

## Publishing to Bazel Central Registry (BCR)

This repository uses [publish-to-bcr](https://github.com/bazel-contrib/publish-to-bcr) to automate publishing to the Bazel Central Registry.

### Automated Publishing

The repository is configured with `.bcr/` directory and `.github/workflows/publish.yml` to automatically publish releases to BCR when a new version tag is pushed:

1. **Update the version** in `MODULE.bazel`
2. **Create and push a tag**:
   ```bash
   git tag v0.6.1
   git push origin v0.6.1
   ```
3. **The publish-to-bcr action** will automatically:
   - Create a fork of bazel-central-registry (if needed)
   - Generate the required BCR files (MODULE.bazel, source.json, metadata.json)
   - Create a pull request to BCR
   - Run automated validation checks

### Manual Publishing

If you need to publish manually, you can use the [publish-to-bcr CLI](https://github.com/bazel-contrib/publish-to-bcr):

```bash
# Install the CLI
npm install -g @bazel/publish-to-bcr

# Publish a release
publish-to-bcr --github=prometheus/client_model --tag=v0.6.1
```

The tool will guide you through the process and create a pull request to the Bazel Central Registry.

### BCR Requirements

- Repository must have a `MODULE.bazel` file at the root
- Version in `MODULE.bazel` must match the git tag (without the 'v' prefix)
- All Bazel targets must build successfully
- Tests must pass

## CI/CD

### GitHub Actions

The repository includes GitHub Actions workflows:
- `.github/workflows/bazel.yml` - Runs Bazel build and tests on push/PR
- Validates builds on Ubuntu and macOS

## Troubleshooting

### Clean Build

```bash
bazel clean
bazel build //...
```
