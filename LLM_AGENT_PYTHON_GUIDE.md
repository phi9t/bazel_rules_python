# LLM Agent Guide: Modern Bazel with rules_python (bzlmod)

This guide is optimized for LLM agents to quickly reference and execute Python-related tasks using modern Bazel with bzlmod (MODULE.bazel). **All examples use bzlmod - no legacy WORKSPACE patterns.**

## Quick Task Reference

### Task: Setup New Python Project
**When**: User asks to set up/initialize Python project with Bazel
**Required Files**: `MODULE.bazel`, `BUILD.bazel`, `.bazelrc`
**Commands**:
```bash
# 1. Create MODULE.bazel with Python extension
# 2. Create root BUILD.bazel
# 3. Create .bazelrc with common flags
# 4. Initialize with: bazel build //...
```

### Task: Add PyPI Dependencies  
**When**: User wants to add external Python packages
**Commands**:
```bash
# 1. Create/update requirements.txt
# 2. Add pip.parse() to MODULE.bazel
# 3. Run: bazel run //:requirements.update
```
**Template**: See [PyPI Dependencies Template](#pypi-dependencies-template)

### Task: Create Python Library
**When**: User wants to create reusable Python code
**Template**:
```python
py_library(
    name = "library_name",
    srcs = ["source_file.py"],
    deps = [
        requirement("package_name"),  # PyPI deps
        "//path/to:other_lib",       # Internal deps
    ],
    visibility = ["//visibility:public"],
)
```

### Task: Create Python Binary
**When**: User wants executable Python program
**Template**:
```python
py_binary(
    name = "binary_name", 
    srcs = ["main.py"],
    main = "main.py",
    deps = [":library_name"],
)
```
**Run command**: `bazel run //:binary_name`

### Task: Create Python Test
**When**: User wants to add tests
**Template**:
```python
py_test(
    name = "test_name",
    srcs = ["test_file.py"],
    deps = [
        ":library_under_test",
        requirement("pytest"),
    ],
)
```
**Run command**: `bazel test //:test_name`

### Task: Build Wheel Package
**When**: User wants to create distributable package
**Template**: See [Wheel Template](#wheel-template)
**Build command**: `bazel build //:package_wheel`

## File Templates

### MODULE.bazel Template (Modern bzlmod)
```python
module(
    name = "my_python_project",
    version = "0.1.0",
)

# Core dependencies
bazel_dep(name = "rules_python", version = "0.36.0")
bazel_dep(name = "bazel_skylib", version = "1.8.1")

# Python toolchain setup
python = use_extension("@rules_python//python/extensions:python.bzl", "python")
python.toolchain(
    python_version = "3.11",
    is_default = True,
)
use_repo(python, "python_3_11")

# Multi-version support (optional)
python.toolchain(python_version = "3.10") 
python.toolchain(python_version = "3.12")
use_repo(python, "python_3_10", "python_3_12")

# Register all toolchains
register_toolchains("@python_3_11//:all")

# PyPI dependencies extension
pip = use_extension("@rules_python//python/extensions:pip.bzl", "pip")

# Modern pip.parse with platform support
pip.parse(
    hub_name = "pip",
    python_version = "3.11",
    requirements_lock = "//:requirements_lock.txt",
    # Enable experimental features
    experimental_index_url = "https://pypi.org/simple",
    # Platform-specific requirements (optional)
    requirements_by_platform = {
        "//:requirements_darwin.txt": "osx_*",
        "//:requirements_linux.txt": "linux_*", 
        "//:requirements_windows.txt": "windows_*",
    },
)
use_repo(pip, "pip")
```

### .bazelrc Template (Modern flags)
```bash
# Modern Bazel flags for Python projects
common --enable_bzlmod
common --registry=https://bcr.bazel.build
common --registry=https://raw.githubusercontent.com/bazelbuild/bazel-central-registry/main/

# Python-specific optimizations
build --incompatible_default_to_explicit_init_py
build --python_top=@python_3_11//:python

# Performance improvements
build --experimental_reuse_sandbox_directories
test --test_output=errors
test --test_summary=short

# For development
common:dev --compilation_mode=dbg
common:dev --test_output=all
```

### PyPI Dependencies Template (Modern bzlmod)
1. Create `requirements.in` (input file):
```text
# Production dependencies
requests>=2.31.0
numpy>=1.24.0
fastapi>=0.104.0

# Development dependencies  
pytest>=7.4.0
black>=23.0.0
mypy>=1.5.0
```

2. Add to MODULE.bazel with modern pip extension:
```python
pip = use_extension("@rules_python//python/extensions:pip.bzl", "pip")

pip.parse(
    hub_name = "pip",
    python_version = "3.11",
    requirements_lock = "//:requirements_lock.txt",
    # Modern features
    experimental_index_url = "https://pypi.org/simple",
    # Support for --find-links and private indexes
    extra_pip_args = ["--trusted-host", "private.pypi.org"],
)
use_repo(pip, "pip")
```

3. Modern compile_pip_requirements in BUILD.bazel:
```python
load("@rules_python//python:pip.bzl", "compile_pip_requirements")

compile_pip_requirements(
    name = "requirements",
    extra_args = [
        "--allow-unsafe", 
        "--resolver=backtracking",
        "--upgrade",
    ],
    requirements_in = "requirements.in",
    requirements_txt = "requirements_lock.txt",
    visibility = ["//visibility:public"],
)

# For cross-platform projects
compile_pip_requirements(
    name = "requirements_linux",
    extra_args = ["--platform", "linux_x86_64"],
    requirements_in = "requirements.in", 
    requirements_txt = "requirements_linux.txt",
)
```

### Modern Wheel Template
```python
load("@rules_python//python:packaging.bzl", "py_wheel")

py_wheel(
    name = "package_wheel",
    # Metadata
    distribution = "my-awesome-package",
    version = "1.0.0",
    author = "Author Name",
    author_email = "author@example.com",
    description = "Modern Python package built with Bazel",
    homepage = "https://github.com/org/repo",
    license = "Apache-2.0",
    
    # Modern Python classifiers
    classifiers = [
        "Development Status :: 4 - Beta",
        "Intended Audience :: Developers", 
        "License :: OSI Approved :: Apache Software License",
        "Operating System :: OS Independent",
        "Programming Language :: Python :: 3",
        "Programming Language :: Python :: 3.10",
        "Programming Language :: Python :: 3.11",
        "Programming Language :: Python :: 3.12",
        "Topic :: Software Development :: Libraries",
    ],
    
    # Dependencies and entry points
    deps = [":package_lib"],
    entry_points = {
        "console_scripts": ["my-tool=my_package.cli:main"],
    },
    
    # Include additional files
    extra_distinfo_files = {
        "//:README.md": "README.md",
        "//:LICENSE": "LICENSE",
    },
)
```

## Modern Command Patterns

### Building (bzlmod)
```bash
# Build specific target
bazel build //:target_name

# Build all targets (modern filter)
bazel build //... --build_tag_filters=-integration-test

# Build with specific toolchain
bazel build //:target --extra_toolchains=@python_3_12//:all

# Build for multiple platforms
bazel build //:target --platforms=@platforms//os:linux,@platforms//os:osx

# Clean and build
bazel clean && bazel build //...
```

### Modern Dependency Commands
```bash
# Update requirements with modern resolver
bazel run //:requirements.update

# Update platform-specific requirements
bazel run //:requirements_linux.update

# Query module graph (bzlmod)
bazel mod graph

# Show dependency tree
bazel mod deps

# Explain module resolution
bazel mod explain rules_python
```

### Testing
```bash
# Run specific test
bazel test //:test_name

# Run all tests
bazel test //... --test_tag_filters=-integration-test

# Run with coverage
bazel coverage //:test_name

# Run specific test method
bazel test //:test_name --test_filter="TestClass.test_method"
```

### Running
```bash
# Run binary
bazel run //:binary_name

# Run with arguments
bazel run //:binary_name -- --arg1 value1

# Run in specific environment
bazel run //:binary_name --run_under="env VAR=value"
```

### Advanced Querying (bzlmod)
```bash
# Check target dependencies
bazel query "deps(//:target_name)"

# Find reverse dependencies  
bazel query "rdeps(//..., //:target_name)"

# List all Python targets
bazel query "kind('py_.*', //...)"

# Show external dependencies (bzlmod)
bazel query --output=build "//external:*" | grep -E "pip|python"

# Analyze dependency conflicts
bazel cquery "deps(//:target)" --output=graph
```

## Decision Trees for Agents

### What rule type to use?
```
Is it executable? 
├─ YES → py_binary
└─ NO → Is it for testing?
    ├─ YES → py_test
    └─ NO → py_library
```

### How to handle dependencies?
```
External package (from PyPI)?
├─ YES → Use requirement("package_name")
└─ NO → Local dependency?
    ├─ Same package → ":target_name" 
    └─ Different package → "//path/to/package:target_name"
```

### Project structure recommendations?
```
Simple project:
├── MODULE.bazel
├── BUILD.bazel  
├── requirements.txt
├── requirements_lock.txt
└── src/
    ├── main.py
    ├── lib.py
    └── test_lib.py

Complex project:
├── MODULE.bazel
├── BUILD.bazel
├── requirements.txt
├── requirements_lock.txt
├── src/
│   ├── BUILD.bazel
│   ├── package1/
│   │   ├── BUILD.bazel
│   │   └── module.py
│   └── package2/
│       ├── BUILD.bazel  
│       └── module.py
└── tests/
    ├── BUILD.bazel
    └── test_*.py
```

## Common Error Solutions

### Import Errors
**Symptom**: `ModuleNotFoundError: No module named 'X'`
**Solutions**:
1. Check if dependency is declared in `deps`
2. Verify requirement() spelling
3. Run `bazel run //:requirements.update` if PyPI package

### Missing Requirements
**Symptom**: Build fails with missing package
**Solution**: Add to requirements.txt and run `bazel run //:requirements.update`

### Wrong Python Version
**Symptom**: Syntax or compatibility errors
**Solution**: Check python_version in MODULE.bazel matches requirements

### Test Discovery Issues
**Symptom**: Tests not found or running
**Solutions**:
1. Ensure test files match pattern (test_*.py or *_test.py)
2. Add `if __name__ == "__main__": unittest.main()` to test files
3. Check test deps include testing framework

## Agent Task Execution Steps

### For "Create Python Library" task:
1. Ask user for library name and source files
2. Create py_library rule in BUILD.bazel
3. If external deps needed, update requirements.txt
4. Run bazel build to verify
5. Suggest adding tests

### For "Add PyPI Package" task:
1. Add package to requirements.txt
2. Run `bazel run //:requirements.update`
3. Add requirement("package") to relevant targets
4. Test with `bazel build //...`

### For "Create Executable" task:
1. Identify main Python file
2. Create py_binary rule
3. Set main parameter correctly
4. Add necessary deps
5. Test with `bazel run`
6. Suggest adding argument parsing if needed

### For "Package for Distribution" task:
1. Create py_wheel rule
2. Set metadata (name, version, author, etc.)
3. Include all necessary deps
4. Build with `bazel build //:wheel_target`
5. Explain distribution process

## Validation Commands

After any change, agents should run:
```bash
# Verify build works
bazel build //...

# Verify tests pass  
bazel test //...

# Check for issues
bazel query "kind(py_.*, //...)" # List all Python targets
```

## Quick Reference: Common Attributes

### py_library
- `name`: Target name
- `srcs`: Python source files
- `deps`: Dependencies  
- `data`: Runtime data files
- `visibility`: Who can depend on this

### py_binary
- `name`: Target name
- `srcs`: Python source files  
- `main`: Entry point file
- `deps`: Dependencies
- `args`: Default arguments

### py_test
- `name`: Target name
- `srcs`: Test source files
- `deps`: Dependencies (including test lib)
- `data`: Test data files
- `size`: "small", "medium", "large"

This guide prioritizes quick lookup and task execution patterns that LLM agents can easily follow.