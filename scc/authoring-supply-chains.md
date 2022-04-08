# Authoring Supply Chains

The Out of the Box Supply Chain, Delivery Basic and Templates packages provide
a set of Kubernetes objects aiming at covering a reference path to production
prescribed by VMware, but recognizing that each organization will have their
own needs, any of those objects can be customized, from the individual
templates for each resource, to the whole supply chains and delivery objects.

Depending on how TAP has been installed, there will be different ways of
achieving such customization of the out of the box supply chains.

Below you'll find sections covering different setups and how to proceed.

## Providing your own supply chain

To create a supply chain from scratch and make it available for Workloads, all
that's required is making sure that the supply chain does not conflict with the
ones that are installed in the cluster, after all, those objects are
cluster-scoped.

If this is your first time creating one, make sure to follow the tutorials from
the Cartographer documentation:
https://cartographer.sh/docs/v0.3.0/tutorials/first-supply-chain/

That said, it's important to observe that any supply chain installed in a TAP
cluster might suffer with two possible cases of collisions:

- **object name**: as mentioned before, supply chains (ClusterSupplyChain
  resource) are cluster scoped (just like any Cartographer resource prefixed
  with `Cluster`), so the name of the custom supply chain must be different
  from the ones the Out of the Box packages provide.

  Either create a supply chain whose name is different, or remove the
  installation of the corresponding `ootb-supply-chain-*` from the TAP.

- **workload selection**: a Workload gets reconciled against a particular
  supply chain based on a set of selection rules as defined by the supply
  chains. If the rules for the supply chain to match a Workload is ambiguous,
  the Workload will not make any progress.

  Either create a supply chain whose selection rules are different from the
  ones used by the Out of the Box Supply Chain packages, or remove the 
  installation of the corresponding `ootb-supply-chain-*` from TAP.
  
  See [Selectors](https://cartographer.sh/docs/v0.3.0/architecture/#selectors)
  to know more about it.

Currently (TAP 1.1), the following selection rules are in place for the
supply chains of the corresponding packages:

- _ootb-supply-chain-basic_
  - ClusterSupplyChain/**basic-image-to-url**
     - label `apps.tanzu.vmware.com/workload-type: web`
     - `workload.spec.image` field set
  - ClusterSupplyChain/**source-to-url**
     - label `apps.tanzu.vmware.com/workload-type: web`

- _ootb-supply-chain-testing_
  - ClusterSupplyChain/**testing-image-to-url**
     - label `apps.tanzu.vmware.com/workload-type: web`
     - `workload.spec.image` field set
  - ClusterSupplyChain/**source-test-to-url**
     - label `apps.tanzu.vmware.com/workload-type: web`
     - label `apps.tanzu.vmware.com/has-test: true`

- _ootb-supply-chain-testing-scanning_
  - ClusterSupplyChain/**scanning-image-scan-to-url**
     - label `apps.tanzu.vmware.com/workload-type: web`
     - `workload.spec.image` field set
  - ClusterSupplyChain/**source-test-scan-to-url**
     - label `apps.tanzu.vmware.com/workload-type: web`
     - label `apps.tanzu.vmware.com/has-test: true`


## Preventing TAP supply chains from being installed

Just like any other package, Using TAP profiles we can prevent supply chains
from being installed you can make use of the `excluded_packages` property in
`tap-values.yml`. For instance:

```yaml
# add to exclued_packages `ootb-*` packages you DON'T want to install
# 
excluded_packages:
  - ootb-supply-chain-basic.apps.tanzu.vmware.com
  - ootb-supply-chain-testing.apps.tanzu.vmware.com
  - ootb-supply-chain-testing-scanning.apps.tanzu.vmware.com

# comment out remove the `supply_chain` property
#
# supply_chain: ""
```

With the profile configured to not install the supply chains, there will be no
TAP-originated ClusterSupplyChain objects in the cluster.


## Modifying a supplychain from ootb-supply-chain-

In case either the shape of a supply chain or the templates that it points at
should be changed, a few steps should be followed.

1. copy one of the reference supply chains
1. remove the old one (add to `excluded_packages`)
1. modify the supply chain object
1. submit to the cluster

### Example

Suppose that we have a new `ClusterImageTemplate` object named `foo` that we
want use for building container images instead of the out of the box one that
makes use of Kpack for that, and the supply chain that we want to apply such
modification to is the `source-to-url` provided by the
`ootb-supply-chain-basic` package.

1. Find out the image that contains the supply chain definition

    ```bash
    kubectl get app ootb-supply-chain-basic \
      -n tap-install \
      -o jsonpath={.spec.fetch[0].imgpkgBundle.image}
    ```
    ```console
    registry.tanzu.vmware.com/tanzu-application-platform/tap-packages@sha256:f2ad401bb3e850940...
    ```

1. Pull the contents of the bundle into a directory named `ootb-supply-chain-basic`

    ```bash
    imgpkg pull \
      -b registry.tanzu.vmware.com/tanzu-application-platform/tap-packages@sha256:f2ad401bb3e850940... \
      -o ootb-supply-chain-basic
    ```
    ```console
    Pulling bundle 'registry.tanzu.vmware.com/tanzu-...
      Extracting layer 'sha256:542f2bb8eb946fe9d2c8a...

    Locating image lock file images...
    The bundle repo (registry.tanzu.vmware.com/tanzu...

    Succeeded
    ```

1. Inspect the files obtained

    ```bash
    tree ./ootb-supply-chain-basic/
    ```
    ```console
    ./ootb-supply-chain-basic/
    ├── config
    │   ├── supply-chain-image.yaml
    │   └── supply-chain.yaml
    └── values.yaml
    ```

1. Modify the desired supply chain to swap the template with another

    ```diff
    --- a/supply-chain.yaml
    +++ b/supply-chain.yaml
    @@ -52,7 +52,7 @@ spec:
       - name: image-builder
         templateRef:
           kind: ClusterImageTemplate
    -      name: kpack-template
    +      name: foo
         params:
           - name: serviceAccount
             value: #@ data.values.service_account
    ```

4. Submit the supply chain to Kubernetes

    The supply chain definition found in the bundle expects some values (the
    ones one provide via `tap-values.yml`) to be interpolated via YTT before
    being submitted to Kubernetes, so before applying the modified supply chain
    to the cluster, use YTT to interpolate those values and then apply:

    ```bash
    ytt \
      --ignore-unknown-comments \
      --file ./ootb-supply-chain-basic/config \
      --data-value registry.server=REGISTRY-SERVER \
      --data-value registry.repository=REGISTRY-REPOSITORY |
      kubectl apply -f-
    ```

    > **Note:** the modified supply chain will not outlive the destruction of
    > the cluster. It's recommended that it gets saved somewhere (e.g., a git
    > repsository) to be installed on every cluster where the supply chain is
    > expected to exist.


## Modifying a template from ootb-templates

The Out of the Box Templates package (`ootb-templates`) concentrates all the
templates and shared Tekton Tasks used by the supply chains shipped via
`ootb-supply-chain-*` packages.

Any templates that one wishes to modify (for instance, to change details about
the resources that are created based on them) would be found as part of this
package.

The workflow for getting any of them updated is as follows:

1. fetch the contents of `ootb-templates`
1. copy and create a new version of any of the templates you wish to modify
   (i.e., copy the contents and create template objects with new names)
1. submit the template to Kubernetes
1. modify the supply chain that makes use of the templates to point at the new
   ones instead



### Example

Suppose that we ...

1. Find out the image that contains the templates

    ```bash
    kubectl get app ootb-templates \
      -n tap-install \
      -o jsonpath={.spec.fetch[0].imgpkgBundle.image}
    ```
    ```console
    registry.tanzu.vmware.com/tanzu-application-platform/tap-packages@sha256:a5e177f38d7287f2ca7ee2afd67ff178645d8f1b1e47af4f192a5ddd6404825e
    ```

3. Pull the contents of the bundle into a directory named `ootb-templates`

    ```bash
    imgpkg pull \
      -b registry.tanzu.vmware.com/tanzu-application-platform/tap-packages@sha256:a5e177f38d7.. \
      -o ootb-templates
    ```
    ```
    ```

4. Observe that we've downloaded all the templates

    ```bash
    tree ./ootb-templates
    ```
    ```console
    ./ootb-templates
    ├── config
    │   ├── cluster-roles.yaml
    │   ├── config-template.yaml
    │   ├── config-writer-template.yaml
    │   ├── convention-template.yaml
    │   ├── deliverable-template.yaml
    │   ├── delivery-source-template.yaml
    │   ├── deployment-template.yaml
    ...
    │   └── testing-pipeline.yaml
    └── values.yaml
    ```




## TAP Profiles

When installing TAP making use of profiles, a `PackageInstall` object is
created, which in turn creates a whole set of children `PackageInstall` objects
for the installation of each individual component that makes up the platform.

```
PackageInstall/tap
└─App/tap
  ├─ PackageInstall/cert-manager
  ├─ PackageInstall/cartographer
  ├─ ...
  └─ PackageInstall/tekton-pipelines
```

Being the installation based on Kubernetes primitives, PackageInstall will
relentlessly try to achieve the state of having all the packages installed.

This is great in overall, but it presents some challanges for modifying the
contents of some of the objects that the installation submits to the cluster:
any live modifications to them will result in the original definition being
persisted instead of the changes.

Given that, before we perform any customizations to what's been provided out of
the box via the Out of the Box packages, we need to pause the top-level
`PackageInstall/tap`.


```
# create the PackageInstall object that installs a certain version of TAP,
# which then translates to a series of PackageInstall objects (one for each
# package) being submitted
#
tanzu package install tap --values-file tap-values.yaml
```





* TAP-specific details (pause package installation .. renaming ...)

    --> if you used profiles:
    --> if you didn't use profiles but did `tanzu package install`


### Profile-based installation

1. pause TAP package install


### Component-based installation

1. pause the PackageInstall for `ootb-templates`, if wanting to change
   templates

1. pause the PackageInstall for `ootb-supply-chain-<>` for changing
   supplychains



## Cartographer


https://cartographer.sh/docs/development/tutorials/first-supply-chain/
