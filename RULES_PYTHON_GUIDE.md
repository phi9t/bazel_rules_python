# Complete Guide to Using rules_python in Bazel

This guide covers how to use rules_python for building, running, and packaging Python binaries and libraries in Bazel.

## Table of Contents
1. [Setup and Installation](#setup-and-installation)
2. [Basic Python Rules](#basic-python-rules)
3. [Managing Dependencies](#managing-dependencies)
4. [Building Python Libraries](#building-python-libraries)
5. [Building Python Binaries](#building-python-binaries)
6. [Running Python Tests](#running-python-tests)
7. [Packaging and Distribution](#packaging-and-distribution)
8. [Advanced Patterns](#advanced-patterns)
9. [Troubleshooting](#troubleshooting)

## Setup and Installation

### Using bzlmod (Recommended)

Add to your `MODULE.bazel` file:

```python
bazel_dep(name = "rules_python", version = "0.36.0")

# Configure Python toolchain
python = use_extension("@rules_python//python/extensions:python.bzl", "python")
python.toolchain(python_version = "3.11")
use_repo(python, "python_3_11")

# Register the toolchain
register_toolchains("@python_3_11//:all")
```

### Using WORKSPACE (Legacy)

Add to your `WORKSPACE` file:

```python
load("@bazel_tools//tools/build_defs/repo:http.bzl", "http_archive")

http_archive(
    name = "rules_python",
    sha256 = "...",  # Get latest from releases page
    strip_prefix = "rules_python-0.36.0",
    url = "https://github.com/bazelbuild/rules_python/releases/download/0.36.0/rules_python-0.36.0.tar.gz",
)

load("@rules_python//python:repositories.bzl", "py_repositories", "python_register_toolchains")

py_repositories()

python_register_toolchains(
    name = "python_3_11",
    python_version = "3.11",
)
```

## Basic Python Rules

### Core Rules Overview

rules_python provides four main rule types:

- **`py_library`**: Collections of Python source files and their dependencies
- **`py_binary`**: Executable Python programs  
- **`py_test`**: Python test targets
- **`py_proto_library`**: Python bindings for protocol buffers

## Managing Dependencies

### PyPI Dependencies with bzlmod

Create a `requirements.txt` file:

```text
requests==2.31.0
numpy>=1.24.0
pytest==7.4.0
```

Add to your `MODULE.bazel`:

```python
pip = use_extension("@rules_python//python/extensions:pip.bzl", "pip")

pip.parse(
    hub_name = "pip",
    python_version = "3.11",
    requirements_lock = "//:requirements_lock.txt",
)
use_repo(pip, "pip")
```

Generate the lock file:

```bash
bazel run //:requirements.update
```

### Using Dependencies in BUILD Files

```python
load("@pip//:requirements.bzl", "requirement")

py_library(
    name = "my_lib",
    srcs = ["my_lib.py"],
    deps = [
        requirement("requests"),
        requirement("numpy"),
    ],
)
```

## Building Python Libraries

### Simple Library

```python
# BUILD.bazel
py_library(
    name = "math_utils",
    srcs = ["math_utils.py"],
    visibility = ["//visibility:public"],
)
```

```python
# math_utils.py
def add(a, b):
    """Add two numbers."""
    return a + b

def multiply(a, b):
    """Multiply two numbers."""
    return a * b
```

### Library with Dependencies

```python
py_library(
    name = "web_client",
    srcs = ["web_client.py"],
    deps = [
        ":math_utils",  # Local dependency
        requirement("requests"),  # PyPI dependency
        "//common:logging_utils",  # Cross-package dependency
    ],
    visibility = ["//visibility:public"],
)
```

### Library with Data Files

```python
py_library(
    name = "config_loader",
    srcs = ["config_loader.py"],
    data = [
        "config.yaml",
        "templates/default.json",
    ],
    deps = [
        requirement("pyyaml"),
    ],
)
```

Access data files in Python:

```python
# config_loader.py
from bazel_tools.tools.python.runfiles import runfiles
import os

def load_config():
    r = runfiles.Create()
    config_path = r.Rlocation("myproject/config.yaml")
    with open(config_path) as f:
        return f.read()
```

## Building Python Binaries

### Simple Binary

```python
# BUILD.bazel
py_binary(
    name = "hello_world",
    srcs = ["hello_world.py"],
    main = "hello_world.py",
)
```

```python
# hello_world.py
def main():
    print("Hello, World!")

if __name__ == "__main__":
    main()
```

Run the binary:

```bash
bazel run //:hello_world
```

### Binary with Dependencies

```python
py_binary(
    name = "web_scraper",
    srcs = ["web_scraper.py"],
    main = "web_scraper.py",
    deps = [
        ":web_client",
        requirement("beautifulsoup4"),
    ],
    args = ["--url", "https://example.com"],
)
```

### Binary with Entry Points

```python
py_binary(
    name = "cli_tool",
    srcs = ["cli_tool.py"],
    main = "cli_tool.py",
    deps = [
        requirement("click"),
    ],
    # Pass arguments to the binary
    args = ["--help"],
)
```

## Running Python Tests

### Simple Test

```python
# BUILD.bazel
py_test(
    name = "math_utils_test",
    srcs = ["math_utils_test.py"],
    deps = [
        ":math_utils",
    ],
)
```

```python
# math_utils_test.py
import unittest
from math_utils import add, multiply

class TestMathUtils(unittest.TestCase):
    def test_add(self):
        self.assertEqual(add(2, 3), 5)
    
    def test_multiply(self):
        self.assertEqual(multiply(3, 4), 12)

if __name__ == "__main__":
    unittest.main()
```

Run the test:

```bash
bazel test //:math_utils_test
```

### Test with External Dependencies

```python
py_test(
    name = "web_client_test",
    srcs = ["web_client_test.py"],
    deps = [
        ":web_client",
        requirement("requests"),
        requirement("pytest"),
        requirement("responses"),  # For mocking HTTP requests
    ],
    # Test-specific data files
    data = ["testdata/sample_response.json"],
)
```

### Parameterized Tests

```python
py_test(
    name = "integration_test",
    srcs = ["integration_test.py"],
    deps = [
        ":my_app",
        requirement("pytest"),
        requirement("pytest-xdist"),  # For parallel test execution
    ],
    # Run tests in parallel
    args = ["-n", "auto"],
)
```

## Packaging and Distribution

### Creating Wheels

```python
load("@rules_python//python:packaging.bzl", "py_wheel")

py_wheel(
    name = "my_package_wheel",
    distribution = "my-package",
    version = "1.0.0",
    author = "Your Name",
    author_email = "your.email@example.com",
    description = "My awesome Python package",
    homepage = "https://github.com/yourorg/my-package",
    license = "MIT",
    classifiers = [
        "Development Status :: 4 - Beta",
        "Intended Audience :: Developers",
        "License :: OSI Approved :: MIT License",
        "Programming Language :: Python :: 3.11",
    ],
    deps = [
        ":my_package",
    ],
)
```

Build the wheel:

```bash
bazel build //:my_package_wheel
```

### Publishing to PyPI

```python
load("@rules_python//python:packaging.bzl", "py_wheel", "py_package")

py_package(
    name = "my_package_pkg",
    packages = ["my_package"],
    deps = [":my_package"],
)

py_wheel(
    name = "my_package_wheel",
    distribution = "my-package",
    version = "1.0.0",
    deps = [":my_package_pkg"],
)

# Publishing target
sh_binary(
    name = "publish",
    srcs = ["publish.sh"],
    data = [":my_package_wheel"],
)
```

```bash
#!/bin/bash
# publish.sh
twine upload bazel-bin/my_package_wheel.whl
```

## Advanced Patterns

### Multi-Python Version Support

```python
# MODULE.bazel
python = use_extension("@rules_python//python/extensions:python.bzl", "python")
python.toolchain(python_version = "3.9")
python.toolchain(python_version = "3.10")
python.toolchain(python_version = "3.11")

use_repo(python, "python_3_9", "python_3_10", "python_3_11")

register_toolchains(
    "@python_3_9//:all",
    "@python_3_10//:all", 
    "@python_3_11//:all",
)
```

```python
# BUILD.bazel
py_test(
    name = "compatibility_test_py39",
    srcs = ["test_compatibility.py"],
    main = "test_compatibility.py",
    python_version = "PY3",
    deps = [":my_library"],
)
```

### Custom Toolchains

```python
# For using a specific Python interpreter
load("@rules_python//python:repositories.bzl", "python_register_toolchains")

python_register_toolchains(
    name = "python_3_11_custom",
    base_url = "https://github.com/indygreg/python-build-standalone/releases/download",
    python_version = "3.11.7",
    tool_versions = {
        "3.11.7": {
            "url": "20231002/cpython-3.11.7+20231002-{platform}.tar.gz",
            "sha256": {
                "x86_64-apple-darwin": "...",
                "x86_64-unknown-linux-gnu": "...",
            },
        },
    },
)
```

### Protocol Buffers

```python
load("@rules_python//python:proto.bzl", "py_proto_library")

proto_library(
    name = "my_proto",
    srcs = ["my_proto.proto"],
)

py_proto_library(
    name = "my_proto_py",
    deps = [":my_proto"],
)

py_library(
    name = "proto_handler",
    srcs = ["proto_handler.py"],
    deps = [":my_proto_py"],
)
```

### Virtual Environments

```python
load("@rules_python//python:repositories.bzl", "py_repositories")
load("@rules_python//python:pip.bzl", "pip_parse")

pip_parse(
    name = "pip_deps",
    requirements_lock = "//:requirements.txt",
    python_interpreter_target = "@python_3_11_host//:python",
)

load("@pip_deps//:requirements.bzl", "install_deps")
install_deps()
```

### Cross-Platform Builds

```python
# Different requirements per platform
pip_parse(
    name = "pip_deps",
    requirements_darwin = "//:requirements_darwin.txt",
    requirements_linux = "//:requirements_linux.txt", 
    requirements_windows = "//:requirements_windows.txt",
    python_interpreter_target = "@python_3_11_host//:python",
)
```

## Troubleshooting

### Common Issues

**Import Errors**
```bash
# Check if all dependencies are declared
bazel query "deps(//:my_target)" --output=graph

# Verify Python path
bazel run //:my_binary --run_under="python -c 'import sys; print(sys.path)'"
```

**Dependency Conflicts**
```bash
# Check for version conflicts in requirements
bazel run @pip//:requirements.update

# Use pip-tools to resolve conflicts
pip-compile requirements.in
```

**Test Failures**
```bash
# Run tests with verbose output
bazel test //:my_test --test_output=all

# Run specific test method
bazel test //:my_test --test_filter="TestClass.test_method"
```

**Performance Issues**
```bash
# Use remote caching
bazel build //:my_target --remote_cache=...

# Parallel test execution
bazel test //... --jobs=8 --local_test_jobs=4
```

### Debugging Tips

1. **Use `--verbose_failures`** to see detailed error messages
2. **Check runfiles** when dealing with data dependencies
3. **Verify Python interpreter** with `bazel info python-bin`
4. **Use query commands** to understand dependency graphs
5. **Enable debug logging** with `--subcommands` flag

### Best Practices

1. **Pin dependency versions** in requirements files
2. **Use virtual environments** for development  
3. **Organize code** into logical packages and libraries
4. **Write comprehensive tests** with good coverage
5. **Use consistent naming** conventions across targets
6. **Document public APIs** and complex build configurations
7. **Leverage Bazel's caching** by keeping targets focused and granular

This guide covers the essential patterns for using rules_python effectively in Bazel projects. For more advanced use cases, refer to the official documentation and examples in the rules_python repository.