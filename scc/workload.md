# Workload

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
  - [`labels`](#labels):
    - `apps.tanzu.vmware.com/workload-type`:
    - `app.kubernetes.io/part-of`:
  - [`annotations`](#annotations): foo

- `spec`:
  - [`source`](#source): The location of the source code for the workload
    - [`git`](#git-source):
      - `url`:
      - `ref`:
        - `branch`:
        - `commit`:
        - `tag`:
    - [`image`](#image-source)
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
    - `limits`
    - `requests`
  - `serviceAccountName`: Service account with permissions to create
    resources submitted by the supply chain. If not set, Cartographer will
    use serviceAccountName from supply chain. If that is also not set, 
    Cartographer will use the default service account in the workload's
    namespace.

- `status`:
  - `conditions`:
  - `supplyChainRef`:
  - `resources`:
  - `observedGeneration`:


## Labels

## Annotations

## Source

### Git source

### Image source

## Pre-built image

## Parameters

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

