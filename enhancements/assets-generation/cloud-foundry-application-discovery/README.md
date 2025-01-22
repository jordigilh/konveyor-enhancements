---
title: cloud-foundry-application-discovery
authors:
  - "@jordigilh"
  - "@gciavarrini"
reviewers:
  - "@dymurray"
  - "@jortel"
  - "@eemcmullan"
  - "@JonahSussman"
  - "@rromannissen"
  - "@savitharaghunathan"
approvers:
  - "@dymurray"
  - "@jortel"
  - "@eemcmullan"
  - "@JonahSussman"
  - "@rromannissen"
  - "@savitharaghunathan"
creation-date: 2025-01-17
last-updated: 2025-01-17
status: provisional
see-also:
  - https://github.com/konveyor/enhancements/pull/210
replaces:
superseded-by:
---

# Cloud Foundry Application Discovery to canonical form

This is the title of the enhancement. Keep it simple and descriptive. A good
title can help communicate what the enhancement is and should be considered as
part of any review.

The YAML `title` should be lowercased and spaces/punctuation should be
replaced with `-`.

To get started with this template:
1. **Pick a domain.** Find the appropriate domain to discuss your enhancement.
1. **Make a copy of this template.** Copy this template into the directory for
   the domain.
1. **Fill out the "overview" sections.** This includes the Summary and
   Motivation sections. These should be easy and explain why the community
   should desire this enhancement.
1. **Create a PR.** Assign it to folks with expertise in that domain to help
   sponsor the process.
1. **Merge at each milestone.** Merge when the design is able to transition to a
   new status (provisional, implementable, implemented, etc.). View anything
   marked as `provisional` as an idea worth exploring in the future, but not
   accepted as ready to execute. Anything marked as `implementable` should
   clearly communicate how an enhancement is coded up and delivered. If an
   enhancement describes a new deployment topology or platform, include a
   logical description for the deployment, and how it handles the unique aspects
   of the platform. Aim for single topic PRs to keep discussions focused. If you
   disagree with what is already in a document, open a new PR with suggested
   changes.

The `Metadata` section above is intended to support the creation of tooling
around the enhancement process.

## Release Signoff Checklist

- [ ] Enhancement is `implementable`
- [ ] Design details are appropriately documented from clear requirements
- [ ] Test plan is defined
- [ ] User-facing documentation is created

## Open Questions [optional]

This is where to call out areas of the design that require closure before deciding
to implement the design.  For instance,
 > 1. This requires exposing previously private resources which contain sensitive
  information.  Can we do this?

## Summary

Cloud Foundry (CF) is a platform-as-a-service solution that simplifies application
deployment by abstracting infrastructure concerns. Applications on CF are deployed
using manifests, which are YAML files specifying application properties, resource
allocations, environment variables, and service bindings. These manifests provide
a structured and declarative way to define the runtime and platform configurations
required for an application.

Move2Kube is an IBM research project with the goal to provide tools to migrate
applications to other platforms, particularly Kubernetes. One of the use cases
of Move2Kube is to migrate Cloud Foundry applications to Kubernetes, but users
have been struggling with the template language used to capture the Kubernetes
resources, as it is not well known and requires an additional effort to master,
 compared to other templating languages.

## Motivation

The challenge brought by the templating language severely impacts the usability
and acceptance of the tool. Thus the existence of this enhancement to provide a
similar tool to MTA, extensible, and that improves on the templating engine, so
that it offers a pluggable design that can be used to implement well known
templating engines.

### Goals

* Identify and understand Cloud Foundry Application manifests (v3) fields.
* Extract and process Cloud Foundry application manifest into a new canonical
  form, capturing the intent of the original field value and with the foresight
  of the future application of the given field in a Kubernetes platform.
* Provide documentation for developers to understand the relationship between
  the original manifest and resulting canonical manifest.

### Non-Goals

* Transformation provider for Kubernetes using Helm Charts.

## Proposal

To migrate applications from Cloud Foundry to Kubernetes, it is essential to
translate these manifests into an intermediate format, or canonical form, that
captures the intent and configuration of the CF manifest. This intermediate
manifest serves as a bridge, retaining critical deployment configurations while
adapting them to Kubernetes-native practices. The format needs to be designed 
as platform-agnostic and compatible with multiple templating engines like Helm
or Ansible, enabling flexibility in how the deployment configurations are
generated and managed.

The following table depicts the relationship between the Cloud Foundry Application
manifest fields and the proposed location in the canonical form manifest.

### Space specification

| Name | Mapped (Y/N) | Canonical Form | Description |
| ----- | :---: | ----- | ----- |
| **applications** | N |  | Direct mapping to a slice of canonical form manifests, each one representing the discovery results of a CF application. See [app-level specification](#application-specification) |
| **space** | Y | Metadata.Space | See [metadata specification](#metadata-specification). This field is only populated at runtime. |
| **version** | N |  | The manifest schema version; currently the only valid version is 1, defaults to 1 if not provided |

### Application specification

| Name | Mapped  | Canonical Form | Comments |
| ----- | :---: | ----- | ----- |
| **name** | Y | Metadata.Name | Name is derived from the application’s Name field, which is stored in the metadata of the discovery manifest, following Kubernetes structured resources format.
See [metadata specification](#metadata-specification). |
| **buildpacks** | N |  | These fields in CF specify how to build your application (e.g., "nodejs\_buildpack", "java\_buildpack"). The canonical form should focus on "what" to deploy, not "how" to build it |
| **docker** | Y | Process.Image | The value of the docker image pullspec is captured for each \`Process\` in the `Image` field. See [process specification](#process-specification). |
| **env** | Y | Env | Direct mapping from the application's `Env` field |
| **no-route** | Y | Routes | Processes will have no route information in the canonical form manifest. See [route specification](#route-specification). |
| **processes** | Y | Processes | See [process specification](#process-specification) |
| **random-route** | Y | Routes | See [route specification](#route-specification). |
| **routes** | Y | Routes | See [route specification](#route-specification). |
| **services** | Y | Services | See [service specification](#service-specification). |
| **sidecars** | Y | Sidecars | See [sidecar specification](#sidecar-specification). |
| **metadata** | Y | Metadata | See [metadata specification](#metadata-specification). |
| **buildpack** | N |  | Already deprecated in CF. See buildpacks. |
| **timeout** | Y | StartupTimeout | Maximum time allowed for an application to respond to readiness or health checks during startup.If the application does not respond within this time, the platform will mark the deployment as failed. |
| **instances** | Y | Replicas | Number of CF application instances |
| **stack** | Y | Stack | Stack is derived from the `stack` field in the application manifest. The value is captured for information purposes because it has no relevance in Kubernetes. |

```go
type Application struct {
  // Metadata captures the name, labels and annotations in the application.
  Metadata Metadata `json:",inline"`
  // Env captures the `env` field values in the CF application manifest.
  Env map[string]string `json:"env,omitempty"`
  // Routes represent the routes that are made available by the application.
  Routes Routes `json:"routes,omitempty"`
  // Services captures the `services` field values in the CF application manifest.
  Services Services `json:"services,omitempty"`
  // Processes captures the `processes` field values in the CF application manifest.
  Processes Processes `json:"processes,omitempty"`
  // Sidecars captures the `sidecars` field values in the CF application manifest.
  Sidecars Sidecars `json:"sidecars,omitempty"`
  // Stack represents the `stack` field in the application manifest.
  // The value is captured for information purposes because it has no relevance
  // in Kubernetes.
  Stack string `json:"stack,omitempty"`
  // StartupTimeout specifies the maximum time allowed for an application to
  // respond to readiness or health checks during startup.
  // If the application does not respond within this time, the platform will mark
  // the deployment as failed. The default value is 60 seconds.
  // https://github.com/cloudfoundry/docs-dev-guide/blob/96f19d9d67f52ac7418c147d5ddaa79c957eec34/deploy-apps/  large-app-deploy.html.md.erb#L35
  StartupTimeout uint `json:"startupTimeout,omitempty"`
}
```

### Sidecar specification

| Name | Mapped  | Canonical Form | Description |
| ----- | :---: | ----- | ----- |
| **name** | Y | Name | Name of the sidecar |
| **process\_types** | Y | ProcessTypes | ProcessTypes captures the different process types defined for the sidecar. Compared to a Process, which has only one type, sidecar processes can accumulate more than one type. See [processtype specification](#processtype-specification).|
| **command** | Y | Command | Command to run this sidecar |
| **Memory** | Y | Memory | (Optional) The amount of memory to allocate to the sidecar. |

```go
type Sidecar struct {
  // Name represents the name of the Sidecar
  Name string `json:"name"`
  // ProcessTypes captures the different process types defined for the sidecar.
  // Compared to a Process, which has only one type, sidecar processes can
  // accumulate more than one type.
  ProcessTypes ProcessTypes `json:"processTypes"`
  // Command captures the command to run the sidecar
  Command []string `json:"command"`
  // Memory represents the amount of memory to allocate to the sidecar.
  // It's an optional field.
  Memory string `json:"memory,omitempty"`
}
```

### Service specification

Maps to Spec.Services in the canonical form. Only \`name\` and \`parameters\` CF fields are captured since \`binding\_name\` is only used when defining a service and not applicable to a Kubernetes application.

| Name | Mapped  | Canonical Form | Description |
| ----- | :---: | ----- | ----- |
| **name** | Y | Name | Name of the service required by the application |
| **parameters** | Y | Parameters | key/value pairs for the application to use when connecting to the service. |

```go
type Service struct {
  // Name represents the name of the Cloud Foundry service required by the
  // application. This field represents the runtime name of the service, captured
  // from the 3 different cases where the service name can be listed.
  // For more information check https://docs.cloudfoundry.org/devguide/deploy-apps/manifest-attributes.html#services-block
  Name string `json:"name"`
  // Parameters contain the k/v relationship for the aplication to bind to the service
  Parameters map[string]interface{} `json:"parameters,omitempty"`
}
```

### Metadata specification

| Name | Mapped  | Canonical Form | Description |
| ----- | :---: | ----- | ----- |
| **Application.name** | Y | Name | Name is derived from the application’s Name field, which is stored in the metadata of the discovery manifest, following Kubernetes structured resources format. |
| **Space.name** | Y | Space | Captured at runtime only and it contains the name of the space where the application is deployed. |
| **labels** | Y | Labels |  Labels capture the labels as defined in the `labels` field in the CF application manifest |
| **annotations** | Y | Annotations | Annotations as defined in the `annotations` field in the CF application manifest |

```go
type Metadata struct {
  // Name capture the `name` field int CF application manifest
  Name string `json:"name"`
  // Space captures the `space` where the CF application is deployed at runtime. The field is empty if the
  // application is discovered directly from the CF manifest. It is equivalent to a Namespace in Kubernetes.
  Space string `json:"space,omitempty"`
  // Labels capture the labels as defined in the `annotations` field in the CF application manifest
  Labels map[string]string `json:"labels,omitempty"`
  // Annotations capture the annotations as defined in the `labels` field in the CF application manifest
  Annotations map[string]string `json:"annotations,omitempty"`
}
```

### Process specification

| Name | Mapped | Canonical Form | Comments |
| ----- | :---: | ----- | ----- |
| **type** | Y | Type | Only web or worker types are supported. |
| **Application.docker** |  | Image | Pull specification of the container image. This field is derived from the docker’s field in the application spec. See [application specification](https://docs.google.com/document/d/1zYBWSe6WYQzv6eLozi6jBL-TYVpOKktuzv2no92tG30/edit?pli=1&tab=t.0#heading=h.5vxln8xxilyw). |
| **command** | Y | Command | The command used to start the process. |
| **disk\_quota** | Y | DiskQuota | Example: 1G unit of measurement: `B`, `K`, `KB`, `M`, `MB`, `G`, `GB`, `T`, or `TB` in upper case or lower case
Note: In CF, limit for all instances of the **web** process; |
| **memory** | Y | Memory | The value at the application level defines the default memory requirements for all processes in the application, when not specified by the process itself. The discovery process will consolidate the amount of memory specific to each process based on the information either in the application or the process fields. Example: 128MB unit of measurement: `B`, `K`, `KB`, `M`, `MB`, `G`, `GB`, `T`, or `TB` in upper case or lower case. Note: In CF, limit for all instances of the **web** process; |
| **health-check-http-endpoint** | Y  | Probe.Endpoint | health-check fields are captured in a Probe structure, common with the readiness-heath-check. See [Probe specification](#probe-specification). |
| **health-check-invocation-timeout** | Y | Probe.Timeout | See [Probe specification](#probe-specification). |
| **health-check-interval** | Y | Probe.Interval | See [Probe specification](#probe-specification). |
| **health-check-type** | N |  | Type of health check to perform; `none` is deprecated and an alias to `process` |
| **readiness-check-http-endpoint** | Y | Probe.Endpoint | See [Probe specification](#probe-specification). |
| **readiness-check-invocation-timeout** | Y | Probe.Timeout | See [Probe specification](#probe-specification). |
| **readiness-check-interval** | Y | Probe.Interval | See [Probe specification](#probe-specification). |
| **readiness-health-check-type** | N |  | Type of health check to perform; `none` is deprecated and an alias to `process` |
| **instances** | Y | Replicas | This field determines how many instances of the process will run in the application. |
| **log-rate-limit-per-second** | Y | LogRateLimit | The log rate limit for all the instances of the process; unit of measurement: `B`, `K`, `KB`, `M`, `MB`, `G`, `GB`, `T`, or `TB` in upper case or lower case, or -1 or 0 |

```go
type Process struct {
  // Type captures the `type` field in the Process specification.
  // Accepted values are `web` or `worker`
  Type ProcessType `json:"type,omitempty"`
  // Image represents the pull spec of the container image.
  Image string `json:"image"`
  // Command represents the command used to run the process.
  Command []string `json:"command,omitempty"`
  // DiskQuota represents the amount of persistent disk requested by the process.
  DiskQuota string `json:"disk,omitempty"`
  // Memory represents the amount of memory requested by the process.
  Memory string `json:"memory,omitempty"`
  // HealthCheck captures the health check information
  HealthCheck Probe `json:"healthCheck"`
  // ReadinessCheck captures the readiness check information.
  ReadinessCheck Probe `json:"readinessCheck"`
  // Replicas represents the number of instances for this process to run.
  Replicas uint `json:"replicas"`
  // LogRateLimit represents the maximum amount of logs to be captured per second.
  LogRateLimit string `json:"logRateLimit,omitempty"`
}
```

### ProcessType specification

Represents a single process type as a string. Possible values are `worker`, or `web`.

```go
type ProcessType string

const (
  // Web represents a `web` application type
  Web ProcessType = "web"
  // Worker represents a `worker` application type
  Worker ProcessType = "worker"
)
```

## Probe specification

| Name | Mapped  | Canonical Form | Description |
| ----- | :---: | ----- | ----- |
| **health-check-http-endpoint** | Y | Endpoint | HTTP endpoint to be used for health checks, specifying the path to be monitored. |
| **health-check-invocation-timeout** | Y | Timeout | Maximum time allowed for each health check invocation to complete. |
| **health-check-interval** | Y | Interval | Interval at which health checks are performed to monitor the application’s status. |
| **health-check-type** | N |  | Specifies the type of health check to perform (`none`, `http`, `tcp`, or `process`). Note: `none` is deprecated and an alias for process. |

```go
type Probe struct {
  // Endpoint represents the URL location where to perform the probe check.
  Endpoint string `json:"endpoint"`
  // Timeout represents the number of seconds in which the probe check can be considered as timedout.
  // https://docs.cloudfoundry.org/devguide/deploy-apps/manifest-attributes.html#timeout
  Timeout uint `json:"timeout"`
  // Interval represents the number of seconds between probe checks.
  Interval uint `json:"interval"`
}
```

## Route specification

Captures the name of the route that will be shown as hostname.

By default, the route URL is set using the application name as the hostname
unless the `no-route` field is set to `true` or a route URL is explicitly defined.
Processes of the `worker` type are not designed to have any ports open.
If the application has globally defined routes, processes of the `web` type
inherit the routes specified in that field. \
Examples:
\---

	...
	routes:
	\- route: example.com
	  protocol: http2
	\- route: www.example.com/foo
	\- route: tcp-example.com:1234

| Name | Mapped  | Canonical Form | Description |
| ----- | :---: | ----- | ----- |
| **url** | Y | URL  | `URL as defined in the route field value.`  |
| **protocol** | Y | Protocol | It can be `HTTP`, `HTTPS` or `TCP`. |

```go
type Route struct {
  // URL captures the FQDN, path and port of the route.
  URL string `json:"url"`
  // Protocol captures the protocol type: http, http2 or tcp. Note that the CF `protocol` field is only available
  // for CF deployments that use HTTP/2 routing.
  Protocol RouteProtocol `json:"protocol"`
}

type RouteProtocol string

const (
	HTTP  RouteProtocol = "http"
	HTTPS RouteProtocol = "https"
	TCP   RouteProtocol = "tcp"
)
```

### User Stories [optional]

* As a DevOps engineer migrating multiple applications, I want an intermediate
  canonical form that abstracts the complexities of Cloud Foundry manifests,
  so that I can easily adapt and reuse it for Kubernetes deployment across
  various platforms.

* As a DevOps engineer migrating multiple applications, I want to clearly
  understand the relationship between Cloud Foundry manifest fields and their
  canonical equivalents, so that I can confidently map my application's
  configurations to a Kubernetes-native environment.

* As a DevOps engineer migrating an application with complex configurations, I
  want the migration tool to preserve the intent and critical settings of the
  original Cloud Foundry manifest, so that my application behaves consistently after migration.

* As a DevOps engineer migrating applications, I want the migration tool to
  focus only on generating a canonical form, so that I can independently choose
  how to apply the canonical form using my preferred Kubernetes templating approach.

### Implementation Details/Notes/Constraints [optional]

What are the caveats to the implementation? What are some important details that
didn't come across above. Go in to as much detail as necessary here. This might
be a good place to talk about core concepts and how they relate.

### Security, Risks, and Mitigations

**Carefully think through the security implications for this change**

_What are the risks of this proposal and how do we mitigate. Think broadly. How_
_will this impact the broader OKD ecosystem? Does this work in a managed services_
_environment that has many tenants?_

_How will security be reviewed and by whom? How will UX be reviewed and by whom?_

_Consider including folks that also work outside your immediate sub-project._

#### Canonical Manifest Misalignment
CF provides features that are not directly mappable to Kubernetes-native
concepts, requiring additional work during the migration process to ensure
compatibility.

- *Buildpacks* \
  Cloud Foundry utilizes buildpacks to handle application dependencies and
  runtime environments, which do not have a direct equivalent in Kubernetes.
  This means that applications relying on buildpacks will require additional
  configuration or alternative solutions when migrating to Kubernetes.
- *Docker Secrets* \
  In CF, secrets management is integrated into the platform, while Kubernetes
  uses a more granular approach with its own secrets management system. This
  discrepancy necessitates a different handling method for sensitive information
  during migration.
- *Services* \
  Services in CF are managed differently than in Kubernetes. For
  instance, CF abstracts many operational concerns, while Kubernetes requires
  explicit configuration for service discovery and networking. This difference
  can lead to complexities during migration as developers must adapt to the
  Kubernetes model of service management.


*Mitigation*

Users will have to do due diligence before deploying on K8S, including the
creation of resources prior to the deployment of the K8S assets, such as
`namespaces`, `services`, `secrets` and container images for `buildpacks`,
for instance. \
Developers may need to refactor applications or implement new solutions that
align with Kubernetes practices.

## Design Details

### Test Plan

**Note:** *Section not required until targeted at a release.*

Consider the following in developing a test plan for this enhancement:
- Will there be e2e and integration tests, in addition to unit tests?
- How will it be tested in isolation vs with other components?

No need to outline all of the test cases, just the general strategy. Anything
that would count as tricky in the implementation and anything particularly
challenging to test should be called out.

All code is expected to have adequate tests (eventually with coverage
expectations).

### Upgrade / Downgrade Strategy

N/A

## Implementation History

- January 2025: Proposal created and approved.
- (Tentative) February-March 2025: Initial MVP with support for CF manifest located in the filesystem where the CLI runs.
  Some limitations might apply based on the use cases to implement first.
- (Tentative) April 2025: Planned tech-preview release.

## Drawbacks

- M2K already provides discovery for Cloud Form application manifests. We could extend
  it to reuse the discovery logic and avoid the cost of redoing what is already provided.
- Breaks with existing M2K CLI support for CF application discovery. Some existing discovery
  functionality might not be supported in the short term and users will have to migrate their
  templates to the new Helm based template engine.

## Alternatives

- Reuse the existing M2K functionality for discovery, at the expense of using a code that is not ideal to manage
  and we are not a principal stakeholder.

## Infrastructure Needed [optional]

- CI/CD pipelines for building and unit/integration testing.
- CI/CD pipelines for E2E testing for QE and releasing.
- Hosting provider for binaries and documentation for releases.
- Project Management tools for tasks, issues and bugs.
- If REST API discovery is implemented, a Korifi instance on a Kubernetes cluster for E2E
  testing with a suite of samples that cover the acceptance test criteria.