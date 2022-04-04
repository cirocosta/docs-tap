# Pre-built image

For those applications that already have a defined way of building their
container images, the supply chains included in the Out of the Box packages
provide the ability for specifying a pre-built image that should be used in the
final application while still going through the same set of stages as any other
workload.

## Workload

To specify a pre-built image, the `workoad.spec.image` field should be set.

Using the Tanzu CLI, that means leveraging the `--image` field of `tanzu apps
workload create`. For instance, assuming we have an image named `IMAGE`:

- `--image`: pre-built image, skips the source resolution and build phases of
  the supply chain

```bash
tanzu apps workload create tanzu-java-web-app \
  --type web \
  --label app.kubernetes.io/part-of=tanzu-java-web-app \
  --image ghcr.io/kontinue/hello-world:tanzu-java-web-app \
  --namespace workload
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

## How it works

Under the hood, an `ImageRepository` object will be created to keep track of
new images pushed under that name, making available to further resources in the
supply chain the final digest-based of the latest it found.

Using `ootb-supply-chain-basic` as an example and the Workload above, we should
see the following object hierarchy formed:

```
NAME
Workload/tanzu-java-web-app
├─ImageRepository/tanzu-java-web-app
├─PodIntent/tanzu-java-web-app
├─ConfigMap/tanzu-java-web-app
└─Runnable/tanzu-java-web-app-config-writer
  └─TaskRun/tanzu-java-web-app-config-writer-p2lzv
    └─Pod/tanzu-java-web-app-config-writer-p2lzv-pod
```

```yaml
apiVersion: carto.run/v1alpha1
kind: Workload
spec:
  image: ghcr.io/kontinue/hello-world:tanzu-java-web-app
status:
  # ...
  resources:
  - name: image-provider
    outputs:
    - name: image
      lastTransitionTime: "2022-04-01T15:05:01Z"
      preview: ghcr.io/kontinue/hello-world:tanzu-java-web-app@sha256:9fb930acce8d33277cd323a6e9528d1e67bded9e05e02432fadebf43b276bb44
    stampedRef:
      apiVersion: source.apps.tanzu.vmware.com/v1alpha1
      kind: ImageRepository
      name: tanzu-java-web-app
      namespace: workload
    templateRef:
      apiVersion: carto.run/v1alpha1
      kind: ClusterImageTemplate
      name: image-provider-template
```



## Dockerfile example

1. create a dockerfile that describes the building of our artifact and how to
   run it

```
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


## `pack` example

```
git clone https://github.com/sample-accelerators/tanzu-java-web-app
pack build ghcr.io/kontinue/hello-world:tanzu-java-web-app-pack --builder cnbs/sample-builder:bionic
```


## Gotchas

As the supply chains still aim at Knative as the runtime for the container
image provided, the application must adhere to Knative standards:

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
