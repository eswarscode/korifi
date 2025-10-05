# Testing in Korifi

This document explains the testing setup and conventions used in the Korifi project.

## Test Framework

Korifi uses **Go with Ginkgo/Gomega** as the primary testing framework:

- **Ginkgo v2** - BDD (Behavior-Driven Development) testing framework
- **Gomega** - Matcher library used with Ginkgo for assertions
- **Counterfeiter** - Tool for generating mocks and fakes

## Test Structure

### Unit Tests

Unit tests are located alongside source code using the `*_test.go` naming convention:

- **API handlers**: `api/handlers/*_test.go`
- **Authorization**: `api/authorization/*_test.go`
- **Controllers**: `controllers/**/*_test.go`
- **Tools/utilities**: `tools/**/*_test.go`

### Integration Tests

Integration tests are organized in the dedicated `tests/` directory:

- **E2E tests**: `tests/e2e/` - End-to-end tests that exercise the full system
- **CRD tests**: `tests/crds/` - Custom Resource Definition validation tests
- **Smoke tests**: `tests/smoke/` - Basic functionality verification tests

## Running Tests

### All Tests
```bash
make test
```
This runs linting, unit tests, tools tests, and e2e tests in sequence.

### Specific Test Types

**Tools tests**:
```bash
make test-tools
```

**End-to-end tests**:
```bash
make test-e2e
```
*Note: E2E tests automatically deploy Korifi to a local kind cluster and require ports 80/443 to be available.*

**CRD tests**:
```bash
make test-crds
```

**Smoke tests**:
```bash
make test-smoke
```

## Test Patterns

### Test Suites

Tests are organized using **test suites** (e.g., `handlers_suite_test.go`) with shared setup in `BeforeEach` blocks. Each test suite typically includes:

- Suite setup and teardown
- Common test fixtures
- Shared helper functions

### Custom Matchers

The project includes custom Gomega matchers located in `tests/matchers/` for domain-specific assertions.

### Test Helpers

Reusable test utilities are available in `tests/helpers/` to reduce duplication across test files.

## End-to-End Testing

E2E tests provide comprehensive validation by:

1. Automatically deploying Korifi to a local kind cluster
2. Testing against the full Cloud Foundry API
3. Verifying real-world usage scenarios

**Prerequisites for E2E tests**:
- Ports 80 and 443 must be available
- Docker and kind must be installed
- No existing Korifi deployment on the standard kind cluster

## Test Configuration

Tests use environment variables for configuration:
- `API_SERVER_ROOT` - API server endpoint (default: https://localhost)
- `APP_FQDN` - Application domain (default: apps-127-0-0-1.nip.io)
- `ROOT_NAMESPACE` - Root namespace (default: cf)
- `SKIP_DEPLOY` - Skip deployment in test setup

## Development Workflow

1. Write unit tests alongside your code changes
2. Run `make test-tools` for quick feedback on utilities
3. Run `make test` before committing to ensure all tests pass
4. Use `make test-e2e` to validate end-to-end functionality

For more details on the development workflow, see [HACKING.md](./HACKING.md).