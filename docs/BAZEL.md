# Bazel Build System Documentation

This repository uses [Bazel](https://bazel.build/) as its build system with [Bzlmod](https://bazel.build/external/overview#bzlmod) for modern dependency management.

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
bazel build //io/prometheus/client:client_go_proto
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
- `//io/prometheus/client:client_go_proto` - Go library

### Aliases (for Envoy compatibility)

- `//io/prometheus/client:client_model` - Alias to metrics_proto
- `//io/prometheus/client:client_model_go_proto` - Alias to client_go_proto

## Integration with Envoy

This repository is designed for compatibility with [Envoy Proxy](https://github.com/envoyproxy/envoy).
The Bazel MODULE configuration uses `repo_name = "prometheus_metrics_model"` to match Envoy's 
expectations from `api/bazel/repositories.bzl`.

### Using with Envoy (Bzlmod)

In your MODULE.bazel:

```starlark
bazel_dep(name = "client_model", version = "0.6.1")
```

This provides the following targets that are compatible with Envoy's naming:

**Proto library (language-agnostic):**
- `@client_model//io/prometheus/client:client_model` - Alias to metrics_proto
- `@client_model//io/prometheus/client:metrics_proto` - Main proto_library

**Language-specific libraries:**
- `@client_model//io/prometheus/client:client_model_cc_proto` - C++ library (alias)
- `@client_model//io/prometheus/client:client_model_py_proto` - Python library (alias)
- `@client_model//io/prometheus/client:client_model_go_proto` - Go library (alias)
- `@client_model//io/prometheus/client:metrics_cc_proto` - C++ library (direct)
- `@client_model//io/prometheus/client:metrics_py_proto` - Python library (direct)
- `@client_model//io/prometheus/client:client_go_proto` - Go library (direct)

**Note:** When using with `@prometheus_metrics_model` repository name (internal to the module),
the targets remain the same but use the module's repo_name:
- `@prometheus_metrics_model//io/prometheus/client:client_model`
- etc.

### Example Usage

**C++ example:**
```starlark
cc_binary(
    name = "my_app",
    srcs = ["main.cc"],
    deps = [
        "@client_model//io/prometheus/client:metrics_cc_proto",
    ],
)
```

**Go example:**
```starlark
go_binary(
    name = "my_app",
    srcs = ["main.go"],
    deps = [
        "@client_model//io/prometheus/client:client_go_proto",
    ],
)
```

### Legacy WORKSPACE Usage

For projects still using WORKSPACE instead of MODULE.bazel:

```starlark
http_archive(
    name = "prometheus_metrics_model",
    sha256 = "...",
    strip_prefix = "client_model-0.6.1",
    urls = ["https://github.com/prometheus/client_model/archive/v0.6.1.tar.gz"],
)
```

The build file content is already included in this repository, so no additional
`build_file_content` parameter is needed.

## Publishing to Bazel Central Registry (BCR)

### Prerequisites for BCR Publishing

1. **Version Tagging**: Create a Git tag following semantic versioning (e.g., `v0.6.1`)
2. **MODULE.bazel**: Ensure the version in MODULE.bazel matches your release tag
3. **Release Archive**: The BCR will fetch the source archive from GitHub releases

### Steps to Publish

1. **Prepare the Release**
   ```bash
   # Update version in MODULE.bazel
   # Commit all changes
   git tag v0.6.1
   git push origin v0.6.1
   ```

2. **Create a Pull Request to BCR**
   
   Fork and clone the [Bazel Central Registry](https://github.com/bazelbuild/bazel-central-registry):
   
   ```bash
   git clone https://github.com/YOUR_USERNAME/bazel-central-registry.git
   cd bazel-central-registry
   ```

3. **Create Module Directory**
   
   ```bash
   mkdir -p modules/client_model/0.6.1
   ```

4. **Create Required Files**

   Create `modules/client_model/0.6.1/MODULE.bazel` - copy from your repository
   
   Create `modules/client_model/0.6.1/source.json`:
   ```json
   {
     "integrity": "sha256-...",
     "strip_prefix": "client_model-0.6.1",
     "url": "https://github.com/prometheus/client_model/archive/refs/tags/v0.6.1.tar.gz"
   }
   ```
   
   Generate the integrity hash:
   ```bash
   curl -L https://github.com/prometheus/client_model/archive/refs/tags/v0.6.1.tar.gz | \
     shasum -a 256 | awk '{print $1}'
   ```
   
   Then format as `sha256-<hash>` for the source.json file.

5. **Create metadata.json** (if this is the first version)
   
   Create `modules/client_model/metadata.json`:
   ```json
   {
     "homepage": "https://github.com/prometheus/client_model",
     "maintainers": [
       {
         "email": "dev@prometheus.io",
         "github": "prometheus",
         "name": "Prometheus Developers"
       }
     ],
     "versions": ["0.6.1"],
     "yanked_versions": {}
   }
   ```

6. **Submit Pull Request**
   
   ```bash
   git checkout -b add-client-model-0.6.1
   git add modules/client_model/
   git commit -m "Add client_model 0.6.1"
   git push origin add-client-model-0.6.1
   ```
   
   Create a pull request to the bazel-central-registry repository.

7. **Automated Validation**
   
   The BCR has automated checks that will:
   - Verify the module builds successfully
   - Check the source archive is valid
   - Validate the integrity hash
   - Run tests

### Automated Publishing with GitHub Actions

You can automate BCR publishing by creating a workflow that:
1. Detects new version tags
2. Generates the required BCR files
3. Creates a pull request to BCR

This is typically done manually for now, but automation can be added in the future.

## CI/CD

### GitHub Actions

The repository includes GitHub Actions workflows:
- `.github/workflows/bazel.yml` - Runs Bazel build and tests on push/PR
- Validates builds on Ubuntu and macOS
- Tests with Bzlmod enabled

### BazelCI

BazelCI configuration is available at `.bazelci/presubmit.yml` and tests:
- Ubuntu 22.04
- macOS  
- Windows
- Bzlmod compatibility

To run BazelCI locally:
```bash
# Install bazelci.py
curl -sSL https://raw.githubusercontent.com/bazelbuild/continuous-integration/master/buildkite/bazelci.py -o bazelci.py

# Run presubmit checks
python3 bazelci.py --print_tasks .bazelci/presubmit.yml
```

## Troubleshooting

### Clean Build

```bash
bazel clean
bazel build //...
```

### Clean Everything (including external dependencies)

```bash
bazel clean --expunge
```

### Update Dependencies

Dependencies are managed in `MODULE.bazel`. After updating:
```bash
bazel sync --only=<dependency_name>
```

## Additional Resources

- [Bazel Documentation](https://bazel.build/)
- [Bzlmod Guide](https://bazel.build/external/overview#bzlmod)
- [Bazel Central Registry](https://registry.bazel.build/)
- [rules_proto Documentation](https://github.com/bazelbuild/rules_proto)
- [rules_go Documentation](https://github.com/bazelbuild/rules_go)
