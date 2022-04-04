# Workload

Workload allows developers to pass information about the app to be delivered
through the supply chain. Most of the fields are used regardless of the supply
chain selected, but there are a few behaviors that are specific to the supply
chains shipped via Out of the Box packages.

Here you'll find reference documentation about the Workload custom resource
definition with regards to the supply chains shipped by TAP. For general
reference documentation, see
https://cartographer.sh/docs/development/reference/workload/#workload.


## Reference

- `apiVersion`: `carto.run/v1alpha1`
- `kind`: `Workload`
- `metadata`:
  - `name`: name of the workload
  - `namespace`: namespace at which the Workload should be created at. In
    this namespace all the auxiliary objects used by the resources managed by
    the supply chain must exist (e.g., serviceaccount that points at the image
    registry credentials, any scan policies, tekton pipelines for running tests,
    etc).
  - [`labels`](#labels): set of labels to be used to matching the Workload with
    a supply chain as well as be passed down to children objects
    - `apps.tanzu.vmware.com/workload-type`
    - `app.kubernetes.io/part-of`
  - [`annotations`](#annotations): foo
- `spec`
  - [`source`](#source): The location of the source code for the workload
    - [`git`](#git-source): Source code location in a git repository.
      - `url`
      - `ref`
        - `branch`
        - `commit`
        - `tag`
    - `subPath`: Subpath inside the Git repository or Image to treat as the
      root of the application. Defaults to the root if left empty.
    - [`image`](#image-source): OCI Image in a repository, containing the
      source code to be used throughout the supply chain.
  - [`image`](#pre-built-image): A pre-built image in a registry. It is an alternative to
    specifying the location of source code for the workload. Specify one of
    `spec.source` or `spec.image`
  - `build`: Build configuration, for the build resources in the supply chain
    - `env`: Environment variables to be passed to the builder of the
      application container image
  - `env`: Environment variables to be passed to the main container running
    the application
  - [`params`](#parameters): Additional parameters specific to the supply chains
  - [`serviceClaims`](#serviceclaims): ServiceClaims to be bound through ServiceBindings.
  - `resources`: Resource constraints for the application
    - `limits`:
    - `requests`:
  - `serviceAccountName`: Service account with permissions to create
    resources submitted by the supply chain. If not set, Cartographer will
    use serviceAccountName from supply chain. If that is also not set, 
    Cartographer will use the default service account in the workload's
    namespace.
- `status`
  - `conditions`: describe this resource's reconcile state. The top level
    condition is of type `Ready`, and follows [Kubernetes conditions
    conventions][kubernetes-api-conventions]:
  - `observedGeneration`: the generation of the spec that resulted in the
    current `status`.
  - `supplyChainRef`: the Supply Chain resource that was used when this status
    was set.
  - `resources`: references to the objects created by the Supply Chain and the
    templates used to create them, as well as inputs and outputs that were
    passed between the templates as the Supply Chain was processed.
    - `name`: name of the resource in the blueprint
    - `inputs`: references to resources that were used to template the object
      in StampedRef
    - `outputs`: values from the object in StampedRef that can be consumed by
      other resources
      - `name`: output type generated from the resource (url, revision, image
        or config)
      - `preview`: preview of the value of the output (limited to 1024 bytes)
      - `lastTransitionTime`: timestamp of the last time the value changed
      - `digest`: sha256 of the full value of the output
    - `stampedRef`: reference to the object that was created by the resource
    - `templateRef`: reference to the template used to create the object in
      StampedRef

[kubernetes-api-conventions]: https://github.com/kubernetes/community/blob/master/contributors/devel/sig-architecture/api-conventions.md#typical-status-properties


## Labels

- passed down to children
- `has-tests` for the supply chians that run tests (testing and scanning based
  when providing sources)
  - must always have `type: web`
  - might use `has-tests` depending on the goal and supply chain installed


## Annotations

- the knative one gets passed down to the knative service? be more exact about
  what goes down

- param `annotations`


## Source

can come from either git or local directory

### Git source

see more at [git source](./building-from-source.md#git-source).

### Image source

see more at [local source](./building-from-source.md#local-source).

## Pre-built image

rather than providing source code to be built, it's possible to bring your own
image.

see more at [pre-built image](./pre-build-image.md).


## Parameters

the ootb packages bring supply chains with pre-configured defaults, but some of
those configurations can be tweaked by the one submitting the Workload to the
cluster.

... where do we provid


## ServiceClaims


## Examples
     


- parameters
  - `gitops_...`
  - annotations for knative

- source configuration

- pre-built image

- build environment variables

- runtime environment variables

- runtime resources (limits and requests)

- debugging


## spec

## status

- conditions
- resources
- supplychainref .. ?

