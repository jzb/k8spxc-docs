# Scale MySQL on Kubernetes and OpenShift

One of the great advantages brought by Kubernetes and the OpenShift
platform is the ease of an application scaling. Scaling an application
results in adding resources or Pods and scheduling them to available
Kubernetes nodes.

Scaling can be vertical and horizontal. Vertical scaling adds more
compute or storage resources to MySQL nodes; horizontal scaling is
about adding more nodes to the cluster.

## Vertical scaling

### Scale compute

There are multiple components that Operator deploys and manages: Percona 
XtraDB Cluster (PXC), HAProxy or ProxySQL, etc. To add or reduce CPU or Memory 
you need to edit corresponding sections in the Custom Resource. We follow 
the structure for `requests` and `limits` that Kubernetes [provides](https://kubernetes.io/docs/concepts/configuration/manage-resources-containers/).

To add more resources to your MySQL nodes in PXC edit the following section in
the Custom Resource:

```yaml
spec:
  pxc:
    resources:
      requests: 
        memory: 4G
        cpu: 2
      limits:
        memory: 4G
        cpu: 2
```

Use our reference documentation for the [Custom Resource options](operator.md#operator-custom-resource-options) 
for more details about other components.

### Scale storage

Kubernetes manages storage with a PersistentVolume (PV), a segment of
storage supplied by the administrator, and a PersistentVolumeClaim
(PVC), a request for storage from a user. In Kubernetes v1.11 the
feature was added to allow a user to increase the size of an existing
PVC object. The user cannot shrink the size of an existing PVC object.

#### Volume Expansion capability

Certain volume types support PVCs expansion (exact details about
PVCs and the supported volume types can be found in [Kubernetes
documentation](https://kubernetes.io/docs/concepts/storage/persistent-volumes/#expanding-persistent-volumes-claims)).

You can run the following command to check if your storage supports the expansion capability:

``` {.bash data-prompt="$" }
$ kubectl describe sc <storage class name> | grep allowVolumeExpansion
```

??? example "Expected output"

    ``` {.text .no-copy}
    allowVolumeExpansion: true
    ```

1. Get the list of volumes for you cluster:

    ``` {.bash data-prompt="$" }
    $ kubectl get pvc -l app.kubernetes.io/instance=<CLUSTER_NAME>
    ```

    ??? example "Expected output"

        ``` {.text .no-copy}
        NAME                     STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS   AGE
        datadir-cluster1-pxc-0   Bound    pvc-90f0633b-0938-4b66-a695-556bb8a9e943   6Gi        RWO            standard       5m13s
        datadir-cluster1-pxc-1   Bound    pvc-7409ea83-15b6-448f-a6a0-12a139e2f5cc   6Gi        RWO            standard       3m52s
        datadir-cluster1-pxc-2   Bound    pvc-90f0b2f8-9bba-4262-904c-1740fdd5511b   6Gi        RWO            standard       2m40s
        ```

2. Patch the volume to increase the size

    You can either edit the pvc or run the patch command:

    ``` {.bash data-prompt="$" }
    $ kubectl patch pvc <pvc-name> -p '{ "spec": { "resources": { "requests": { "storage": "NEW STORAGE SIZE" }}}}'
    ```

    ??? example "Expected output"

        ``` {.text .no-copy}
        persistentvolumeclaim/datadir-cluster1-pxc-0 patched
        ```

3. Check if expansion is successful by running describe:

    ``` {.bash data-prompt="$" }
    $ kubectl describe pvc <pvc-name>
    ```

    ??? example "Expected output"

        ``` {.text .no-copy}
        ...
        Normal  ExternalExpanding           3m52s              volume_expand                                                                                     CSI migration enabled for kubernetes.io/gce-pd; waiting for external resizer to expand the pvc
        Normal  Resizing                    3m52s              external-resizer pd.csi.storage.gke.io                                                            External resizer is resizing volume pvc-90f0633b-0938-4b66-a695-556bb8a9e943
        Normal  FileSystemResizeRequired    3m44s              external-resizer pd.csi.storage.gke.io                                                            Require file system resize of volume on node
        Normal  FileSystemResizeSuccessful  3m10s              kubelet                                                                                           MountVolume.NodeExpandVolume succeeded for volume "pvc-90f0633b-0938-4b66-a695-556bb8a9e943"
        ```

    Repeat step 2 for all the volumes of your cluster.

4. Now we have increased storage, but our StatefulSet 
    and Custom Resource are not in sync. Edit your Custom
    Resource with new storage settings and apply:

    ``` {.text .no-copy}
    spec:
      pxc:
        volumeSpec:
          persistentVolumeClaim:
            resources:
              requests:
                storage: <NEW STORAGE SIZE>
    ```

    Apply the Custom Resource:

    ``` {.bash data-prompt="$" }
    $ kubectl apply -f cr.yaml
    ```

5. Delete the StatefulSet to syncronize it with Custom
    Resource:

    ``` {.bash data-prompt="$" }
    $ kubectl delete sts <statefulset-name> --cascade=orphan
    ```

    The Pods will not go down and Operator is going to recreate
    the StatefulSet:

    ``` {.bash data-prompt="$" }
    $ kubectl get sts <statefulset-name>
    ```

    ??? example "Expected output"

        ``` {.text .no-copy}
        cluster1-pxc       3/3     39s
        ```

#### No Volume Expansion capability

Scaling the storage without Volume Expansion is also possible. We will
need to delete Pods one by one and their persistent volumes to resync 
the data to the new volumes. This can also be used to shrink the storage.

1. Edit the Custom Resource with the new storage size as follows:

    ``` {.text .no-copy}
    spec:
      pxc:
        volumeSpec:
          persistentVolumeClaim:
            resources:
              requests:
                storage: <NEW STORAGE SIZE>
    ```

    Apply the Custom Resource update in a usual way:

    ``` {.bash data-prompt="$" }
    $ kubectl apply -f deploy/cr.yaml
    ```

2. Delete the StatefulSet with the `orphan` option

    ``` {.bash data-prompt="$" }
    $ kubectl delete sts <statefulset-name> --cascade=orphan
    ```

    The Pods will not go down and the Operator is going to recreate
    the StatefulSet:

    ``` {.bash data-prompt="$" }
    $ kubectl get sts <statefulset-name>
    ```

    ??? example "Expected output"

        ``` {.text .no-copy}
        cluster1-pxc       3/3     39s
        ```

3. Scale up the cluster (Optional)

    Changing the storage size would require us to terminate the Pods, which 
    decreases the computational power of the cluster and might cause performance 
    issues. To improve performance during the operation we are going to 
    change the size of the cluster from 3 to 5 nodes:

    ```yaml
    ...
    spec:
      pxc:
        size: 5
    ```
    
    Apply the change:
    
    ``` {.bash data-prompt="$" }
    $ kubectl apply -f deploy/cr.yaml
    ```

    New Pods will already have new storage:
    
    ``` {.bash data-prompt="$" }
    $ kubectl get pvc
    ```

    ??? example "Expected output"

        ``` {.text .no-copy}
        NAME                     STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS   AGE
        datadir-cluster1-pxc-0   Bound    pvc-90f0633b-0938-4b66-a695-556bb8a9e943   10Gi       RWO            standard       110m
        datadir-cluster1-pxc-1   Bound    pvc-7409ea83-15b6-448f-a6a0-12a139e2f5cc   10Gi       RWO            standard       109m
        datadir-cluster1-pxc-2   Bound    pvc-90f0b2f8-9bba-4262-904c-1740fdd5511b   10Gi       RWO            standard       108m
        datadir-cluster1-pxc-3   Bound    pvc-439bee13-3b57-4582-b342-98281aca50ba   19Gi       RWO            standard       49m
        datadir-cluster1-pxc-4   Bound    pvc-2d4f3a60-4ec4-48a0-96cd-5243e2f05234   19Gi       RWO            standard       47m
        ```

4. Delete PVCs and Pods with old storage size one by one. Wait for data to sync 
    before you proceeding to the next node.

    ``` {.bash data-prompt="$" }
    $ kubectl delete pvc <PVC NAME>
    $ kubectl delete pod <POD NAME>
    ```
    The new PVC is going to be created along with the Pod. 

## Horizontal scaling

Size of the cluster is controlled by a [size key](operator.md#pxc-size) in the [Custom Resource options](operator.md#operator-custom-resource-options) configuration. That’s why scaling the cluster needs
nothing more but changing this option and applying the updated
configuration file. This may be done in a specifically saved config:

```yaml
...
spec:
  pxc:
    size: 5
```
    
Apply the change:
    
``` {.bash data-prompt="$" }
$ kubectl apply -f deploy/cr.yaml
```

Alternatively, you cana do it on the fly, using the following command:

``` {.bash data-prompt="$" }
$ kubectl scale --replicas=5 pxc/<CLUSTER NAME>
```

In this example we have changed the size of the Percona XtraDB Cluster
to `5` instances.

## Automated scaling

To automate horizontal scaling it is possible to use [Horizontal 
Pod Autoscaler (HPA)](https://kubernetes.io/docs/tasks/run-application/horizontal-pod-autoscale/). 
It will scale the Custom Resource itself, letting Operator to deal 
with everything else.

It is also possible to use [Kuvernetes Event-driven Autoscaling (KEDA)](https://keda.sh/), 
where you can apply more sophisticated logic for decision making on scaling.

For now it is not possible to use Vertical Pod Autoscaler (VPA) with 
the Operator due to the limitations it introduces for objects with 
owner references.
