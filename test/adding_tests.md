# Adding tests

If you are [developing knative](/DEVELOPMENT.md) you may need to add or change:

* [e2e tests](./e2e)
* [Conformance tests](./conformance)

Both tests can use our [test library](#test-library).

Reviewers of conformance and e2e tests (i.e. [OWNERS](/test/OWNERS)) are responsible for the style and quality of the resulting tests. In order to not discourage contributions, when style change are required, the reviewers can make the changes themselves.

All e2e and conformance tests _must_ be marked with the `e2e` [build constraint](https://golang.org/pkg/go/build/)
so that `go test ./...` can be used to run only [the unit tests](README.md#running-unit-tests), i.e.:

```go
// +build e2e
```

## Test library

In the [`test`](/test/) dir you will find several libraries in the `test` package
you can use in your tests.

You can:

* [Use common test flags](#use-common-test-flags)
* [Output logs](#output-logs)
* [Emit metrics](#emit-metrics)
* [Get access to client objects](#get-access-to-client-objects)
* [Make requests against deployed services](#make-requests-against-deployed-services)
* [Check Knative Serving resources](#check-knative-serving-resources)
* [Verify resource state transitions](#verify-resource-state-transitions)
* [Generate boilerplate CRDs](#generate-boilerplate-crds)
* [Ensure test cleanup](#ensure-test-cleanup)

### Use common test flags

These flags are useful for running against an existing cluster, making use of your existing
[environment setup](/DEVELOPMENT.md#environment-setup).

By importing `github.com/knative/serving/test` you get access to a global variable called
`test.Flags` which holds the values of [the command line flags](/test/README.md#flags).

```go
imagePath := strings.Join([]string{test.Flags.DockerRepo, image}, "/"))
```

_See [e2e_flags.go](./e2e_flags.go)._

### Output logs

[When tests are run with `--logverbose`
option](README.md#output-verbose-logs), debug logs will be emitted to stdout.

We are using the common [e2e logging library](/pkg/test/logging) that uses the [Knative logging library](/pkg/logging/) for structured logging. 
It is built on top of [zap](https://github.com/uber-go/zap). Tests should initialize the global logger to use a test specifc context with `logging.GetContextLogger`:

```go
// The convention is for the name of the logger to match the name of the test.
logging.GetContextLogger("TestHelloWorld")
```

Logs can then be emitted using the `logger` object which is required by
many functions in the test library. To emit logs:

```go
logger.Infof("Creating a new Route %s and Configuration %s", route, configuration)
logger.Debugf("The LogURL is %s, not yet verifying", logURL)
```

_See [logging.go](./logging/logging.go)._

### Emit metrics

You can emit metrics from your tests using [the opencensus
library](https://github.com/census-instrumentation/opencensus-go), which [is being
used inside Knative as well](/docs/telemetry.md). These metrics will be emitted by
the test if the test is run with [the `--emitmetrics` option](README.md#emit-metrics).

You can record arbitrary metrics with
[`stats.Record`](https://github.com/census-instrumentation/opencensus-go#stats) or
measure latency with [`trace.StartSpan`](https://github.com/census-instrumentation/opencensus-go#traces):

```go
ctx, span := trace.StartSpan(context.Background(), "MyMetric")
```

* These traces will be emitted automatically by [the generic crd polling
  functions](#check-knative-serving-resources).
* The traces are emitted by [a custom metric exporter](./logging.go)
  that uses the global logger instance.

#### Metric format

When a `trace` metric is emitted, the format is `metric name startTime endTime duration`. The name
of the metric is arbitrary and can be any string. The values are:

* "metric" - Indicates this log is a metric
* name - Arbitrary string indentifying the metric. TODO(#1379) determine metric format
* startTime - Unix time in nanoseconds when measurement started
* endTime - Unix time in nanoseconds when measurement ended
* duration - The difference in ms between the startTime and endTime

For example:

```bash
metric WaitForConfigurationState/prodxiparjxt/ConfigurationUpdatedWithRevision 1529980772357637397 1529980772431586609 73.949212ms
```

_The [`Wait` methods](#check-knative-serving-resources) (which poll resources) will
prefix the metric names with the name of the function, and if applicable, the name of the resource,
separated by `/`. In the example above, `WaitForConfigurationState` is the name of
the function, and `prodxiparjxt` is the name of the configuration resource being polled.
`ConfigurationUpdatedWithRevision` is the string passed to `WaitForConfigurationState` by
the caller to identify what state is being polled for._

### Get access to client objects

To initialize client objects that you can use [the command line flags](#use-flags)
that describe the environment:

```go
func setup(t *testing.T) *test.Clients {
    clients, err := test.NewClients(kubeconfig, cluster, namespaceName)
    if err != nil {
        t.Fatalf("Couldn't initialize clients: %v", err)
    }
    return clients
}
```

The `Clients` struct contains initialized clients for accessing:

* Kubernetes objects
* `Routes`
* `Configurations`
* `Revisions`

For example, to create a `Route`:

```bash
_, err = clients.ServingClient.Routes.Create(test.Route(namespaceName, routeName, configName))
```

And you can use the client to clean up `Route` and `Configuration` resources created
by your test:

```go
func tearDown(clients *test.Clients) {
    if clients != nil {
        clients.Delete([]string{routeName}, []string{configName})
    }
}
```

_See [clients.go](./clients.go)._

### Make requests against deployed services

After deploying (i.e. creating a `Route` and a `Configuration`) an endpoint will not be
ready to serve requests right away. To poll a deployed endpoint and wait for it to be
in the state you want it to be in (or timeout) use `WaitForEndpointState`:

```go
err = test.WaitForEndpointState(
		clients.KubeClient,
		logger,
		test.ServingFlags.ResolvableDomain,
		updatedRoute.Status.Domain,
		test.EventuallyMatchesBody(expectedText),
		"SomeDescription")
if err != nil {
    t.Fatalf("The endpoint for Route %s at domain %s didn't serve the expected text \"%s\": %v", routeName, updatedRoute.Status.Domain, expectedText, err)
}
```

This function makes use of [the environment flag `resolvableDomain`](#use-flags) to determine if the ingress
should be used or the domain should be used directly.

_See [request.go](./request.go)._

If you need more low-level access to the http request or response against a deployed
service, you can directly use the `SpoofingClient` that `WaitForEndpointState` wraps.


```go
// Error handling elided for brevity, but you know better.
client, err := spoof.New(clients.KubeClient.Kube, logger, route.Status.Domain, test.Flags.ResolvableDomain)
req, err := http.NewRequest(http.MethodGet, fmt.Sprintf("http://%s", route.Status.Domain), nil)

// Single request.
resp, err := client.Do(req)

// Polling until we meet some condition.
resp, err := client.Poll(req, test.BodyMatches(expectedText))
```

_See [spoof.go](./spoof/spoof.go)._

### Check Knative Serving resources

After creating Knative Serving resources or making changes to them, you will need to wait for the system
to realize those changes. You can use the Knative Serving CRD check and polling methods to check the
resources are either in or reach the desired state.

The `WaitFor*` functions use the kubernetes [`wait` package](https://godoc.org/k8s.io/apimachinery/pkg/util/wait).
To poll they use [`PollImmediate`](https://godoc.org/k8s.io/apimachinery/pkg/util/wait#PollImmediate)
and the return values of the function you provide behave the same as
[`ConditionFunc`](https://godoc.org/k8s.io/apimachinery/pkg/util/wait#ConditionFunc):
a `bool` to indicate if the function should stop or continue polling, and an `error` to indicate if
there has been an error.

For example, you can poll a `Configuration` object to find the name of the `Revision` that was created
for it:

```go
var revisionName string
err := test.WaitForConfigurationState(clients.ServingClient, configName, func(c *v1alpha1.Configuration) (bool, error) {
    if c.Status.LatestCreatedRevisionName != "" {
        revisionName = c.Status.LatestCreatedRevisionName
        return true, nil
    }
    return false, nil
}, "ConfigurationUpdatedWithRevision")
```

_[Metrics will be emitted](#emit-metrics) for these `Wait` method tracking how long test poll for._

We also have `Check*` variants of many of these methods with identical signatures, same example:

```go
var revisionName string
err := test.CheckConfigurationState(clients.ServingClient, configName, func(c *v1alpha1.Configuration) (bool, error) {
    if c.Status.LatestCreatedRevisionName != "" {
        revisionName = c.Status.LatestCreatedRevisionName
        return true, nil
    }
    return false, nil
})
```

_See [crd_checks.go](./crd_checks.go) and [kube_checks.go](./kube_checks.go)._

### Verify resource state transitions

To use the [check functions](#check-knative-serving-resources) you must provide a function to check the
state. Some of the expected transition states (as defined in [the Knative Serving spec](/docs/spec/spec.md))
are expressed in functions in [states.go](./states.go).

For example when a `Revision` has been created, the system will start the resources required to
actually serve it, and then the `Revision` object will be updated to indicate it is ready. This
can be polled with `test.IsRevisionReady`:

```go
err := test.WaitForRevisionState(clients.ServingClient, revisionName, test.IsRevisionReady(revisionName))
if err != nil {
    t.Fatalf("Revision %s did not become ready to serve traffic: %v", revisionName, err)
}
```

Once the `Revision` is created, all traffic for a `Route` should be routed to it. This can be polled with
`test.AllRouteTrafficAtRevision`:

```go
err = test.WaitForRouteState(clients.ServingClient, routeName, test.AllRouteTrafficAtRevision(routeName, revisionName))
if err != nil {
    t.Fatalf("The Route %s was not updated to route traffic to the Revision %s: %v", routeName, revisionName, err)
}
```

_See [states.go](./states.go)._

### Generate boilerplate CRDs

Your tests will probably need to create `Route` and `Configuration` objects. You can use the
existing boilerplate to describe them.

You can also use the function `RandomizedName` to create a random name for your `crd` so that
your tests can use unique names each time they run.

For example to create a `Configuration` object that uses a certain docker image with a
randomized name:

```go
var names test.ResourceNames
names.Config := test.RandomizedName('hotdog')
_, err := clients.ServingClient.Create(test.Configuration(namespaceName, names, imagePath))
if err != nil {
    return err
}
```

Please expand these functions as more use cases are tested.

_See [crd.go](./crd.go)._

### Ensure test cleanup

To ensure your test is cleaned up, you should defer cleanup to execute after your
test completes and also ensure the cleanup occurs if the test is interrupted:

```go
defer tearDown(clients)
test.CleanupOnInterrupt(func() { tearDown(clients) })
```

_See [cleanup.go](./cleanup.go)._
