# Building from source

Regardless of the Out of the Box Supply Chain installed, when providing source
code for the Workload, that can either come from a developer's machine
(directory in the filesystem) or a Git repository.


## Git source

To provide to the supply chains source code from a Git repository,
`workload.spec.source.git` should be filled.

Using the `tanzu` CLI, one can do so with the following flags:

- `--git-branch`: branch within the git repo to checkout
- `--git-commit`: commit SHA within the git repo to checkout
- `--git-repo`: git url to remote source code
- `--git-tag`: tag within the git repo to checkout

For instance, we could create a `Workload` whose source code comes from the
`main` branch of the repository
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

Note that the Git repository URL **must** include the scheme (`http://`,
`https://`, or `ssh://`).


### Private git repository

To fetch source code from a repository that requires credentials to be
presented, one must provide those via a Kubernetes Secret object that's
referenced by the `GitRepostiory` object created for that Workload.

```
Workload/tanzu-java-web-app
└─GitRepository/tanzu-java-web-app  
                   └───────────> `secretRef: {name: <secret_name>}`
```

Platform operators can customize the default name of the Secret during the
installation of TAP via the `gitops.ssh_secret` field in thej
`ootb-supply-chain-*` packages, or by supplying the corresponding parameter
(`gitops_ssh_secret`) to the Workloads.


#### HTTP-based auth

Despite both the package value and parameter being called `gitops_ssh_secret`,
it's possible to make use of HTTP(S) transports just as well.

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: git-http
  annotations:
    tekton.dev/git-0: https://github.com
type: kubernetes.io/basic-auth
stringData:
  username: admin
  password: admin
```

#### SSH auth


```yaml
apiVersion: v1
kind: Secret
metadata:
  name: git-ssh
type: kubernetes.io/ssh-auth
stringData:
  known_hosts: string             # git server public keys
  identity: string                # private key with pull permissions
  identity.pub: string            # public of the `identity` private key
```

1. generate a new key pair (`identity` and `identity.pub`)


once done, head to your git provider and add the `identity.pub` as a deployment
key for the repository of interest or add to an account that has access to it.
for instance, for github: `https://github.com/<repository>/settings/keys/new`.

```bash
ssh-keygen -t rsa -q -b 4096 -f "identity" -N "" -C ""
```


gather public keys from the provider (e.g., github):

```bash
ssh-keyscan github.com > ./known_hosts
```


create the secret:

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
