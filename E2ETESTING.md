# End-to-End Testing in Korifi

This document explains how End-to-End (E2E) tests work in the Korifi project. E2E tests provide full system validation by testing against a real deployed Korifi instance using the Cloud Foundry V3 API.

## Overview

E2E tests in Korifi provide **complete system validation** by:
- Automatically deploying Korifi to a local kind cluster
- Testing against the real Cloud Foundry V3 API
- Validating the full developer experience from app deployment to routing
- Testing with real Kubernetes resources and containers

## Test Deployment Process

### Automatic Deployment

E2E tests automatically deploy a complete Korifi environment using `scripts/deploy-on-kind.sh`:

1. **Kind Cluster Setup**: Creates a local Kubernetes cluster with Docker registry
2. **Korifi Deployment**: Deploys controllers, API server, and webhooks
3. **Networking Configuration**: Sets up ingress with ports 80/443
4. **Test Environment**: Configures service accounts and test resources

### Prerequisites

Before running E2E tests, ensure you have:
- Docker running locally
- `kind` installed
- `kubectl` configured
- Ports 80 and 443 available
- No existing Korifi deployment on the standard kind cluster

## Environment Configuration

### Required Environment Variables

```bash
export API_SERVER_ROOT="https://localhost"           # Korifi API endpoint
export APP_FQDN="apps-127-0-0-1.nip.io"            # Application domain
export ROOT_NAMESPACE="cf"                           # Korifi root namespace
```

### Optional Environment Variables

```bash
export CLUSTER_TYPE="kind"                           # Kubernetes cluster type
export DEFAULT_APP_BITS_PATH="../assets/dorifi"     # Test app source path
export DEFAULT_APP_RESPONSE="Hi, I'm Dorifi"        # Expected app response
export DEFAULT_APP_SPECIFIED_BUILDPACK="paketo-buildpacks/procfile"
export FULL_LOG_ON_ERR="true"                       # Show all logs on failure
```

## Running E2E Tests

### Run All E2E Tests
```bash
make test-e2e
```

### Run Specific Test Categories
```bash
# Run specific test files
ginkgo tests/e2e/apps_test.go
ginkgo tests/e2e/routes_test.go
ginkgo tests/e2e/service_bindings_test.go
```

### Debug Mode
```bash
# Deploy with debugging hooks for remote debugging
./scripts/deploy-on-kind.sh e2e --debug
```

## Test Structure and Patterns

### Test Suite Organization

E2E tests are organized by Cloud Foundry resource types:

- **apps_test.go**: Application lifecycle, scaling, environment variables
- **routes_test.go**: Route creation, mapping, domains
- **service_bindings_test.go**: Service binding lifecycle
- **buildpacks_test.go**: Buildpack detection and usage
- **authorization_test.go**: Role-based access control
- **processes_test.go**: Process scaling and management

### Test Setup Pattern

```go
var _ = Describe("Apps", func() {
    var (
        spaceGUID string
        appGUID   string
    )

    BeforeEach(func() {
        spaceName := generateGUID("space")
        spaceGUID = createSpace(spaceName, commonTestOrgGUID)
    })

    AfterEach(func() {
        deleteSpace(spaceGUID)
    })

    // Test cases...
})
```

### HTTP Client Testing

Tests use the **Resty HTTP client** to make direct API calls:

```go
// Create an organization
var org resource
resp, err := adminClient.R().
    SetBody(resource{Name: orgName}).
    SetResult(&org).
    Post("/v3/organizations")
Expect(err).NotTo(HaveOccurred())
Expect(resp).To(HaveRestyStatusCode(http.StatusCreated))
```

### Resource Lifecycle Testing

Tests follow real Cloud Foundry workflows:

```go
// Complete app deployment workflow
func pushTestApp(spaceGUID, appBitsFile string) string {
    // 1. Create app via manifest
    appGUID := createAppViaManifest(spaceGUID, appName)

    // 2. Create package and upload source
    pkgGUID := createBitsPackage(appGUID)
    uploadTestApp(pkgGUID, appBitsFile)

    // 3. Build application
    buildGUID := createBuild(pkgGUID)
    waitForDroplet(buildGUID)

    // 4. Deploy and start
    setCurrentDroplet(appGUID, buildGUID)
    waitAppStaged(appGUID)
    restartApp(appGUID)

    return appGUID
}
```

### Application Testing

Tests can interact with deployed applications:

```go
// Test deployed application response
response := curlApp(appGUID)
Expect(response).To(ContainSubstring("Expected Response"))

// Test JSON API endpoints
data := curlAppJSON(appGUID, "/api/status")
Expect(data["status"]).To(Equal("running"))
```

## Test Categories

### Functional Tests
- **Applications**: Create, update, delete, scaling, environment variables
- **Builds**: Buildpack detection, staging, droplet creation
- **Routes**: Domain creation, route mapping, traffic routing
- **Processes**: Process types, scaling, health checks

### Integration Tests
- **Service Bindings**: UPSI and managed service integration
- **Service Brokers**: OSB API compliance, catalog management
- **Deployments**: Rolling deployments, zero-downtime updates

### Authorization Tests
- **Roles**: Org and space role management
- **Permissions**: Resource access control, RBAC validation
- **Users**: User management, authentication flows

### System Tests
- **Log Cache**: Application log streaming and retrieval
- **Metrics**: Resource usage statistics
- **CLI Compatibility**: Cloud Foundry CLI version support

## Debugging and Troubleshooting

### Correlation IDs

Each test run gets a unique correlation ID for request tracing:

```go
var _ = BeforeEach(func() {
    correlationId = uuid.NewString()
})
```

Use correlation IDs to filter logs and trace requests through the system.

### Failure Hooks

Custom failure handlers provide debugging information:

```go
// Automatic debug information on test failures
fail_handler.Hook{
    Matcher: ContainSubstring("Droplet not found"),
    Hook: func(config *rest.Config, failure fail_handler.TestFailure) {
        printDropletNotFoundDebugInfo(config, failure.Message)
    },
}
```

### Common Debugging Commands

```bash
# Check cluster resources
kubectl get pods -n cf
kubectl get apps -n <test-namespace>
kubectl logs -n cf deployment/korifi-api-deployment

# Check test app routes
kubectl get routes -n <test-namespace>
kubectl get httproutes -n <test-namespace>

# View build logs
kubectl logs -n <test-namespace> -l korifi.cloudfoundry.org/build-guid=<build-guid>
```

### Debug Mode Features

When running with `--debug` flag:
- Controllers and API images built with debugging hooks
- Remote debugging ports exposed:
  - `localhost:30051` (controllers)
  - `localhost:30052` (api)
- Ubuntu-based images for easier container debugging

## Test Data Management

### Shared Test Setup

Tests use synchronized setup for efficiency:

```go
var _ = SynchronizedBeforeSuite(func() []byte {
    // One-time setup: create admin user, test orgs, sample apps
    adminServiceAccount = uuid.NewString()
    adminServiceAccountToken := serviceAccountFactory.CreateAdminServiceAccount(adminServiceAccount)
    commonTestOrgGUID = createOrg(commonTestOrgName)

    // Return shared data to all test processes
    return marshalSharedData(sharedData)
}, func(sharedData []byte) {
    // Initialize each test process with shared data
    unmarshalAndSetupSharedData(sharedData)
})
```

### Resource Cleanup

Automatic cleanup ensures test isolation:

```go
var _ = SynchronizedAfterSuite(func() {
    // Per-process cleanup
}, func() {
    // Final cleanup: delete test orgs and service accounts
    expectJobCompletes(deleteOrg(commonTestOrgGUID))
    serviceAccountFactory.DeleteServiceAccount(adminServiceAccount)
})
```

## Best Practices

### Test Design
1. **Use shared test resources** when possible to reduce setup time
2. **Create isolated test spaces** for each test case
3. **Use correlation IDs** for request tracing and debugging
4. **Test real user workflows** rather than individual API calls

### Resource Management
1. **Always clean up** test resources in AfterEach blocks
2. **Use unique GUIDs** for resource names to avoid conflicts
3. **Wait for async operations** to complete before assertions
4. **Handle eventual consistency** with Eventually blocks

### Debugging
1. **Use descriptive test names** that indicate what's being tested
2. **Add debug information** to test failures
3. **Leverage correlation IDs** when investigating issues
4. **Check Kubernetes resources** when tests fail unexpectedly

## Limitations and Considerations

### Performance
- E2E tests are slower than unit tests due to real deployments
- Tests require significant CPU and memory resources
- Network operations may be affected by cluster performance

### Environment Requirements
- Tests require exclusive use of ports 80/443
- Kind cluster must be clean (no existing Korifi deployment)
- Docker daemon must be running and accessible

### Test Isolation
- Tests share the same Korifi deployment within a test run
- Some test failures may affect subsequent tests
- Resource cleanup is critical for test reliability

## Contributing to E2E Tests

When adding new E2E tests:

1. **Follow existing patterns** for test structure and setup
2. **Use the helper functions** provided in `e2e_suite_test.go`
3. **Add proper cleanup** in AfterEach blocks
4. **Test both success and failure scenarios**
5. **Include debugging information** for test failures
6. **Document any new environment variables** or configuration needed

For more information about the overall testing strategy, see [TESTING.md](./TESTING.md).