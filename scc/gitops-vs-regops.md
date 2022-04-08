# GitOps vs RegistryOps

Regardless of the supply chain that a Workload goes through, at the end of it
some Kubernetes configuration gets pushed to an external entity - either a Git
repository, or a container image registry.

```
Supply Chain
   
  -- fetch source 
    -- test 
      -- build 
        -- scan 
          -- apply-conventions 
            -- push config        * either to Git or Registry
```

Here we dive into the specifics of that last phase of the supply chains
constrating the use case of pushing configuration to a Git repository and an
image registry.

> **Note:** For more information about providing source code either from a
> local directory or git repository, see [Building from
> Source](building-from-source.md).  


## GitOps

Typically associated with an outerloop workflow where the configuration
produced by the supply chains should be persisted in a Git repository, it only
gets used if certain parameters are set in the supply chain:

- `gitops.repository_prefix`, configured during the Out of the Box Supply
  Chains package installation, or

- `gitops_repository`, as a Workload parameter

For instance, assuming the installation of the supply chain packages through
TAP profiles and a `tap-values.yml` as such:

```yaml
ootb_supply_chain_basic:
  registry:
    server: REGISTRY-SERVER
    repository: REGISTRY-REPOSITORY

  gitops:
    repository_prefix: https://github.com/my-org/
```

we'd expect to see that any Workloads in the cluster would end up with the
Kubernetes configuration produced throughout the supply chain to get pushed to
the repository whose name would be formed by concatenating
`gitops.repository_prefix` with the name of the Workload (in this case,
something like `https://github.com/my-org/$(workload.metadata.name).git`

Alternatively, assuming that `gitops.repository_prefix` is _not_ configured
during the installation of TAP, it'd still be possible to force a Workload to
have the configuration published in a Git repository by providing to the
Workload the `gitops_repository` parameter:

```
tanzu apps workload create tanzu-java-web-app \
  --app tanzu-java-web-app \
  --type web \
  --git-repo https://github.com/sample-accelerators/tanzu-java-web-app \
  --git-branch main \
  --param gitops_ssh_secret=SECRET-NAME \
  --param gitops_repository=https://github.com/my-org/config-repo
```

in which case, at the end of the supply chain the configuration for this
Workload would be published to the repository provided under the
`gitops_repository` parameter.


### Authentication

Regardless of how those have been configured, as long as pushing to Git is
configured (via repository prefix or repository name), credentials must be
provided for the push to successfully occur.

#### HTTP(S) Basic-auth / Token-based authentication

If the repository at which configuration will be published makes use of
`https://` or `http://` as their URL scheme, the Kubernetes Secret that
provides the credentials for that repository must provide credentials in a
Secret as follows:

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: SECRET-NAME
  annotations:
    tekton.dev/git-0: GIT-SERVER        # ! required
type: kubernetes.io/basic-auth          # ! required
stringData:
  username: GIT-USERNAME
  password: GIT-PASSWORD
```

> **Note:** both the Tekton annotation and the `basic-auth` secret type must be
> set. GIT-SERVER must be prefixed with the appropriate URL scheme and the git
> server. E.g., for https://github.com/vmware-tanzu/cartographer,
> https://github.com should be provided as the GIT-SERVER.


### SSH


In the example above, we make use of a public repository. To
make use of a private repository instead, you create a Secret in the
same namespace as the one where the Workload is being submitted to named after
the value of `gitops.ssh_secret` (the installation defaults the name to
`git-ssh`):

```
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

>**Note**: For a particular Workload, you can override the name of the secret
by using the `gitops_ssh_secret` parameter (`--param gitops_ssh_secret`)
in the Workload.

If this is your first time setting up SSH credentials for your user, the following
steps can serve as a guide:

```
# generate a new keypair.
#
#   - `identity`     (private)
#   - `identity.pub` (public)
#
# once done, head to your git provider and add the `identity.pub` as a
# deployment key for the repository of interest or add to an account that has
# access to it. for instance, for github:
#
#   https://github.com/<repository>/settings/keys/new
#
ssh-keygen -t rsa -q -b 4096 -f "identity" -N "" -C ""


# gather public keys from the provider (e.g., github):
#
ssh-keyscan github.com > ./known_hosts


# create the secret.
#
kubectl create secret generic git-ssh \
    --from-file=./identity \
    --from-file=./identity.pub \
    --from-file=./known_hosts
```

>**Note**: When you create a Secret that provides credentials for accessing your
private git repository, you can create a deploy key if your Git Provider
supports it (GitHub does). Any Git secrets you apply to
your cluster can potentially be viewed by others who have access to that
cluster. So, it is better to use Deploy keys or shared bot accounts instead of
adding personal Git Credentials.

With the namespace configured and having added the secret to be used for
fetching source code from a private repository, you can create the
Workload:


```
tanzu apps workload create tanzu-java-web-app \
  --git-branch main \
  --git-repo https://github.com/sample-accelerators/tanzu-java-web-app
  --label app.kubernetes.io/part-of=tanzu-java-web-app \
  --type web
```
```
Create workload:
      1 + |---
      2 + |apiVersion: carto.run/v1alpha1
      3 + |kind: Workload
      4 + |metadata:
      5 + |  labels:
      6 + |    apps.tanzu.vmware.com/workload-type: web
      7 + |    app.kubernetes.io/part-of: tanzu-java-web-app
      8 + |  name: tanzu-java-web-app
      9 + |  namespace: default
     10 + |spec:
     11 + |  source:
     12 + |    git:
     13 + |      ref:
     14 + |        branch: main
     15 + |      url: https://github.com/sample-accelerators/tanzu-java-web-app
```

#### <a id="local-with-git"></a> Local Iteration with Code from Git

Similar to local iteration with local code, here we make use of the same type
(`web`), but instead of pointing at source code that we have locally, we can
make use of a git repository to feed the supply chain with new changes as they
are pushed to a branch.

>**Note**: If you plan to use a private git repository, skip
to the next section, [Private Source Git Repository](#private-source).


```
tanzu apps workload create tanzu-java-web-app \
  --git-branch main \
  --git-repo https://github.com/sample-accelerators/tanzu-java-web-app
  --label app.kubernetes.io/part-of=tanzu-java-web-app \
  --type web
```
```
Create workload:
      1 + |---
      2 + |apiVersion: carto.run/v1alpha1
      3 + |kind: Workload
      4 + |metadata:
      5 + |  labels:
      6 + |    apps.tanzu.vmware.com/workload-type: web
      7 + |    app.kubernetes.io/part-of: tanzu-java-web-app
      8 + |  name: tanzu-java-web-app
      9 + |  namespace: default
     10 + |spec:
     11 + |  source:
     12 + |    git:
     13 + |      ref:
     14 + |        branch: main
     15 + |      url: https://github.com/sample-accelerators/tanzu-java-web-app
```

This scenario is only possible if the installation of the supply
chain did not include a default git repository prefix
(`gitops.repository_prefix`).


##### <a id="private-source"></a> Private Source Git Repository

In the example above, we make use of a public repository. To
make use of a private repository instead, you create a Secret in the
same namespace as the one where the Workload is being submitted to named after
the value of `gitops.ssh_secret` (the installation defaults the name to
`git-ssh`):

```
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

>**Note**: For a particular Workload, you can override the name of the secret
by using the `gitops_ssh_secret` parameter (`--param gitops_ssh_secret`)
in the Workload.

If this is your first time setting up SSH credentials for your user, the following
steps can serve as a guide:

```
# generate a new keypair.
#
#   - `identity`     (private)
#   - `identity.pub` (public)
#
# once done, head to your git provider and add the `identity.pub` as a
# deployment key for the repository of interest or add to an account that has
# access to it. for instance, for github:
#
#   https://github.com/<repository>/settings/keys/new
#
ssh-keygen -t rsa -q -b 4096 -f "identity" -N "" -C ""


# gather public keys from the provider (e.g., github):
#
ssh-keyscan github.com > ./known_hosts


# create the secret.
#
kubectl create secret generic git-ssh \
    --from-file=./identity \
    --from-file=./identity.pub \
    --from-file=./known_hosts
```

>**Note**: When you create a Secret that provides credentials for accessing your
private git repository, you can create a deploy key if your Git Provider
supports it (GitHub does). Any Git secrets you apply to
your cluster can potentially be viewed by others who have access to that
cluster. So, it is better to use Deploy keys or shared bot accounts instead of
adding personal Git Credentials.

With the namespace configured and having added the secret to be used for
fetching source code from a private repository, you can create the
Workload:


```
tanzu apps workload create tanzu-java-web-app \
  --git-branch main \
  --git-repo https://github.com/sample-accelerators/tanzu-java-web-app
  --label app.kubernetes.io/part-of=tanzu-java-web-app \
  --type web
```
```
Create workload:
      1 + |---
      2 + |apiVersion: carto.run/v1alpha1
      3 + |kind: Workload
      4 + |metadata:
      5 + |  labels:
      6 + |    apps.tanzu.vmware.com/workload-type: web
      7 + |    app.kubernetes.io/part-of: tanzu-java-web-app
      8 + |  name: tanzu-java-web-app
      9 + |  namespace: default
     10 + |spec:
     11 + |  source:
     12 + |    git:
     13 + |      ref:
     14 + |        branch: main
     15 + |      url: https://github.com/sample-accelerators/tanzu-java-web-app
```


#### GitOps

Differently from local iteration, with the GitOps approach we end up at the end
of the supply chain having the configuration that got created by it pushed to a
git repository where that is persisted and used at the basis for further
deployments.

```
SUPPLY CHAIN

    given a Workload
      watches sourcecode repo
        builds container image
          prepare configuration
            pushes config to git


DELIVERY

    given a Deliverable
      watches configurations repo
        deploys the kubernetes configurations

```

Given the extra capability of pushing to git,
there must be in the developer namespace (i.e., same namespace as the one
where the Workload is submitted to) a Secret containing credentials to a git
provider (e.g., GitHub), regardless of whether the source code comes from a
private git repository or not.

Before proceeding, make sure you have a secret with following shape fields and
annotations set:

```
apiVersion: v1
kind: Secret
metadata:
  name: git-ssh   # `git-ssh` is the default name.
                  #   - operators can change the default using `gitops.ssh_secret`.
                  #   - developers can override using `gitops_ssh_secret`
  annotations:
    tekton.dev/git-0: github.com  # git server host   (!! required)
type: kubernetes.io/ssh-auth
stringData:
  ssh-privatekey: string          # private key with push-permissions
  known_hosts: string             # git server public keys
  identity: string                # private key with pull permissions
  identity.pub: string            # public of the `identity` private key
```

>**Note**: Because of incompatibilities between Kubernetes resources
 `ssh-privatekeys` must be set to the same value as `identity`.

With the Secret created, we can move on to the Workload.


### Workload Using Default Git Organization

During the installation of `ootb-*`, one of the values that operators can
configure is one that dictates what the prefix the supply chain should use when
forming the name of the repository to push to the Kubernetes configurations
produced by the supply chains - `gitops.repository_prefix`.

That being set, all it takes to change the behavior towards using GitOps is
setting the source of the source code to a git repository and then as the
supply chain progresses, configuration are pushed to a repository named
after `$(gitops.repository_prefix) + $(workload.name)`.

e.g, having `gitops.repository_prefix` configured to `git@github.com/foo/` and
a Workload as such:

```
tanzu apps workload create tanzu-java-web-app \
  --git-branch main \
  --git-repo https://github.com/sample-accelerators/tanzu-java-web-app
  --label app.kubernetes.io/part-of=tanzu-java-web-app \
  --type web
```
```
Create workload:
      1 + |---
      2 + |apiVersion: carto.run/v1alpha1
      3 + |kind: Workload
      4 + |metadata:
      5 + |  labels:
      6 + |    apps.tanzu.vmware.com/workload-type: web
      7 + |    app.kubernetes.io/part-of: tanzu-java-web-app
      8 + |  name: tanzu-java-web-app
      9 + |  namespace: default
     10 + |spec:
     11 + |  source:
     12 + |    git:
     13 + |      ref:
     14 + |        branch: main
     15 + |      url: https://github.com/sample-accelerators/tanzu-java-web-app
```

 You see the Kubernetes configuration pushed to
`git@github.com/foo/tanzu-java-web-app.git`.

Regardless of the setup, the repository where configuration is pushed to can be
also manually overridden by the developers by tweaking the following parameters:

-  `gitops_ssh_secret`: Name of the secret in the same namespace as the
   Workload where SSH credentials exist for pushing the configuration produced
   by the supply chain to a git repository.
   Example: "ssh-secret"

-  `gitops_repository`: SSH URL of the git repository to push the Kubernetes
   configuration produced by the supply chain to.
   Example: "ssh://git@foo.com/staging.git"

-  `gitops_branch`: Name of the branch to push the configuration to.
   Example: "main"

-  `gitops_commit_message`: Message to write as the body of the commits
   produced for pushing configuration to the git repository.
   Example: "ci bump"

-  `gitops_user_name`: Username to use in the commits.
   Example: "Alice Lee"

-  `gitops_user_email`: User email address to use for the commits.
   Example: "foo@example.com"


## RegistryOps

Typically used for inner loop flows where configuration is treated as an
artifact from quick iterations by developers, in this scenario at the very end
of the supply chain configuration gets pushed to a container image registry in
the form of an [imgpkg bundle](https://carvel.dev/imgpkg/docs/v0.27.0/) - think
of it as a container image whose sole purpose is to carry arbitrary files.

For this mode of operation to be enabled, the supply chains must be configured
**without** the following parameters being configured during the installation
of the `ootb-` packages (or overwritten by the Workload via parameters):

- `gitops_repository_prefix`
- `gitops_repository`

If none of those are set, the configuration will end up being pushed to the
same container image registry as where the container image is pushed to (i.e.,
the registry configured under the `registry: {}` section of the `ootb-`
values).

For instance, assuming the installation of TAP via profiles having the
`ootb-supply-chain*` package configured as such: 

```yaml
ootb_supply_chain_basic:
  registry:
    server: REGISTRY-SERVER
    repository: REGISTRY_REPOSITORY
```

We'd expect Kubernetes configuration produced by the supply chain to be pushed
to an as an image named after `REGISTRY-SERVER/REGISTRY-REPOSITORY` including
the Workload name.

