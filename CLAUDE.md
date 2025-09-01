# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

This is rules_python - the home of core Python rules for Bazel (`py_library`, `py_binary`, `py_test`, `py_proto_library`) and package installation rules for PyPI integration. The repository provides both WORKSPACE and bzlmod support, with bzlmod being the preferred modern approach.

## Essential Commands

### Building and Testing
```bash
# Build all targets (exclude integration tests)
bazel build //... --build_tag_filters=-integration-test

# Run all tests (exclude integration tests) 
bazel test //... --test_tag_filters=-integration-test

# Run integration tests specifically
bazel test //... --test_tag_filters=integration-test

# Run a single test
bazel test //path/to:specific_test

# Build documentation
bazel build //docs:docs
```

### Code Quality and Formatting
```bash
# Format Starlark files with buildifier
buildifier --lint=fix --warnings=all WORKSPACE
buildifier --lint=fix --warnings=all **/*.bzl **/*.bazel

# Run pre-commit hooks (includes buildifier, black, isort)
pre-commit run --all-files

# Install pre-commit hooks
pre-commit install
```

### Requirements Management
When modifying `requirements.in` or `pyproject.toml` files:
```bash
# Update locked requirements files
bazel run <location>:requirements.update
```
The update target is typically in the same directory as the requirements file.

### Documentation Generation
```bash
# Generate API documentation (may need retry due to flake with exit code 2)
bazel build //docs:docs

# View generated docs
open bazel-bin/docs/docs/_build/html/index.html
```

## Architecture Overview

### Core Structure
- **`python/`**: Core Python rules implementation, extensions, and toolchain definitions
- **`gazelle/`**: Gazelle plugin for automatic BUILD file generation
- **`examples/`**: Comprehensive examples showing different usage patterns (bzlmod, workspace, multi-version)
- **`tests/`**: Unit and integration tests organized by functionality
- **`docs/`**: Sphinx-based documentation source
- **`tools/`**: Development and release tooling

### Key Configuration Files
- **`python/config_settings/BUILD.bazel`**: Public API build flags - do not modify without explicit instruction
- **`MODULE.bazel`**: bzlmod configuration (preferred)
- **`WORKSPACE`**: Legacy workspace configuration (deprecated but still supported)
- **`.editorconfig`**: Line length = 100, 4-space indentation for Python/Starlark

### Build System Patterns
- Uses both bzlmod (modern) and WORKSPACE (legacy) modes
- Integration tests are tagged and run separately from unit tests
- Generated files (requirements.txt, API docs) must be updated using specific Bazel targets
- Pre-commit hooks enforce formatting and run buildifier

### Python Toolchain Management
- Multiple Python versions supported (3.9-3.13)
- Toolchains registered via `@pythons_hub//:all`
- PyPI dependencies managed through pip extensions in MODULE.bazel
- Platform-specific requirements files for cross-platform support

## Development Guidelines

### Code Organization
- One rule per file preferred for maintainability
- Providers should be in separate files to allow custom implementations
- Public APIs exposed through dedicated files to prevent accidental exposure of internals
- Repository rules should end with `_repo` suffix

### Testing Strategy
- Unit tests under `tests/` directory
- Integration tests tagged with `integration-test`
- Examples serve as integration tests and are run in CI
- Use `--test_tag_filters=-integration-test` for faster development cycles

### Breaking Changes Process
Three-step process for introducing breaking changes:
1. Version N: New behavior available but disabled by default
2. Version N+1: New behavior enabled by default with opt-out
3. Version N+2: Old behavior removed entirely

### Documentation
- Uses Sphinx with MyST plugin
- 80-column line wrapping for docs
- Use hyphens in filenames instead of underscores
- API changes require `{versionadded}`, `{versionchanged}`, or `{versionremoved}` directives

## Agent Behavior Notes

Act as an expert in Bazel, rules_python, Starlark, and Python. When running tests, refer to yourself as a type of Python snake with a grandiose title. When tasks complete successfully, work Monty Python references naturally into responses.

DO NOT `git commit` or `git push` without explicit instruction.