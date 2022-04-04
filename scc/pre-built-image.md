# Pre-built image

For those applications that already have a predefined way of building their
container images, the supply chains included in the Out of the Box packages
provide the ability for specifying a pre-built image to be used in the final
application while still going through the same set of stages as any other
Workload.


## Workload

To specify a pre-built image, the `workoad.spec.image` field should be set to
the name of the container image that contains the application to be deployed.

Using the Tanzu CLI, that means leveraging the `--image` field of `tanzu apps
workload create`:

- `--image`: pre-built image, skips the source resolution and build phases of
  the supply chain

For instance, assuming we have an image named `IMAGE`, we can create a workload
making use of the flag mentioned above:

```bash
tanzu apps workload create tanzu-java-web-app \
  --app tanzu-java-web-app \
  --type web \
  --image IMAGE
```
```console
Create workload:
      1 + |---
      2 + |apiVersion: carto.run/v1alpha1
      3 + |kind: Workload
      4 + |metadata:
      5 + |  labels:
      6 + |    app.kubernetes.io/part-of: hello-world
      7 + |    apps.tanzu.vmware.com/workload-type: web
      8 + |  name: tanzu-java-web-app
      9 + |  namespace: default
     10 + |spec:
     11 + |  image: IMAGE
```


## How it works

Under the hood, an `ImageRepository` object will be created to keep track of
new images pushed under that name, making available to further resources in the
supply chain the final digest-based of the latest it found.

Whenever a new image is pushed under that image name, the ImageRepository
object will detect it, and then make it available to further resources by
updating its own `imagerepository.status.artifact.revision` with the absolute
name of that image (i.e., the image name including the digest of the latest one
found).

For instance, let's create a Workload using an image named `hello-world`,
tagged `tanzu-java-web-app` hosted under `ghcr.io` in the `kontinue`
repository:

```bash
tanzu apps workload create tanzu-java-web-app \
  --app tanzu-java-web-app \
  --type web \
  --image ghcr.io/kontinue/hello-world:tanzu-java-web-app
```

After a couple seconds, we should see the ImageRepository created for keeping
track of images named `ghcr.io/kontinue/hello-world:tanzu-java-web-app`


```scala
Workload/tanzu-java-web-app
├─ImageRepository/tanzu-java-web-app
├─PodIntent/tanzu-java-web-app
├─ConfigMap/tanzu-java-web-app
└─Runnable/tanzu-java-web-app-config-writer
  └─TaskRun/tanzu-java-web-app-config-writer-p2lzv
    └─Pod/tanzu-java-web-app-config-writer-p2lzv-pod
```

inspecting the Workload status (more specifically,
`workload.status.resources`), we can see the `image-provider` resource
promoting to further resources the image it found:

```yaml
apiVersion: carto.run/v1alpha1
kind: Workload
spec:
  image: ghcr.io/kontinue/hello-world:tanzu-java-web-app
status:
  resources:
    - name: image-provider
      outputs:
        # output being made available to further resources in the supply chain
        # (in this case, the latest image it found under that name).
        #
        - name: image
          lastTransitionTime: "2022-04-01T15:05:01Z"
          preview: ghcr.io/kontinue/hello-world:tanzu-java-web-app@sha256:9fb930a...

      # reference to the object managed by the supply chain for this
      # resource
      #
      stampedRef:
        apiVersion: source.apps.tanzu.vmware.com/v1alpha1
        kind: ImageRepository
        name: tanzu-java-web-app
        namespace: workload
      
      # reference to the template that defined how this object should look
      # like
      #
      templateRef:
        apiVersion: carto.run/v1alpha1
        kind: ClusterImageTemplate
        name: image-provider-template
```

That image found the ImageRepository is then carried through the supply chain
all the way to the final configuration that gets pushed to a git repository or
image registry so that it can be deployed in a run cluster.


## Examples

Aside from the gotchas outlined, it's mostly transparent to the supply chain
how one came up with the image being provided.

In the examples below we shouldcase a couple ways that one could end up
building container images for a Java-based application and having it carried
through the supply chains all the way to a running service.

### Dockerfile

1. create a Dockerfile that describes how to build our application and make it
   available as a container image

```Dockerfile
ARG BUILDER_IMAGE=maven
ARG RUNTIME_IMAGE=gcr.io/distroless/java17-debian11


FROM $BUILDER_IMAGE AS build

        ADD . .
        RUN unset MAVEN_CONFIG && ./mvnw clean package -B -DskipTests


FROM $RUNTIME_IMAGE AS runtime

        COPY --from=build /target/demo-0.0.1-SNAPSHOT.jar /demo.jar
        CMD [ "/demo.jar" ]
```

2. push the container image to an image registry

```bash
IMAGE_NAME=<my_registry>/tanzu-java-web-app

docker build -t $IMAGE_NAME .
docker push $IMAGE_IMAGE
```

3. create a workload that makes use of it

```bash
tanzu apps workload create tanzu-java-web-app \
  --type web \
  --label app.kubernetes.io/part-of=hello-world \
  --image ghcr.io/kontinue/hello-world:tanzu-java-web-app
```
```console
Create workload:
      1 + |---
      2 + |apiVersion: carto.run/v1alpha1
      3 + |kind: Workload
      4 + |metadata:
      5 + |  labels:
      6 + |    app.kubernetes.io/part-of: hello-world
      7 + |    apps.tanzu.vmware.com/workload-type: web
      8 + |  name: tanzu-java-web-app
      9 + |  namespace: default
     10 + |spec:
     11 + |  image: ghcr.io/kontinue/hello-world:tanzu-java-web-app
```

4. observe that we get it running


## spring maven docker

TODO


## buildpacks

```
git clone https://github.com/sample-accelerators/tanzu-java-web-app
pack build ghcr.io/kontinue/hello-world:tanzu-java-web-app-pack --builder cnbs/sample-builder:bionic
```


## Gotchas

As the supply chains still aim at Knative as the runtime for the container
image provided, the application must adhere to Knative standards:


- the image WILL NOT be relocated to the internal image registry, so, any
  components that end up touching the image must have the necessary credentials
  for pulling it.


- must listen on port 8080 on all interfaces (0.0.0.0:8080)

```
ports:
  - containerPort: 8080
    name: user-port
    protocol: TCP
```

- must be possible to run as user 1000

```
securityContext:
  runAsUser: 1000
```

- gracefully support scaling to zero

```
metadata:
  annotations:
    autoscaling.knative.dev/minScale: "1"
```
