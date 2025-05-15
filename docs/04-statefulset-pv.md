# Troubleshooting StatefulSet

In this senario, I'm deploying a statefulSet using [this](https://github.com/git-tux/kubernetes-troubleshooting-zero-to-hero/blob/main/04-statefulset-pv/sample-statefulset.yaml) manifest and I'm going to apply any fixes in order to make sure that the deployed application is running as expected.

## Deployment

For the deployment I'm using a local kubernetes cluster created with the [kind](https://kind.sigs.k8s.io/) Kubernetes tool. I can created a specific namespace for this exercise and I have set it as default context

```
kubectl create namespace troubleshooting
kubectl config set-context --current --namespace=troubleshooting
kubectl apply -f 04-statefulset-pv/sample-statefulset.yaml
```

## Troubleshooting

After the deployment, I'm checking the status of the web statefulset:

```
kubectl get sts
NAME   READY   AGE
web    0/3     6s
```

Which show me that there are zero replicas deployed out of the three desired. I'm checking  the status of the running pods, and there is one web-0 replica in status **Pending**

```
kubectl get pods
NAME                                READY   STATUS             RESTARTS   AGE
web-0                               0/1     Pending            0          3m40s
```

Using the kubectl events cmd it seems that the pod cannot be started because the requested storage couldn't be provisioned due to a wrong storage class request (ebs)

```
2m7s        Normal    SuccessfulCreate        statefulset/web                          create Pod web-0 in StatefulSet web successful
12m         Warning   ProvisioningFailed      persistentvolumeclaim/www-web-0          storageclass.storage.k8s.io "ebs" not found
11m         Warning   ProvisioningFailed      persistentvolumeclaim/www-web-0          storageclass.storage.k8s.io "default" not found
7m2s        Warning   ProvisioningFailed      persistentvolumeclaim/www-web-0          storageclass.storage.k8s.io "default" not found
6m50s       Normal    WaitForFirstConsumer    persistentvolumeclaim/www-web-0          waiting for first consumer to be created before binding
6m50s       Normal    ExternalProvisioning    persistentvolumeclaim/www-web-0          Waiting for a volume to be created either by the external provisioner 'rancher.io/local-path' or manually by the system administrator. If volume creation is delayed, please verify that the provisioner is running and correctly registered.
6m50s       Normal    Provisioning            persistentvolumeclaim/www-web-0          External provisioner is provisioning volume for claim "troubleshooting/www-web-0"
6m47s       Normal    ProvisioningSucceeded   persistentvolumeclaim/www-web-0          Successfully provisioned volume pvc-7f54d8f1-6962-457f-9c5a-733c570c48e1
8s          Warning   ProvisioningFailed      persistentvolumeclaim/www-web-0          storageclass.storage.k8s.io "ebs" not found
```

Next step, I had to delete the statefulset configuration and the pvc that was in Pending state and re-apply the manifest with the correct storage class (standard).

```
sed -i s/ebs/standard/ 04-statefulset-pv/sample-statefulset.yaml
kubectl delete sts web
kubectl delete pvc www-web-0
kubectl apply -f 04-statefulset-pv/sample-statefulset.yaml
```

After this change, the pvc was successfully provisioned. However, the pods were in **ImagePullBackOff** status because the image schema is not supported:

```
  Warning  Failed     25s                kubelet            Error: ImagePullBackOff
  Normal   Pulling    10s (x2 over 26s)  kubelet            Pulling image "registry.k8s.io/nginx-slim:0.8"
  Warning  Failed     10s (x2 over 25s)  kubelet            Failed to pull image "registry.k8s.io/nginx-slim:0.8": failed to pull and unpack image "registry.k8s.io/nginx-slim:0.8": failed to get converter for "registry.k8s.io/nginx-slim:0.8": Pulling Schema 1 images have been deprecated and disabled by default since containerd v2.0. As a workaround you may set an environment variable `CONTAINERD_ENABLE_DEPRECATED_PULL_SCHEMA_1_IMAGE=1`, but this will be completely removed in containerd v2.1.
  Warning  Failed     10s (x2 over 25s)  kubelet            Error: ErrImagePull
```

I could try to build the image if I had its Dockerfile. However, I chose to pull the image locally and push it back to a public container registry. This way, as the docker version in my workstation is newer, the image will be uploaded with a supported schema. I decided to use github packages as the container registry since I already had a fork of the [kubernetes-troubleshooting-zero-to-hero](https://github.com/iam-veeramalla/kubernetes-troubleshooting-zero-to-hero)

```
 docker login --username git-tux --password $GITHUB_TOKEN ghcr.io
 docker image pull registry.k8s.io/nginx-slim:0.8 
 docker image tag registry.k8s.io/nginx-slim:0.8 ghcr.io/git-tux/nginx-slim:0.8
 docker image push docker image push ghcr.io/git-tux/nginx-slim:0.8
```

Then I had to link this image with my github repository and configure the image with public access. All I had to do next was to kill the the web-0 instance that it was in state ErrImagePull:

```
kubectl delete pod web-0
```

After a few seconds the 3 replicas of the web statefulset were up and running:

```
kubect get sts
NAME   READY   AGE
web    3/3     3h38m

kubectl get pods
NAME                               READY   STATUS    RESTARTS   AGE
web-0                              1/1     Running   0          3h9m
web-1                              1/1     Running   0          3h9m
web-2                              1/1     Running   0          3h8m
```
