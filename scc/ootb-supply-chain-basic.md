# Out of the Box Supply Chain Basic

This Cartographer Supply Chain ties together a series of Kubernetes resources which
drive a developer-provided Workload from source code to a Kubernetes configuration
ready to be deployed to a cluster.

This is the most basic supply chain that provides a quick path to deployment. It
makes no use of testing or scanning steps.

```
SUPPLYCHAIN
  source-provider                          flux/GitRepository|vmware/ImageRepository
       <--[src]-- image-builder            kpack/Image           : kpack/Build
           <--[img]-- convention-applier   convention/PodIntent
             <--[config]-- config-creator  corev1/ConfigMap
              <--[config]-- config-pusher  carto/Runnable        : tekton/TaskRun

DELIVERY
  config-provider                           flux/GitRepository|vmware/ImageRepository
    <--[src]-- app-deployer                 kapp-ctrl/App
```

- Watching a Git repository or local directory for changes
- Building a container image out of the source code with Buildpacks
- Applying operator-defined conventions to the container definition
- Deploying the application to the same cluster


## <a id="prerequisites"></a> Prerequisites

To use this supply chain, you must:

- Install [Out of the Box Templates](ootb-templates.html)
- Install [Out of the Box Delivery Basic](ootb-delivery-basic.html)
- Configure the Developer namespace with auxiliary objects that are used by the
  supply chain as described below

### <a id="developer-namespace"></a> Developer Namespace

The supply chains provide definitions of many of the objects that they create to transform the source code
to a container image and make it available as an application in the cluster.

The developer must provide or configure particular objects in the developer namespace so that
the supply chain can provide credentials and use permissions granted to a
particular development team.

The objects that the developer must provide or configure include:

- **[image secret](#image-secret)**: A Kubernetes secret of type
  `kubernetes.io/dockerconfigjson` that contains credentials for pushing the
  container images built by the supply chain

- **[service account](#service-account)**: The identity to be used for any interaction with the
  Kubernetes API made by the supply chain

- **[role](#role-rolebinding)**: The set of capabilities that you want to assign to the service
  account. It must provide the ability to manage all of the resources that the
  supplychain is responsible for.

- **[rolebinding](#role-rolebinding)**: Binds the role to the service account. It grants the
  capabilities to the identity.

- (Optional) **[git credentials secret](#git-credentials-secret)**: When using GitOps for managing the
  delivery of applications or a private git source, this secret provides the
  credentials for interacting with the git repository.


#### <a id="image-secret"></a> Image Secret

Regardless of the supply chain that a Workload goes through, there must be a secret
in the developer namespace. This secret contains the credentials to be passed to:

* Resources that push container images to image registries, such as Tanzu Build
Service
* Those resources that must pull container images from such image registry, such
as Convention Service and Knative.

Use the `tanzu secret registry add` command from the Tanzu CLI to provision a
secret that contains such credentials.

```
# create a Secret object using the `dockerconfigjson` format using the
# credentials provided, then a SecretExport (`secretgen-controller`
# resource) so that it gets exported to all namespaces where a
# placeholder secret can be found.
#
#
tanzu secret registry add image-secret \
  --server https://index.docker.io/v1/ \
  --username $REGISTRY_USERNAME \
  --password $REGISTRY_PASSWORD
```
```
- Adding image pull secret 'image-secret'...
 Added image pull secret 'image-secret' into namespace 'default'
```

With the command above, the secret `image-secret` of type
`kubernetes.io/dockerconfigjson` is created in the namespace.
This makes the secret available for Workloads in this same namespace.

To export the secret to all namespaces, use the `--export-to-all-namespaces`
flag.


#### <a id="service-account"></a> ServiceAccount

In a Kubernetes cluster, a ServiceAccount provides a way of representing an
identity within the Kubernetes role base access control (RBAC) system. In
the case of a developer namespace, this represents a developer or development team.

You can directly attach secrets to the ServiceAccount as bind roles. This allows you
to provide indirect ways for resources to find credentials without them needing to know the
exact name of the secrets, as well as reduce the set of permissions that a
group would have, through the use of Roles and RoleBinding objects.


```
apiVersion: v1
kind: ServiceAccount
metadata:
  name: default
secrets:
  - name: image-secret
imagePullSecrets:
  - name: image-secret
```

The ServiceAccount must have the secret created above linked to
it. If it does not, services like Tanzu Build Service (used in the supply chain)
lack the necessary credentials for pushing the images it builds for that
Workload.

#### <a id="role-rolebinding"></a> Role and RoleBinding

As the Supply Chain takes action in the cluster on behalf of the users who
created the Workload, it needs permissions within Kubernetes' RBAC system to do
so.

To achieve that, you must first describe a set of permissions for particular
resources, meaning create a Role, and then bind those permissions to an actor.
For example, creating a RoleBinding that binds the Role to the ServiceAccount.

Then bind it to the ServiceAccount:

```
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: default
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: default
subjects:
  - kind: ServiceAccount
    name: default
```


### <a id="developer-workload"></a> Developer workload

TODO
