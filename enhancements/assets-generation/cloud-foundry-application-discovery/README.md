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

Cloud Foundry (CF) is a platform-as-a-service solution that simplifies application deployment by abstracting infrastructure concerns. Applications on CF are deployed using manifests, which are YAML files specifying application properties, resource allocations, environment variables, and service bindings. These manifests provide a structured and declarative way to define the runtime and platform configurations required for an application.

Move2Kube is an IBM research project with the goal to provide tools to migrate applications to other platforms, particularly Kubernetes. One of the use cases of Move2Kube is to migrate Cloud Foundry applications to Kubernetes, but users have been struggling with the template language used to capture the Kubernetes resources, as it is not well known and requires an additional effort to master, compared to other templating languages.

## Motivation

The challenge brought by the templating language severely impacts the usability and acceptance of the tool. Thus the existence of this enhancement to provide a similar tool to MTA, extensible, and that improves on the templating engine, so that it offers a pluggable design that can be used to implement well known templating engines.

### Goals

* Identify and understand Cloud Foundry Application manifests (v3) fields.
* Extract and process Cloud Foundry application manifest into a new canonical form, capturing the intent of the original field value and
  with the foresight of the future application of the given field in a Kubernetes platform.
* Provide documentation for developers to understand the relationship between the original manifest and resulting canonical manifest.

### Non-Goals

* Transformation provider for Kubernetes using Helm Charts.

## Proposal

To migrate applications from Cloud Foundry to Kubernetes, it is essential to translate these manifests into an intermediate format, or canonical form, that captures the intent and configuration of the CF manifest. This intermediate manifest serves as a bridge, retaining critical deployment configurations while adapting them to Kubernetes-native practices. The format needs to be designed as platform-agnostic and compatible with multiple templating engines like Helm or Ansible, enabling flexibility in how the deployment configurations are generated and managed.

The following table depicts the relationship between the Cloud Foundry Application manifest fields and the proposed location in the canonical form manifest.

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

### Sidecar specification

| Name | Mapped  | Canonical Form | Description |
| ----- | :---: | ----- | ----- |
| **name** | Y | Name | Name of the sidecar |
| **process\_types** | Y | ProcessTypes | ProcessTypes captures the different process types defined for the sidecar. Compared to a Process, which has only one type, sidecar processes can accumulate more than one type. See [processtype specification](#processtype-specification).|
| **command** | Y | Command | Command to run this sidecar |
| **Memory** | Y | Memory | (Optional) The amount of memory to allocate to the sidecar. |

### Service specification

Maps to Spec.Services in the canonical form. Only \`name\` and \`parameters\` CF fields are captured since \`binding\_name\` is only used when defining a service and not applicable to a Kubernetes application.

| Name | Mapped  | Canonical Form | Description |
| ----- | :---: | ----- | ----- |
| **name** | Y | Name | Name of the service required by the application |
| **parameters** | Y | Parameters | key/value pairs for the application to use when connecting to the service. |

### Metadata specification

| Name | Mapped  | Canonical Form | Description |
| ----- | :---: | ----- | ----- |
| **Application.name** | Y | Name | Name is derived from the application’s Name field, which is stored in the metadata of the discovery manifest, following Kubernetes structured resources format. |
| **Space.name** | Y | Space | Captured at runtime only and it contains the name of the space where the application is deployed. |
| **labels** | Y | Labels |  Labels capture the labels as defined in the `labels` field in the CF application manifest |
| **annotations** | Y | Annotations | Annotations as defined in the `annotations` field in the CF application manifest |

### Process specification

| Name | Mapped | Canonical Form | Comments |
| ----- | :---: | ----- | ----- |
| **type** | Y | Type | Only web or worker types are supported. |
| **Application.docker** |  | Image | Pull specification of the container image. This field is derived from the docker’s field in the application spec. See [application specification](https://docs.google.com/document/d/1zYBWSe6WYQzv6eLozi6jBL-TYVpOKktuzv2no92tG30/edit?pli=1&tab=t.0#heading=h.5vxln8xxilyw). |
| **command** | Y | Command | The command used to start the process. |
| **disk\_quota** | Y | DiskQuota | Example: 1G unit of measurement: `B`, `K`, `KB`, `M`, `MB`, `G`, `GB`, `T`, or `TB` in upper case or lower case
Note: In CF, limit for all instances of the **web** process; |
| **memory** | Y | Memory | The value at the application level defines the default memory requirements for all processes in the application, when not specified by the process itself. The discovery process will consolidate the amount of memory specific to each process based on the information either in the application or the process fields. Example: 128MB unit of measurement: `B`, `K`, `KB`, `M`, `MB`, `G`, `GB`, `T`, or `TB` in upper case or lower case. Note: In CF, limit for all instances of the **web** process; |
| **health-check-http-endpoint** | Y  | Probe.Endpoint | health-check fields are captured in a Probe structure, common with the readiness-heath-check. See (Probe specification)[#probe-specification]. |
| **health-check-invocation-timeout** | Y | Probe.Timeout | See (Probe specification)[#probe-specification]. |
| **health-check-interval** | Y | Probe.Interval | See (Probe specification)[#probe-specification]. |
| **health-check-type** | N |  | Type of health check to perform; `none` is deprecated and an alias to `process` |
| **readiness-check-http-endpoint** | Y | Probe.Endpoint | See (Probe specification)[#probe-specification]. |
| **readiness-check-invocation-timeout** | Y | Probe.Timeout | See (Probe specification)[#probe-specification]. |
| **readiness-check-interval** | Y | Probe.Interval | See (Probe specification)[#probe-specification]. |
| **readiness-health-check-type** | N |  | Type of health check to perform; `none` is deprecated and an alias to `process` |
| **instances** | Y | Replicas | This field determines how many instances of the process will run in the application. |
| **log-rate-limit-per-second** | Y | LogRateLimit | The log rate limit for all the instances of the process; unit of measurement: `B`, `K`, `KB`, `M`, `MB`, `G`, `GB`, `T`, or `TB` in upper case or lower case, or -1 or 0 |

### ProcessType specification

Represents a single process type as a string. Possible values are `worker`, or `web`.

## Probe specification

| Name | Mapped  | Canonical Form | Description |
| ----- | :---: | ----- | ----- |
| **health-check-http-endpoint** | Y | Endpoint | HTTP endpoint to be used for health checks, specifying the path to be monitored. |
| **health-check-invocation-timeout** | Y | Timeout | Maximum time allowed for each health check invocation to complete. |
| **health-check-interval** | Y | Interval | Interval at which health checks are performed to monitor the application’s status. |
| **health-check-type** | N |  | Specifies the type of health check to perform (`none`, `http`, `tcp`, or `process`). Note: `none` is deprecated and an alias for process. |

## Route specification

Captures the name of the route that will be shown as hostname.

By default, the route URL is set using the application name as the hostname unless the `no-route` field is set to `true` or a route URL is explicitly defined.
Processes of the `worker` type are not designed to have any ports open.
If the application has globally defined routes, processes of the `web` type inherit the routes specified in that field.
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

If applicable, how will the component be upgraded and downgraded? Make sure this
is in the test plan.

Consider the following in developing an upgrade/downgrade strategy for this
enhancement:
- What changes (in invocations, configurations, API use, etc.) is an existing
  cluster required to make on upgrade in order to...
  -  keep previous behavior?
  - make use of the enhancement?

## Implementation History

Major milestones in the life cycle of a proposal should be tracked in `Implementation
History`.

## Drawbacks

The idea is to find the best form of an argument why this enhancement should _not_ be implemented.

## Alternatives

Similar to the `Drawbacks` section the `Alternatives` section is used to
highlight and record other possible approaches to delivering the value proposed
by an enhancement.

## Infrastructure Needed [optional]

Use this section if you need things from the project. Examples include a new
subproject, repos requested, github details, and/or testing infrastructure.

Listing these here allows the community to get the process for these resources
started right away.
