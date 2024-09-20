---
title: extended-platform-tests
authors:
  - "@jupierce"
  - "@stbenjam"
reviewers:
  - "@deads2k"
creation-date: 2024-09-05
last-updated: 2024-09-05
status: implementable
---

<!-- TOC -->
* [OpenShift Tests Extensions](#openshift-tests-extensions)
  * [Release Signoff Checklist](#release-signoff-checklist)
  * [Summary](#summary)
  * [Motivation](#motivation)
    * [Goals](#goals)
    * [Non-Goals](#non-goals)
  * [Proposal](#proposal)
    * [Concepts](#concepts)
      * [Component](#component)
      * [Subcomponent](#subcomponent)
      * [Test ID](#test-id)
      * [Test Environment](#test-environment)
      * [Test Context](#test-context)
    * [Test Extension Binaries](#test-extension-binaries)
      * [Binary Discovery](#binary-discovery)
        * [OpenShift Payload Extension Binaries](#openshift-payload-extension-binaries)
        * [Non-Payload Extension Binaries](#non-payload-extension-binaries)
      * [Binary Format](#binary-format)
      * [Binary Extraction](#binary-extraction)
      * [Extension Interface](#extension-interface)
        * [Info - Extension Metadata](#info---extension-metadata)
        * [List - Extension Test Listing](#list---extension-test-listing)
        * [Run-Test - Running Extension Tests](#run-test---running-extension-tests)
        * [Config - Component Configuration Testing](#config---component-configuration-testing)
      * [Test Result Aggregation](#test-result-aggregation)
    * [Risks and Mitigations](#risks-and-mitigations)
      * [Binary Incompatibility](#binary-incompatibility)
        * [CPU Architecture](#cpu-architecture)
      * [Runtime Size / Speed](#runtime-size--speed)
      * [Image Size](#image-size)
      * [Poor Extension Implementation](#poor-extension-implementation)
    * [Version Skew Strategy](#version-skew-strategy)
  * [Alternatives](#alternatives)
<!-- TOC -->

# OpenShift Tests Extensions

## Release Signoff Checklist

- [ ] Enhancement is `implementable`
- [ ] Design details are appropriately documented from clear requirements
- [ ] Test plan is defined
- [ ] Graduation criteria for dev preview, tech preview, GA
- [ ] User-facing documentation is created in [openshift-docs](https://github.com/openshift/openshift-docs/)

## Summary

Define a pattern by which repos other than origin can contribute tests to 
OpenShift test suites.
// Layered on commits from Stephen with good description from gdoc if we move forward.

## Motivation

Increase the number of tests in the OpenShift Tests Suite by reducing the
effort, complexity, and risk of introducing new tests.

### Goals

1. Reduce the effort, complexity, and risk of introducing new tests to OpenShift.
2. Allow tests to be introduced in the same pull request as the PR which fixes a bug or introduces a feature.
3. Allow component owners to introduce new test suites / add tests to existing suites.
4. Allow non-OpenShift components, including layered products, to participate in extending OpenShift tests and Component Readiness.
5. Maintain and increase testing coverage.
6. Maintain and increase code quality.
7. Maintain and improve the signal TRT provides through Component Readiness.
8. Provide a standard model for testing component configurations.
9. Allow distributed contribution of tests while introducing centralized mechanisms to require important tests from component owners and improve coverage over time.
10. Improve testing efficiency by reducing initialization time.
11. Allow centralized and methodical pre-submit testing of the test extension implementations. For example, preventing a test rename without preserving information about the original name.

### Non-Goals
1. Fully decompose `openshift-tests` or decentralize test orchestration & reporting.

## Proposal

### Concepts

#### Component
A component is a part of a software product. Component information includes the product name, 
component name, and the type of the component. A single invocation of an external binary 
can only associate with a single component (e.g. it can only list tests for / execute tests
for a single component).

#### Subcomponent
A further refinement of a component, if necessary. When combined with Component information,
it should be sufficient to file a Jira ticket that will reach domain experts. 

#### Test ID
A unique combination of Component (Product, Type, Component Name), Subcomponent, 
and original test name. A Test ID should not be repeated in a test listing across all 
participating extensions. The Test ID representing a unit of testing logic should
not change over time (even if the human-readable test name changes, the Test ID
should remain consistent by using the original test name). 

#### Test Environment
A Test ID references a unit of testing logic. That logic may pass in some environments,
fail in others (e.g. a test that passes in gcp but fails in aws), and be 
intentionally skipped in still others. A Test Environment
describes relevant facets of the environment in which a test would run or was run, including 
configuration information.

Component Readiness may analyze test results using all, some, or none of this Environment
as dimensions along with Test ID to determine regressions, depending on the desired
insight (e.g. if the goal is to detect whether a test has regressed, on average, across 
all environments, the environment will be ignored for the analysis).

#### Test Context
This is information known only by `openshift-tests` as a part of orchestrating test
execution. It includes information like the random seed being used for test execution
planning and the number of times a test has been run against a cluster.

Rarely, Component Readiness may analyze results using these dimensions to, for example,
determine that different runs of the same test on the same cluster behave differently
in a statistically significant way.

### Test Extension Binaries
Test extension binaries will be defined as a first class method of introducing new tests and suites
into OpenShift origin's test framework. Historically, all tests had to be contributed to 
github.com/openshift/origin, but extension binaries can be developed in other repositories
and built independently of origin.

Extension binaries must implement an "extension interface" consisting of CLI verbs
and arguments that allow tests to be discovered and executed. The extension interface
is non-trivial to implement, so to simplify the creation of extensions, a Go module & 
repository will be maintained by TRT: https://github.com/openshift-eng/openshift-tests-extension . 

Vendoring this module and following its integration guide will provide component test authors
the majority of the logic necessary to fulfill the test extension interface. It will
also provide a means of centrally defining new testing requirements. As component owners re-vendor
the module, they would be required to implement those new testing requirements
for the module to compile and run a test extension binary successfully.

During a run of the origin test framework (`openshift-tests`), the framework will:
1. Discover available extension binaries.
2. Discover metadata and component configuration options for those extensions (running `info` verb and parsing standard out).
3. Discover tests available from those extensions (running `list` verb and parsing standard out).
4. Plan an appropriate and efficient testing strategy based on the framework invocation.
5. Execute the tests using subsequent invocations of the discovered binaries (`run-test` invocations).
6. Collect the result and output of the test invocations and integrate it into overall test suite results.


#### Binary Discovery
The existence of test extension binaries can be registered one of two ways by test authors. 

##### OpenShift Payload Extension Binaries
For OpenShift payload components contributors can advertise the existence of an extension binary
by adding information (the imagestream tag for the OCP payload component and the path to the binary 
within their image) to a simple registry datastructure in github.com/openshift/origin. 

##### Non-Payload Extension Binaries
For non-payload components, contributors must advertise their extension binary for `openshift-tests`
to discover. This is accomplished by creating an ImageStream / ImageStreamTag with a special label:

`testextension.redhat.io/component=<product-name>-<component-name>`

An annotation on the ImageStreamTag will then identify the binary to extract and (optional)
arguments to pass ahead of the extension interface verbs.
`testextension.redhat.io/binary=<binary-path>.gz [--argument=value]`

Well-behaved operators should gate the creation of this imagestream/label/annotation on the existence of the
`TestExtensionAdmission` custom resource definition (the code should not attempt to 
inspect `TestExtensionAdmissions` instances -- just check for the CRD itself so that operators
don't liter clusters with ImageStreams only applicable for testing in a production environment).

Optional Operator authors must ensure that the image carrying the extension binary
is identified in their ClusterServiceVersion (CSV) so that tools like `oc-mirror`
will copy image(s) bearing extension binaries to disconnected clusters.

Instances of `TestExtensionAdmission` gate which extension binaries `openshift-tests`
will use for test discovery. This is because extension binaries will be invoked
with a `system:admin` kubeconfig when tests are run. Administrators
must opt-in to this risk by instantiating one or more `TestExtensionAdmission`
objects. 

`TestExtensionAdmission` contain a list of namespaces/imagestreams 
from which `openshift-tests` is permitted to extract extension binaries. This
can include wildcard patterns for namespace, imagestream, or both.

```yaml
kind: TestExtensionAdmission
metadata:
  name: example
spec:
  permit:
  # All imagestreams in the openshift namespace with the testextension label
  - "openshift/*" 
  # All imagestreams on the cluster with the testextension label
  - "*/*" 
```

`openshift-tests` will SKIP a single synthetic test whenever an extension binary is detected in 
any imagestreamtag but which is not permitted for execution by a `TestExtensionAdmission` 
instance. This will allow users to detect tests that are available but not yet explicitly
permitted to run.

#### Binary Format
Extension binaries are extracted from container images into a running Pod executing 
`openshift-tests`. Contributors do not necessarily know which version of RHEL the
`openshift-tests` binary will be running on, so, for maximum portability, test
extension binaries should be statically compiled.

Statically linked binaries are prohibited in FIPS and will cause failures if
detected by product pipeline scans. To avoid this, extension binaries should be
gzipped before being committed to container images.

For compliance reasons, when a binary is compiled by a golang builder image
supplied by ART, a wrapper around the `go` compiler will force FIPS compliant
compilation (e.g. dynamic linking). For this reason, extension binaries
should include `GO_COMPLIANCE_POLICY="exempt_all"` in the environment when
compiling an extension binary. This will inhibit the wrapper from 
modifying `CGO_ENABLED` or other compiler arguments required for static compilation.

#### Binary Extraction
After discovering a test extension binary, the origin test framework will extract the binary
from the container image which carries it and store it in /tmp storage of the pod in which
it is running.

If the binary-path ends in `.gz`, the binary will be decompressed.

#### Extension Interface
Test Extension binaries must implement a well-defined interface through which information about the
tests the binary offers can be discovered and executed. This interface includes several command line
verbs the binary must implement, arguments those verbs must support, and output formats the binary
must adhere to.

Running an extension binary will output the following help text for the initial version of the interface. 
```
info      - Output test contribution extension version and metadata.
list      - Output tests supported by this extension.
run-test  - Run one or more tests and output results.
config    - Component configuration management.
update    - Update git metadata for extension.
```

##### Info - Extension Metadata
The extension interface will evolve over time. For this reason, the `info` verb
will output information about which version of the interface the binary supports. The origin
framework will support some level of skew in the interface versions and invoke the binaries
with verbs & arguments consistent with the interface version they support.

Info also provides information to origin about how to construct new or contribute new
test suites.

Annotated example `info` output is provided below.
```python
{
    # The extension interface version the binary implements.
    # The details of the interface and version will usually be
    # unnecessary for a contributor to understand as they will
    # vendor in the majority of the implementation from the
    # "Test Extension Support Module" explained later.
    "apiVersion": "1.0",

    "source": {
        # The commit from which this binary was compiled.
        "commit": "87fdbaa",
        # The git repository (if known) that this
        # extension was built from.
        "source_url": "github.com/openshift/kubernetes"
    },

    # A single extension can carry tests for multiple
    # components. However, each invocation of the binary
    # can only represent and act on information relative
    # to a single component.
    # If a binary carries tests for multiple components,
    # it must register itself multiple times, with
    # different pre-verb arguments which openshift-tests
    # will provide verbatim with every invocation; e.g.
    # if the binary registers with "--component hyperkube",
    # verb invocations will be as follows:
    #    extension-binary --component hyperkube info
    #    extension-binary --component hyperkube list ...
    "component": {
        "product": "openshift",
        "type": "payload",
        "name": "hyperkube"
    },

    "configurations": [
        {
            "profile": "common-mode-1",
            "description": "Enables a common, non-default mode of operation where ....",
            "application": {
                # If applying the configuration implies a disruption, inform 
                # openshift-tests, so that it can be accounted for in overall
                # disruption reporting.
                "disruption": "1m",
                # If, after applying the component configuration, openshift-tests
                # should await cluster steady state before running configuration
                # specific tests. If not specified, openshift-tests assumes
                # the tests can be run immediately after applying the configuration
                # profile.
                "await": "steady-state"
            },
            
            "resources": {
              
              # openshift-tests will only apply a single configuration
              # profile to a component at a given time.
              # However, inter-component conflicts might still occur.
              # Configurations can request to be run in isolation
              # from other component configurations. 
              # If two component configurations have a conflict name in common, 
              # test planning will ensure that there is no attempt to 
              # apply those profiles simultaneously.
              "isolation": {
                "conflict":  [
                  "gpu",
                ]
              },
                
              # Tests that require significant resources can identify
              # that fact so that the planning algorithm will not
              # overdraw on allocatable pod resources for a set of
              # tests run in parallel.
              "memory": "1Gi",
              
              # If a duration can be roughly predicted, inclusion of this
              # information may improve the test execution 
              # planning algorithm performance.  
              "duration": "2s",
                
              # Timeout information may also support efficient
              # planning.
              "timeout": "16s",
            },
            
            
        }  
    ],
    
    # Additional suites the extension wants to advertise / participate in.
    "suites": [
        {
            # Here, the extension is advertising a new suite, unknown
            # to the origin framework until discovered through the
            # binary. 
            "name": "fips/conformance",

            "parents": [

                # This suite can be run separately or will be included
                # automatically in other suites known to origin.
                # Here, the extension is informing origin that the suite
                # is a sub-suite of openshift/conformance. All information
                # about a suite is additive across extension binaries.
                # For example, multiple binaries can advertise the same
                # suite -- if they advertise different parents, then
                # origin will treat the suite as a subset of each 
                # identified parent.
                "openshift/conformance"
            ],

            # Test cases can be advertised from across a number
            # of extension binaries. The following cel expressions
            # are OR'd together. If they select a test advertised
            # from another binary, they will be considered part of
            # this advertised suite.
            # Test tags, names, and other attributes will be
            # described later.
            "qualifiers": [
                "(test.tags.suite==\"fips\" || test.name.contains(\"fips\")) && !test.labels.has(\"Disruptive\")"
            ]
        }
    ]
    
}
```

##### List - Extension Test Listing
The "list" verb will output information about tests exposed by the extension
for different environments. A test "environment" describes the attributes of 
cluster under test (e.g. the cloud provider, the topology, etc).

The "list" verb should eliminate tests that are not appropriate for the
specified environment. 

The arguments accepted by the verb may vary over time and will be
defined by the extension interface version implemented by the binary.

**Version 1**
```
$ extension-binary list 
  --platform      The hardware or cloud platform ("aws", "gcp", "metal", ...).
  --network       The network of the target cluster ("ovn", "sdn"). 
  --upgrade       The upgrade that was performed prior to the test run ("micro", "minor").
  --topology      The target cluster topology ("ha", "microshift", ...).
  --architecture  The CPU architecture of the target cluster ("amd64", "arm64").
  --installer     The installer used to create the cluster ("ipi", "upi", "assisted", ...).
  --config        The component configuration to assume is active on the cluster.
  --version       "major.minor" version of target cluster. 
```

The "list" verb will not be provided a kubeconfig and must output tests based solely
on environment information. If no environment arguments are provided, all tests must be listed.

Listings are formatted as JSONL with one test description per line. A full listing contains
zero or more test description objects. The following example shows an abbreviated JSONL listing 
for three tests.
```python
{ "name": "T1", }
{ "name": "T2", }
{ "name": "T3", }
```

A test description object contains a number of attributes to inform `openshift-tests`
discovery and planning. Note that the following annotated example would be encoded into a single 
line of output in actual listing output.
```python
{
  
  # Base, human-readable test name. 
  "name": "openssl version compliance",

  # If a test name is updated at any time in the future,
  # originalName must report the original name of the 
  # testing logic. This allows component readiness
  # to display the human-readable version of the test
  # name while considering test runs across name changes.
  "originalName": "security version compliance",
    
  # Optional subcomponent information.
  "subcomponent": "ui",
    
  # Labels are text strings will can be used like 
  # Ginkgo labels to group and run tests. Until labels
  # are part of bigquery data, labels will be appended
  # to test names with "[label]" so that name based
  # suite construction will work. 
  "labels": [
    "sig-..."
  ],
  
  # Tags are key=value pairs that can be used to 
  # further classify tests.
  "tags": {
    "key": "value"
  },
  
  "resources": {
    # Tests can request to be run in isolation
    # from others at different levels:
    # "instance" - avoids being called along with a conflicting
    #              test in a single run-test invocation.
    # "exec"     - avoids being called at the same time in a
    #              across all parallel extension binary invocations.
    # "bucket"   - avoids being called in the same planning bucket.
    # If two tests have a conflict name in common, the isolation
    # mode will apply. "*" is a special conflict name will 
    # ensures complete isolation in the requested mode (e.g.
    # guaranteeing the test will be the only one to
    # a given invocation of run-test if mode=instance).
    "isolation": {
      "mode": "exec",
      "conflict":  [
        "gpu",
      ]
    },
      
    # Tests that require significant resources can identify
    # that fact so that the planning algorithm will not
    # overdraw on allocatable pod resources for a set of
    # tests run in parallel.
    "memory": "1Gi",
    
    # If a duration can be roughly predicted, inclusion of this
    # information may improve the test execution 
    # planning algorithm performance.  
    "duration": "2s",
      
    # Timeout information may also support efficient
    # planning.
    "timeout": "16s",
  },

  # Tests can be identified as "informing" or "production".
  # Informing tests will not negatively impact default views
  # in Component Readiness.
  # However, policies may be established in the future that 
  # cause tests that remain informing too long to be treated
  # as if they are production.
  "lifecycle": "informing",
  
}
```

##### Run-Test - Running Extension Tests
The `run-test` verb is used to cause the extension binary to actually run 
discovered tests. `run-test` accepts test environment arguments as discussed
in the "list" verb section (in order to reduce fact discovery time needed
for each test run) and one or more test names to invoke.
```
$ extension-binary run-test 
  --platform      The hardware or cloud platform ("aws", "gcp", "metal", ...).
  ...other context arguments...
  --config        The component configuration profile to assume is active on the cluster.
  --name | -n     Test name to invoke (-n can be specified multiple times).
```

`run-test` should run the specified tests in parallel. To prevent 
test cases developing a dependence on one another, `run-test` must make no
assumption about the number or composition of tests it is asked to run
(other than they will be consistent with the test environment and isolation
requirements defined by the "list" verb).

`openshift-tests` will randomize tests into multiple different parallel
executions of the extension binary in order to amortize initialization
costs.

Standard output from the invocation will be serialized into JSONL with
each line associated with a single test run result.

```python
{ "name": "T1", "result": "success", }
{ "name": "T1", "result": "success", }
{ "name": "T1", "result": "success", }
```

A single test result will contain the following information, encoded on a single line:
```python
{
    "name": "test name",
    
    # Duration of test run in milliseconds
    "duration": "1012",
    
    # The outcome of the test; pass, fail, skip, timeout.
    "result": "pass",
    "output": "standard output of the test",
    "error": "standard error of the test",
    
    # Human-readable messages to further explain 
    # skips / timeouts / etc.
    "messages": [
        "example message"
    ]
}
```

##### Config - Component Configuration Testing

A component can advertise that it wants to be exercised in multiple different configuration.
`openshift-tests` will plan an efficient method of testing those configurations and
ensure tests appropriate to that configuration are run.

The `config` verb will not return until the extension has watched the component
fully apply the configuration (or it should timeout with an error).
```
$ extension-binary config
  --profile <profile name>      The configuration profile to apply or "default".
```

The component configuration is considered part of the Environment and thus passed
in to `list` in order to determine appropriate tests for the configuration. 

Component authors may choose to reduce the number of tests run for non-default
configuration profiles, focusing only on tests likeliest to fail based on the 
configuration change, in order to reduce overall execution time.

`openshift-tests` will plan an efficient testing strategy based on the 
extension binaries it discovers, the configuration profiles they support,
and the tests that need to run in each configuration. It will:
1. Discover extension binaries.
2. Discover extension binary metadata (`info` verb), including configuration profiles.
3. Discover tests that should be run for each configuration profile (`list` verb with `--config <profile>`, for each profile).
4. Plan test execution by grouping configuration profiles into buckets.

After `openshift-tests` plans test orchestration, it will proceed to execute tests in
each configuration bucket. For a given bucket, `openshift-tests`
will:
1. Iterate through all involved components and configure them with non-conflicting configuration profiles (`config` verb).
2. Await a high-level cluster steady state (cluster operators available, machines config up-to-date) if any configuration profile requires it.
3. Run tests within the bucket with non-conflicting, test-level parallelization.

Between buckets, the component may be asked to set its required `default` profile in order
to return to its install-time configuration.

#### Update - Metadata Validation
Component owners will be responsible for implementing the extension interface. To prevent common mistakes
and ensure conformance with the evolution of their implementation, `make` (or similar build system)
must run `<extension binary> [component parameters] update [--basedir <basedir>]` after the extension binary is built.

The `update` verb will create or update files under `hack/.openshift-tests-exension/product/type/component/*`, by default
(basedir defaults to `hack/.openshift-tests-extension`).
If an incompatible change is introduced from the prior invocation of `update` (e.g. changing 
a test name without preserving the original), `update` will raise an error 
which the component owner must correct before committing their change in git.

The content of the files stored under `.openshift-tests-extension` is subject to change and wholly
defined by the `github.com/openshift-eng/openshift-tests-extension` module. Individual component
owners should not modify files under this path except through invocations of `update`.

#### Test Result Aggregation

`openshift-tests` will store test results in the CI artifacts repository in a JSONL file.
The output will be similar to that of the `run-test` output, but each test result 
will be expanded to include:
- Component & subcomponent
- Environment
- Testing Context

The goal here is to allow a single file to contain comprehensive information necessary for
a system like Component Readiness to analyze it without, for example, needing to draw information
from prowjob names.

```python
{
    # Human-readable test name
    "name": "openssl security compliance",
    
    # Original human-readable test name, if it has changed over time.
    "originalName": "fips security compliance",
    
    # The Test ID openshift/origin has defined for this test based
    # on component and test metadata. It is meant to be unique
    # across all other components and consistent across time
    # (even if the human-readable name for a unit of test logic
    # is changed).
    "id": "openshift-payload-api-server-fips security compliance",
    
    "result": "pass",

    # ...other test result information...
    
    # Information necessary to fully qualify the 
    # test.
    "component": {
        "product": "openshift",
        "type": "payload",
        "component": "hyperkube",
        "subcomponent": "logs",
    },
    
    "environment": {
        "platform": "aws",
        "architecture": "amd64",
        # ...others...
        
        # Configuration information is also included. 
        "configuration": {
            # The configuration id applied to the component
            # before the test was run.
            "component": "default",
        }
    },
    
    # Information about how the test was run
    "context": {
        # A small number of seeds can be used to randomize test
        # parallelization and bucketing. Analyzing seeds against
        # one another should highlight parallelization issues.
        # Using the same seed with the same tests should result
        # in identical planning and parallelization. 
        "seed": 15,
        
        # A sha256 of the tests IDs which were run. This, plus
        # seed should ensure identical execution pattern between
        # two runs.
        "testHash": "deadbeef",
        
        # In the future, we may run tests several times against
        # the same cluster in order to aggregate
        # test results more quickly. We may want to 
        # analyze results including this dimension if, for example,
        # we want to check whether the first run is as consistent
        # as the following N.
        "run": 0
    }
}
```

### Risks and Mitigations

#### Binary Incompatibility
Copying a binary from a different image into a running pod exposes us to binary incompatibility. 

##### CPU Architecture 
An arm64 binary will not run in an amd64 based pod. `openshift-tests` will need to run on 
a build farm node with a CPU architecture consistent with architecture of the cluster 
payload under test.

#### Runtime Size / Speed
Statically linked binaries are large relative to dynamically linked binaries. As the number
of extension binaries increase, so will the required tmpfs space used by pods running the
tests as those binaries are stored locally within the pod.

To mitigate this risk, we need to be aware of it, overprovision test pods with respect to 
the memory normal test execution requires, and also include a test failure that will 
TRT attention if the pod is using > 80% of allocatable memory.

#### Image Size
Statically linked binaries are relatively large. As the number of test extension binaries
increases, it may lead to a noticeable increase in overall payload size.

#### Poor Extension Implementation
Providing a contract that must be implemented without by test extensions without centralized 
control & review of the implementation leaves space for contributions that violate the 
expectations of that contract.

### Version Skew Strategy

The majority of test extension binaries will reside within the OpenShift payload under test, and thus cannot
skew. For non-payload components, test filtering, performed by the extension binary itself can take the current
platform version into account when determining appropriate tests to run.

## Alternatives

1. Run extension binaries from Pods/independent container images without extracting them to tmpfs.