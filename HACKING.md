# Notes for hacking on local-storage-operator

## For developing on OpenShift and OLM

1. Download and install `opm` tool via - https://github.com/operator-framework/operator-registry

2. Create images as documented below

## Running the operator locally

1. Install LSO via OLM/OperatorHub in GUI

2. Build the operator
```
make build-operator
```

3. Export env variables required by LSO [deployment](https://github.com/openshift/local-storage-operator/blob/8fc42cc8b990907c88a6da551dc85b55c2dc4417/config/manifests/4.10/local-storage-operator.clusterserviceversion.yaml#L363)
```
export DISKMAKER_IMAGE=quay.io/openshift/origin-local-storage-diskmaker:latest
export KUBE_RBAC_PROXY_IMAGE=quay.io/openshift/origin-kube-rbac-proxy:latest
export PRIORITY_CLASS_NAME=openshift-user-critical
```

4. Define LSO namespace as env variable
```
export WATCH_NAMESPACE=openshift-local-storage
```

5. Scale down the operator (on remote cluster)
```
oc scale --replicas=0 deployment.apps/local-storage-operator -n openshift-local-storage
```

6. Run the operator locally
```
~> ./_output/bin/local-storage-operator -kubeconfig=$KUBECONFIG
```

## Automatic creation of operator, bundle and index images

All the images including operator, diskmaker, bundle and index images can be created in one shot using following command:

```
~> ./hack/sync_bundle -o <operator_image> -d <diskmaker_image> -b <bundle_image> -i <index_image> bundle
```

This command also pushes the images to selected docker registry, so if you ran this command with following arguments:

```
~> ./hack/sync_bundle -o quay.io/gnufied/local-storage-operator:latest  \
        -d quay.io/gnufied/local-storage-diskmaker:latest \
        -b quay.io/gnufied/local-storage-bundle:v1 \
        -i quay.io/gnufied/gnufied-index:v1 bundle
```

The command will create all of these images and push them to quay. Operator and diskmaker arguments to `sync_bundle` script can be skipped and in that case `quay.io/openshift/origin-local-storage-diskmaker:latest` and `quay.io/openshift/origin-local-storage-operator:latest` version of those images are used:


```
~> ./hack/sync_bundle -b quay.io/gnufied/local-storage-bundle:v1 \
        -i quay.io/gnufied/gnufied-index:v1 bundle
```

This should give us index image `quay.io/gnufied/gnufied-index:v1`. Update the `CatalogSource` entry in `examples/olm/catalog-create-subscribe.yaml`
to point to your newly created index image. Once updated, we can install local-storage-operator via following command:

```
~> oc create -f examples/olm/catalog-create-subscribe.yaml
```

## Manual creation of bundle and index image.

1. Since we will be going to test with our version of images, we need to modify CSV file to point to our version of image. This can be done by modifying following file:

```
~> vim config/manifests/local-storage-operator.clusterserviceversion.yaml
```

and change image names in deployment field.

2. Now lets build a bundle image which can be used by index image. This can be done by:

```
~> cd config
~> docker build -f ./bundle.Dockerfile -t quay.io/gnufied/local-storage-bundle:bundle1 .
```

3. Tag and push image to quay.io (or a container registry of your choice). Make sure that images are publicly available.
4. Now lets build index image which we can use from Openshift:

```
~> opm index add --bundles quay.io/gnufied/local-storage-bundle:bundle1 --tag quay.io/gnufied/gnufied-index:1.0.0 --container-tool docker
```

If you are using podman then there is no need to specify container-tool option.

5. Tag and push index image to quay.io (or a container registry of your choice). Make sure that images are publicly available.

6. Edit the catalog source template example `examples/olm/catalog-create-subscribe.yaml` to point to your index image:

```
~> vim examples/olm/catalog-create-subscribe.yaml
```

7. Create a catalogSource and subscribe to the source by applying `examples/olm/catalog-create-subscribe.yaml`.

```
~> oc create -f examples/olm/catalog-create-subscribe.yaml
```

8. Switch to `openshift-local-storage` project and proceed with creating CR and start using the operator.

```
~> oc project openshift-local-storage
```

### Cleaning up after a deploy

When deploying on OpenShift and OLM, just deleting catalog and subscription is not enough. You obviously have to run:

```
oc delete -f examples/olm/catalog-create-subscribe.yaml
```

But then you may also have a leftover CSV which must be deleted:


```
oc get csv|grep local
```

You also will have leftover CRD object which must be deleted:


```
oc get crd|grep local
```
