# Linode Block Storage CSI Driver [![Build Status](https://travis-ci.org/linode/linode-blockstorage-csi-driver.svg?branch=master)](https://travis-ci.org/linode/linode-blockstorage-csi-driver)

## Overview
The Container Storage Interface ([CSI](https://github.com/container-storage-interface/spec)) Driver for Linode Block Storage enables container orchestrators such as Kubernetes to manage the life-cycle of persistant storage claims.

More information about the Kubernetes CSI can be found in the GitHub [Kubernetes CSI](https://kubernetes-csi.github.io/docs/Example.html) and [CSI Spec](https://github.com/container-storage-interface/spec/) repos.

## Deployment

### Requirements

* Kubernetes v1.13+
* The node `hostname` must match the Linode Instance `label`
* `--allow-privileged` must be enabled for the API server and kubelet
* Should have the following [feature gates enabled](https://kubernetes.io/docs/reference/command-line-tools-reference/feature-gates/#overview): `CSINodeInfo`, `CSIDriverRegistry`
* The following feature gates may be used in future versions: `BlockVolume`, `CSIBlockVolume`

### Secure a Linode API Access Token:

Generate a Personal Access Token (PAT) using the [Linode Cloud Manager](https://cloud.linode.com/profile/tokens).

This token will need:

* Read/Write access to Volumes (to create and delete volumes)
* Read/Write access to Linodes (to attach and detach volumes)
* A sufficient "Expiry" to allow for continued use of your volumes

### Create a Kubernetes secret

Run the following commands to stash a `LINODE_TOKEN` in your Kubernetes cluster:

```bash
read -s -p "Linode API Access Token: " LINODE_TOKEN
read -p "Linode Region of Cluster: " LINODE_REGION
cat <<EOF | kubectl create -f -
```

Paste the following text at the prompt:

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: linode
  namespace: kube-system
stringData:
  token: "$LINODE_TOKEN"
  region: "$LINODE_REGION"
EOF
```

You should receive notification that the secret was created.  You can confirm this by running:

```sh
$ kubectl -n kube-system get secrets
NAME                  TYPE                                  DATA      AGE
linode          Opaque                                2         18h
```

### Deploy the CSI

The following command will deploy the CSI driver with the related Kubernetes volume attachment, driver registration, and provisioning sidecars:

```sh
kubectl apply -f https://raw.githubusercontent.com/linode/linode-blockstorage-csi-driver/master/pkg/linode-bs/deploy/releases/linode-blockstorage-csi-driver-v0.1.2.yaml
```

This deployment is a concatenation of all of the `yaml` files in [pkg/linode-bs/deploy/kubernetes/](https://github.com/linode/linode-blockstorage-csi-driver/tree/master/pkg/linode-bs/deploy/kubernetes/).

Notably, this deployment will:

* set the default storage class to `linode-block-storage` [Learn More](https://kubernetes.io/docs/tasks/administer-cluster/change-default-storage-class/)

  This behavior can be modified in the [csi-storageclass.yaml](https://github.com/linode/linode-blockstorage-csi-driver/blob/master/pkg/linode-bs/deploy/kubernetes/05-csi-storageclass.yaml) section of the deployment by toggling the `storageclass.kubernetes.io/is-default-class` annotation.

  ```sh
  $ kubectl get storageclasses
  NAME                             PROVISIONER               AGE
  linode-block-storage (default)   linodebs.csi.linode.com   2d
  ```
  
* use a `reclaimPolocy` of `Released` [Learn More](https://kubernetes.io/docs/tasks/administer-cluster/change-pv-reclaim-policy/)
  
  Volumes created by this CSI driver will automatically be deleted when they are no longer needed.

### Example Usage

This repository contains [two manifests](https://github.com/linode/linode-blockstorage-csi-driver/tree/master/pkg/linode-bs/examples/kubernetes) that demonstrate use of the Linode BlockStorage CSI.  These manifests will create a PersistentVolume Claim using the `linode-block-storage` storage class and then consume it in a minimal pod.

Once you have installed the Linode BlockStorage CSI, the following commands will run the example.  Be sure to delete the pod and PVC and ensure that the Linode BlockStorage Volume has been deleted.

```sh
kubectl create -f https://raw.githubusercontent.com/linode/linode-blockstorage-csi-driver/master/pkg/linode-bs/examples/kubernetes/csi-pvc.yaml
kubectl create -f https://raw.githubusercontent.com/linode/linode-blockstorage-csi-driver/master/pkg/linode-bs/examples/kubernetes/csi-app.yaml
```

Verify that the pod is running and can consume the volume:

```sh
kubectl get pvc/csi-example-pvc pods/csi-example-pod
kubectl describe pvc/csi-example-pvc pods/csi-example-pod | less
```

Now, let's add some data into the PVC, delete the POD, and recreate the pod.  Our data will remain intact.

```sh
$ kubectl exec -it csi-example-pod -- /bin/sh -c "echo persistence > /data/example.txt; ls -l /data"
total 20
-rw-r--r--    1 root     root            12 Dec  5 13:06 example.txt
drwx------    2 root     root         16384 Dec  5 06:03 lost+found

$ kubectl delete pods/csi-example-pod
pod "csi-example-pod" deleted

$ kubectl create -f https://raw.githubusercontent.com/linode/linode-blockstorage-csi-driver/master/pkg/linode-bs/examples/kubernetes/csi-app.yaml
pod/csi-example-pod created

$ sleep 30; kubectl exec -it csi-example-pod -- /bin/sh -c "ls -l /data; cat /data/example.txt"
total 20
-rw-r--r--    1 root     root            12 Dec  5 13:06 example.txt
drwx------    2 root     root         16384 Dec  5 06:03 lost+found
persistence

```

## Disclaimers

* This is not officially supported by Linode.  [Join us on Slack](#join-us-on-slack) for community support.
* Until this driver has reached v1.0.0 it may not maintain compatibility between driver versions
* This driver does not work with versions of Kubernetes earlier than 1.13
* Requests for Persistent Volumes with a `require_size` less than the Linode minimum Block Storage size will be fulfilled with a Linode Block Storage volume of the minimum size (currently 10GiB), this is [in accordance with the CSI specification](https://github.com/container-storage-interface/spec/blob/v1.0.0/spec.md#createvolume).  The upper-limit size constraint (`limit_bytes`) will also be honored so the size of Linode Block Storage volumes provisioned will not exceed this parameter.

## Contribution Guidelines

Want to improve the linode-blockstorage-csi-driver? Please start [here](.github/CONTRIBUTING.md).

## Doing a Release

1. Open up the Makefile.
1. Check that the `IMAGE_VERSION` variable is set to the value that you want. You should always start with a `canary` release.
1. run `make push`. As of this writing, the `push` target depends upon the `linode-container`, `linode`, `vendor, and `test` targets. So the code should be vendored, the tests should run, the build should run, a container should be built, and it should be pushed to the container registry.
1. Test the new canary version on a live cluster.
1. Change `IMAGE_VERSION` to the new version of the CSI driver, for exmaple `v0.2.1`
1. Run `make push` again
1. Push a git tag for the new version, perferrably pgp signed.
1. Make a GitHub release and write the release notes for the release.

## Join us on Slack

For general help or discussion, join the [Kubernetes Slack](http://slack.k8s.io/) channel [#linode](https://kubernetes.slack.com/messages/CD4B15LUR).

For development and debugging, join the [Gopher's Slack](https://invite.slack.golangbridge.org/) channel [#linodego](https://gophers.slack.com/messages/CAG93EB2S).

