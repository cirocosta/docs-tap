# Building from source

Regardless of the Out of the Box Supply Chain Package installed, when it comes
to providing source code for the Workload, that can either come from a
developer's machine (directory in the filesystem) or a Git repository.

Below we'll dive into details about both.

> **Note:** If you don't want to have the application built from scratch using
> the supply chain, but instead provide a pre-built container image, check out
> [Pre-built image](pre-built-image.md).


## Git source

To provide to the supply chains source code from a Git repository,
`workload.spec.source.git` should be filled.

Using the `tanzu` CLI, one can do so with the use of the following flags:

- `--git-branch`: branch within the git repo to checkout
- `--git-commit`: commit SHA within the git repo to checkout
- `--git-repo`: git url to remote source code
- `--git-tag`: tag within the git repo to checkout

For instance, having installed `ootb-supply-chain-basic`, we could create a
`Workload` whose source code comes from the `main` branch of the repository
`https://github.com/sample-accelerators/tanzu-java-web-app` issuing the
following command:

```bash
tanzu apps workload create tanzu-java-web-app \
  --app tanzu-java-web-app \
  --type web \
  --git-repo https://github.com/sample-accelerators/tanzu-java-web-app \
  --git-branch main
```
```console
Create workload:
      1 + |---
      2 + |apiVersion: carto.run/v1alpha1
      3 + |kind: Workload
      4 + |metadata:
      5 + |  labels:
      6 + |    app.kubernetes.io/part-of: tanzu-java-web-app
      7 + |    apps.tanzu.vmware.com/workload-type: web
      8 + |  name: tanzu-java-web-app
      9 + |  namespace: default
     10 + |spec:
     11 + |  source:
     12 + |    git:
     13 + |      ref:
     14 + |        branch: main
     15 + |      url: https://github.com/sample-accelerators/tanzu-java-web-app
```

> **Note:** the Git repository URL must include the scheme (`http://`,
`https://`, or `ssh://`).


### Private git repository

To fetch source code from a repository that requires credentials, one must
provide those via a Kubernetes Secret object that's referenced by the
`GitRepostiory` object created for that Workload (see [how it
works](#how-it-works) to know more about the underlying process of detecting
changes to the repository).

```scala
Workload/tanzu-java-web-app
└─GitRepository/tanzu-java-web-app  
                   └───────> secretRef: {name: SECRET-NAME}
                                                   |
                                      either a default from TAP installation or
                                           gitops_ssh_secret Workload parameter
```

Platform operators that installed the Out of the Box Supply Chain packages
using TAP profiles can customize the default name of the secret (`git-ssh`, by
default) by tweaking the corresponding `ootb_supply_chain*` property in the
`tap-values.yml` file:

```yaml
ootb_supply_chain_basic:
  gitops:
    ssh_secret: SECRET-NAME
```

For those that installed the `ootb-supply-chain-*` package individually via
`tanzu package install`, one can tweak the `ootb-supply-chain-*-values.yml` as
such:

```yaml
gitops:
  ssh_secret: SECRET-NAME
```

Ultimately, it's also possible to override the default secret name directly in
the Workload by leveraging the `gitops_ssh_secret` parameter, regardless of how
TAP has been installed. Using the Tanzu CLI, that can be done with the
`--param` flag. For instance:

```bash
tanzu apps workload create tanzu-java-web-app \
  --app tanzu-java-web-app \
  --type web \
  --git-repo https://github.com/sample-accelerators/tanzu-java-web-app \
  --git-branch main \
  --param gitops_ssh_secret=SECRET-NAME
```
```console
Create workload:
      1 + |---
      2 + |apiVersion: carto.run/v1alpha1
      3 + |kind: Workload
      4 + |metadata:
      5 + |  labels:
      6 + |    app.kubernetes.io/part-of: tanzu-java-web-app
      7 + |    apps.tanzu.vmware.com/workload-type: web
      8 + |  name: tanzu-java-web-app
      9 + |  namespace: default
     10 + |spec:
     11 + |  params:
     12 + |  - name: gitops_ssh_secret	#! parameter that overrides the default
     13 + |    value: SECRET-NAME       #! secret name
     14 + |  source:
     15 + |    git:
     16 + |      ref:
     17 + |        branch: main
     18 + |      url: https://github.com/sample-accelerators/tanzu-java-web-app
```

> **Note:** a secret reference will only be provided to GitRepository if
> `gitops_ssh_secret` has been set to a non-empty string in some fashion
> (either package property or workload parameter). If you need to force a
> GitRepository to not reference a secret, set the value to an empty string
> (`""`).

With the name of secret defined, we can move on to the definition of the secret
itself.


#### HTTP-based auth

Despite both the Package value being called `gitops.ssh_secret` and Workload
parameter `gitops_ssh_secret`, it's possible to make use of HTTP(S) transports
just as well.

To do so, first make sure that the repository in the Workload spec makes use of
http OR https scheme in the URL (e.g., `https://github.com/my-org/my-repo`).
Then, create a Kubernetes Secret object of type `kubernetes.io/basic-auth` like
so:


```yaml
apiVersion: v1
kind: Secret
metadata:
  name: git-credentials
type: kubernetes.io/basic-auth
stringData:
  username: GIT-USERNAME
  password: GIT-PASSWORD
```

> **Note:** when leveraging the GitOps workflow for the supply chains (see
> [GitOps vs RegistryOps](gitops-vs-regops.md)) with HTTP-based authentication,
> an extra annotation (`tekton.dev/git-0: <git-server-address>`) must be
> included.


For instance, assuming we have a repository called `kontinue/hello-world` in
GitHub that requires authentication and that we have an access token with the
privileges of reading the contents of the repository, we can create the Secret
as follows:

```
apiVersion: v1
kind: Secret
metadata:
  name: git-credentials
type: kubernetes.io/basic-auth
stringData:
  username: ""
  password: GITHUB-ACCESS-TOKEN
```

> **Note** In the example above we use an access token because GitHub
> deprecated basic auth with plain username and password. See [GitHub
> docs][gh-creating-access-token] to know more.

[gh-creating-access-token]: https://docs.github.com/en/authentication/keeping-your-account-and-data-secure/creating-a-personal-access-token


#### SSH auth

Aside from using HTTP(S) as a transport, it's also possible to make use of SSH.

First make sure that the repository URL in the Workload spec makes use of
`ssh://` as the scheme in the URL (e.g.,
`ssh://git@github.com:my-org/my-repo.git`).  Then, create a Kubernetes Secret
object of type `kubernetes.io/ssh-auth` like so:

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: git-ssh
type: kubernetes.io/ssh-auth
stringData:
  known_hosts: string             # git server public keys
  identity: string                # private key with pull permissions
  identity.pub: string            # public key of the `identity` key pair
```



1. Generate a new SSH key pair (`identity` and `identity.pub`)

    ```bash
    ssh-keygen -t ecdsa -b 521 -C "" -f "identity" -N ""
    ```

    Once done, head to your git provider and add the `identity.pub` as a
    deployment key for the repository of interest or add to an account that has
    access to it. For instance, for GitHub, visit
    `https://github.com/<repository>/settings/keys/new`.

1. gather public keys from the provider (e.g., github):

    ```bash
    ssh-keyscan github.com > ./known_hosts
    ```

1. Create the Kubernetes Secret based using the contents of the files above:

    ```bash
    kubectl create secret generic git-ssh \
      --from-file=./identity \
      --from-file=./identity.pub \
      --from-file=./known_hosts
    ```

### How it works

With the `workload.spec.source.git` filled, the supply chain takes care of
creating a child `GitRepository` object that keeps track of commits made to the
git repository stated in `workload.spec.source.git`.

For each revision found, `gitrepository.status.artifact` gets updated providing
information about an HTTP endpoint that it makes available for other components
to fetch the source code from within the cluster, as well as the digest of the
latest commit found:

```yaml
apiVersion: source.toolkit.fluxcd.io/v1beta1
kind: GitRepository
metadata:
  name: tanzu-java-web-app
spec:
  gitImplementation: go-git
  ignore: '!.git'
  interval: 1m0s
  ref: {branch: main}
  timeout: 20s
  url: https://github.com/sample-accelerators/tanzu-java-web-app
status:
  artifact:
    checksum: 375c2daee5fc8657c5c5b49711a8e94d400994d7
    lastUpdateTime: "2022-04-07T15:02:30Z"
    path: gitrepository/default/tanzu-java-web-app/d85df1fc.tar.gz
    revision: main/d85df1fc28c6b86ca54bd613f55991645d3b257c
    url: http://source-controller.flux-system.svc.cluster.local./gitrepository/default/tanzu-java-web-app/d85df1fc.tar.gz
  conditions:
  - lastTransitionTime: "2022-04-07T15:02:30Z"
    message: 'Fetched revision: main/d85df1fc28c6b86ca54bd613f55991645d3b257c'
    reason: GitOperationSucceed
    status: "True"
    type: Ready
  observedGeneration: 1
```

This way, with Cartographer passing the artifact URL and revision for further
components, those just need to be able to consume source code from an internal
URL where a tarball with the source code can be fetch, not having to deal with
any Git-specific details.



### Related Parameters

There are a couple of parameters that can be passed via the Workload object's
`workload.spec.params` field to override the default behavior of the
GitRepository object created for keeping track of the changes to a repository.

These are:

- `gitImplementation`: name of the git implementation (one of `libgit2` or
  `go-git`) to be used for fetching the source code
- `gitops_ssh_secret`: name of the secret in the same namespace as the Workload
  where credentials to for fetching the repository can be found.


## Local source

To provide source code from a local directory (e.g., a directory in the
developer's filesystem), the `tanzu` CLI provides two flags that allows one to
tell where the source code is at in the filesystem, and where it should be
pushed to as a container image:

- `--local-path`: path on the local file system to a directory of source code
  to build for the workload
- `--source-image`: destination image repository where source code is staged
  before being built

This way, regardless of whether the cluster the developer is targetting is
really local (a cluster in the developer's machine) or not, the source code
will be made available by using an image registry for that.

### How it works

When a Workload specifies that source code should come from an image (i.e.,
`workload.spec.source.image` is set pointing at the registry provided via
`--source-image`), instead of having a GitRepository object created, an
ImageRepository object is instantiated instead, with its spec filled in such a
way to keep track of images pushed the registry provided by the user.

For instance, given the following Workload

```yaml
apiVersion: carto.run/v1alpha1
kind: Workload
metadata:
  name: app
  labels:
    app.kubernetes.io/part-of: app
    apps.tanzu.vmware.com/workload-type: web
spec:
  source:
    image: 10.188.0.3:5000/test:latest
```

we should see that instead of a GitRepository object, we get an
ImageRepository:

```diff
  Workload/app
  │
- ├─GitRepository/app
+ ├─ImageRepository/app
  │
  ├─Image/app
  │ ├─Build/app-build-1
  │ │ └─Pod/app-build-1-build-pod
  │ ├─PersistentVolumeClaim/app-cache
  │ └─SourceResolver/app-source
  │
  ├─PodIntent/app
  │
  ├─ConfigMap/app
  │
  └─Runnable/app-config-writer
    └─TaskRun/app-config-writer-2zj7w
      └─Pod/app-config-writer-2zj7w-pod
```

Workload parameters:

- `serviceAccount`: the name of the serviceaccount that specifies the secret to
  be used by the objects created by the Workload according to the supply
chains.


### Authentication

#### Developer

As the `tanzu` CLI needs to push content (the source code) to an image registry
indicated via `--source-image`, it's important for the CLI to find the
credentials that allows it to do so, requiring the developer to configure their
machine accordingly.

To ensure credentials are available, use `docker` or other tools that are
compatible with docker's authentication mechanisms to log into the image
registry being targetted:

```
docker login <image_registry>
```

#### Supply chain components

Aside from the developer's ability to push source code to the image registry,
the cluster must also have the proper credentials for being able to pull that
container image and unpack it so it can run tests, build the application, etc.

To do so, make sure the serviceaccount used by the Workload points at the
Kubernetes secret that contains the credentials.
