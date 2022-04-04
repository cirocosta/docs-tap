# Building from source

Regardless of the supply chain, when providing source code it can either come
from a developer's machine (directory in the filesystem) or a Git repository.

When it comes to building applications from source code, there are two ways
that such source code can be made available for the supply chain components:
either via git or a container image.


## Git source

To provide to the supply chains source code from a Git repository,
`workload.spec.source.git` should be filled.

Using the `tanzu` CLI, one can do so with the following flags:

- `--git-branch`: branch within the git repo to checkout
- `--git-commit`: commit SHA within the git repo to checkout
- `--git-repo`: git url to remote source code
- `--git-tag`: tag within the git repo to checkout

For instance:

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


### How it works

With the `git` field under `spec.source` filled, the supply chain takes care of
creating a child GitRepository object that keeps track of commits made to the
git repository stated in `workload.spec.source.git`, which in turn makes
available for further components in the supply chain the latest commits made to
that branch in that repository via an HTTP-based URL that can be reached within
the cluster.

- `serviceAccount`:
- `gitImplementation`:
- `gitops_ssh_secret`:


### Private git repository

secret must be provided

- `gitops_ssh_secret` set to the name of the secret to be referenced


#### HTTP-based auth

secret

```
```

#### SSH auth

secret

```
```




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
