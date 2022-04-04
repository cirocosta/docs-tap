# Building from source

Regardless of the supply chain, when providing source code it can either come
from a developer's machine (directory in the filesystem) or a Git repository.


## Local source

source code in a directory in the developer's machine

- `--local-path`: path on the local file system to a directory of source code
  to build for the workload
- `--source-image`: destination image repository where source code is staged
  before being built


authentication:

- developer must have the credentials for pushing to that image registry
- serviceaccount must point at the secret that contain the credentials for that
  image to be pulled by ImageRepository


### How it works

ImageRepository object is created
  - references serviceacccount to find the credentials

parameters:

- `serviceAccount`


## Git source

source code coming from a git repository

- `--git-branch`: branch within the git repo to checkout
- `--git-commit`: commit SHA within the git repo to checkout
- `--git-repo`: git url to remote source code
- `--git-tag`: tag within the git repo to checkout

### How it works

GitRepository is created
  - reference a secret (spec.secretRef) to find credentials

parameters:

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

