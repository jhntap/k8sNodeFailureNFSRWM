# Trident Force Detach Test for NFS Backend with RWM access mode

This repository contains sample YAML files for a PersistentVolumeClaim (PVC) and a deployment that can be used to test the force detach feature of NetApp Trident during non-graceful Kubernetes node failures.

### Overview

Starting with Kubernetes v1.28, non-graceful node shutdown (NGNS) is enabled by default. This feature allows storage orchestrators like Trident to forcefully detach volumes from nodes that are not responding. This is particularly useful in scenarios where a node becomes unresponsive due to a crash, hardware failure, or network partitioning.

The force detach feature ensures that storage volumes can be quickly and safely detached from failed nodes and attached to other nodes, minimizing downtime and potential data corruption.

* Force detach is available for all of Trident drivers including ontap-san, ontap-san-economy, onatp-nas, and onatp-nas-economy. (Trident 25.02.0 and above)

### Testing Environment
All samples have been tested with the following environment:

* Trident with Kubernetes Advanced v6.0 (Trident 24.02.0 & Kubernetes 1.29.4) <https://labondemand.netapp.com/node/878>

Please note that while these samples have been tested in a specific environment, they may need to be adjusted for use in other setups.

### Test Steps for Force Detach Feature in Trident NFS Backend with ReadWriteMany Access Mode

The following steps outline how to test the force detach feature of the Trident with iSCSI backend. This feature ensures that a Persistent Volume Claim (PVC) can be detached from a failed node and reattached to a healthy node, allowing the pod that uses the PVC to be rescheduled.

#### 1. Enable Force Detach on Trident (Optional - If Force Detach is disabled)
Update Force Detach on Trident Orchestrator Custom Resource (torc) - This will restart Trident pods.

```
# kubectl get torc -n trident -o yaml | grep enableForceDetach
    enableForceDetach: false
      enableForceDetach: "false"

# kubectl patch torc trident -n trident --type=merge -p '{"spec":{"enableForceDetach":true}}'

# kubectl get torc -n trident -o yaml | grep enableForceDetach
    enableForceDetach: true
      enableForceDetach: "true"

# kubectl get pod -n trident
NAME                                  READY   STATUS    RESTARTS   AGE
trident-controller-6b7cdddf44-szdlk   6/6     Running   0          2m41s
trident-node-linux-9p6b2              2/2     Running   0          118s
trident-node-linux-pcsbk              2/2     Running   0          2m40s
trident-node-linux-v6vmv              2/2     Running   0          2m19s
```

#### 2. Create the PVC
Create a Persistent Volume Claim (PVC) that will be used by the pod. 

```
# pvc-nfs-rwm.yaml
kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  name: pvc-nfs-rwm
spec:
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 1Gi
  storageClassName: storage-class-nfs
```

Apply the PVC definition to the Kubernetes cluster:

```
# kubectl apply -f pvc-nfs-rwm.yaml
persistentvolumeclaim/pvc-iscsi created
# kubectl get pvc -o wide
NAME          STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS        VOLUMEATTRIBUTESCLASS   AGE   VOLUMEMODE
pvc-nfs-rwm   Bound    pvc-76520289-76f0-498a-8165-f358829750c3   1Gi        RWX            storage-class-nfs   <unset>                 13s   Filesystem

```

#### 3. Create the deployment
Create a deployment that uses the PVC created in the previous step. 

```
# test-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: test-deployment
  labels:
    app: test
spec:
  replicas: 3
  selector:
    matchLabels:
      app: test
  template:
    metadata:
      labels:
        app: test
    spec:
      volumes:
        - name: nfs-vol
          persistentVolumeClaim:
            claimName: pvc-nfs-rwm
      containers:
      - name: alpine
        image: alpine:3.19.1
        command:
          - /bin/sh
          - "-c"
          - "sleep 7d"
        volumeMounts:
          - mountPath: "/data"
            name: nfs-vol
      tolerations:
      - effect: NoExecute
        key: node.kubernetes.io/not-ready
        operator: Exists
        tolerationSeconds: 30
      - effect: NoExecute
        key: node.kubernetes.io/unreachable
        operator: Exists
        tolerationSeconds: 30
      nodeSelector:
        kubernetes.io/os: linux
```

Apply the deployment to the Kubernetes cluster:

```
# kubectl apply -f test-deployment.yaml
deployment.apps/test-deployment created
# ./verify_status.sh
kubectl get deployment.apps/test-deployment -o wide
NAME              READY   UP-TO-DATE   AVAILABLE   AGE     CONTAINERS   IMAGES          SELECTOR
test-deployment   3/3     3            3           2m58s   alpine       alpine:3.19.1   app=test
kubectl get replicaset.apps/test-deployment-66f546f986 -o wide
NAME                         DESIRED   CURRENT   READY   AGE     CONTAINERS   IMAGES          SELECTOR
test-deployment-66f546f986   3         3         3       2m58s   alpine       alpine:3.19.1   app=test,pod-template-hash=66f546f986
kubectl get pod -o wide
NAME                               READY   STATUS    RESTARTS   AGE     IP               NODE    NOMINATED NODE   READINESS GATES
test-deployment-66f546f986-57kkl   1/1     Running   0          2m58s   192.168.25.107   rhel3   <none>           <none>
test-deployment-66f546f986-6rw9n   1/1     Running   0          2m58s   192.168.26.14    rhel1   <none>           <none>
test-deployment-66f546f986-sdfgt   1/1     Running   0          2m58s   192.168.28.67    rhel2   <none>           <none>
kubectl get persistentvolumeclaim/pvc-nfs-rwm -o wide
NAME          STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS        VOLUMEATTRIBUTESCLASS   AGE     VOLUMEMODE
pvc-nfs-rwm   Bound    pvc-76520289-76f0-498a-8165-f358829750c3   1Gi        RWX            storage-class-nfs   <unset>                 4m27s   Filesystem
kubectl get volumeattachments -o wide
NAME                                                                   ATTACHER                PV                                         NODE    ATTACHED   AGE
csi-29d573214882e502742c2e079d8f50ec8842b018f96204ff9177591033f5ffc1   csi.trident.netapp.io   pvc-76520289-76f0-498a-8165-f358829750c3   rhel1   true       2m58s
csi-318e002ea6b1f3a8a5db0bcc837a699af1f65b596c800f98623819ad11101de2   csi.trident.netapp.io   pvc-76520289-76f0-498a-8165-f358829750c3   rhel3   true       2m58s
csi-f8029c43dd1a903c9bebe82221b15837f2961a482f304ae597a7e4f24f19c863   csi.trident.netapp.io   pvc-76520289-76f0-498a-8165-f358829750c3   rhel2   true       2m58s
```

Identify the status of nodes running the pod from the deployment.

```
# kubectl get pod -o wide
NAME                               READY   STATUS    RESTARTS   AGE     IP               NODE    NOMINATED NODE   READINESS GATES
test-deployment-66f546f986-57kkl   1/1     Running   0          2m58s   192.168.25.107   rhel3   <none>           <none>
test-deployment-66f546f986-6rw9n   1/1     Running   0          2m58s   192.168.26.14    rhel1   <none>           <none>
test-deployment-66f546f986-sdfgt   1/1     Running   0          2m58s   192.168.28.67    rhel2   <none>           <none>

# kubectl get node -o wide
NAME    STATUS   ROLES           AGE    VERSION   INTERNAL-IP    EXTERNAL-IP   OS-IMAGE                              KERNEL-VERSION                 CONTAINER-RUNTIME
rhel1   Ready    <none>          338d   v1.29.4   192.168.0.61   <none>        Red Hat Enterprise Linux 9.3 (Plow)   5.14.0-362.24.1.el9_3.x86_64   cri-o://1.30.0
rhel2   Ready    <none>          338d   v1.29.4   192.168.0.62   <none>        Red Hat Enterprise Linux 9.3 (Plow)   5.14.0-362.24.1.el9_3.x86_64   cri-o://1.30.0
rhel3   Ready    control-plane   338d   v1.29.4   192.168.0.63   <none>        Red Hat Enterprise Linux 9.3 (Plow)   5.14.0-362.24.1.el9_3.x86_64   cri-o://1.30.0

# tridentctl get node rhel1 -o yaml -n trident | grep publicationState
  publicationState: clean
# tridentctl get node rhel2 -o yaml -n trident | grep publicationState
  publicationState: clean
# tridentctl get node rhel3 -o yaml -n trident | grep publicationState
  publicationState: clean
```

Verify NFS PVC attachment to the node.

```
# ssh root@rhel1 mount | grep 192.168.0.131
192.168.0.131:/trident_pvc_76520289_76f0_498a_8165_f358829750c3 on /var/lib/kubelet/pods/e8264098-47be-49c1-ac50-110556264212/volumes/kubernetes.io~csi/pvc-76520289-76f0-498a-8165-f358829750c3/mount type nfs4 (rw,relatime,vers=4.2,rsize=65536,wsize=65536,namlen=255,hard,proto=tcp,timeo=600,retrans=2,sec=sys,clientaddr=192.168.0.61,local_lock=none,addr=192.168.0.131)
# ssh root@rhel2 mount | grep 192.168.0.131
192.168.0.131:/trident_pvc_76520289_76f0_498a_8165_f358829750c3 on /var/lib/kubelet/pods/e68aabbb-1966-425d-8116-a99f1107e6ed/volumes/kubernetes.io~csi/pvc-76520289-76f0-498a-8165-f358829750c3/mount type nfs4 (rw,relatime,vers=4.2,rsize=65536,wsize=65536,namlen=255,hard,proto=tcp,timeo=600,retrans=2,sec=sys,clientaddr=192.168.0.62,local_lock=none,addr=192.168.0.131)
# ssh root@rhel3 mount | grep 192.168.0.131
192.168.0.131:/trident_pvc_76520289_76f0_498a_8165_f358829750c3 on /var/lib/kubelet/pods/0560bcda-08e8-497f-b8bf-b0a8aac0a39d/volumes/kubernetes.io~csi/pvc-76520289-76f0-498a-8165-f358829750c3/mount type nfs4 (rw,relatime,vers=4.2,rsize=65536,wsize=65536,namlen=255,hard,proto=tcp,timeo=600,retrans=2,sec=sys,clientaddr=192.168.0.63,local_lock=none,addr=192.168.0.131)

# nfs connected-clients show

     Node: cluster1-01
  Vserver: nassvm
  Data-Ip: 192.168.0.131
Client-Ip      Protocol Volume    Policy   Idle-Time    Local-Reqs Remote-Reqs
-------------- -------- --------- -------- ------------ ---------- ----------
Trunking
-------
192.168.0.61   nfs4.2   nassvm_root default 26m 54s     9        0     false
192.168.0.61   nfs4.2   trident_pvc_76520289_76f0_498a_8165_f358829750c3 trident_pvc_76520289_76f0_498a_8165_f358829750c3 46s 42    0 false
192.168.0.62   nfs4.2   nassvm_root default 26m 58s     9        0     false
192.168.0.62   nfs4.2   trident_pvc_76520289_76f0_498a_8165_f358829750c3 trident_pvc_76520289_76f0_498a_8165_f358829750c3 1m 22s 43    0 false
192.168.0.63   nfs4.2   nassvm_root default 26m 58s     9        0     false
192.168.0.63   nfs4.2   trident_pvc_76520289_76f0_498a_8165_f358829750c3 trident_pvc_76520289_76f0_498a_8165_f358829750c3 2s 44    0 false
6 entries were displayed.
```

#### 4. Shutdown Node Running the Pod

```
[root@rhel1 ~]# shutdown -h now
```

Chech the status of the pod.

```
# kubectl get node
NAME    STATUS     ROLES           AGE    VERSION
rhel1   NotReady   <none>          338d   v1.29.4
rhel2   Ready      <none>          338d   v1.29.4
rhel3   Ready      control-plane   338d   v1.29.4

# ./verify_status.sh
kubectl get deployment.apps/test-deployment -o wide
NAME              READY   UP-TO-DATE   AVAILABLE   AGE   CONTAINERS   IMAGES          SELECTOR
test-deployment   3/3     3            3           31m   alpine       alpine:3.19.1   app=test
kubectl get replicaset.apps/test-deployment-66f546f986 -o wide
NAME                         DESIRED   CURRENT   READY   AGE   CONTAINERS   IMAGES          SELECTOR
test-deployment-66f546f986   3         3         3       31m   alpine       alpine:3.19.1   app=test,pod-template-hash=66f546f986
kubectl get pod -o wide
NAME                               READY   STATUS        RESTARTS   AGE   IP               NODE    NOMINATED NODE   READINESS GATES
test-deployment-66f546f986-57kkl   1/1     Running       0          31m   192.168.25.107   rhel3   <none>           <none>
test-deployment-66f546f986-6rw9n   1/1     Terminating   0          31m   192.168.26.14    rhel1   <none>           <none>
test-deployment-66f546f986-sdfgt   1/1     Running       0          31m   192.168.28.67    rhel2   <none>           <none>
test-deployment-66f546f986-vmwzc   1/1     Running       0          63s   192.168.28.127   rhel2   <none>           <none>
kubectl get persistentvolumeclaim/pvc-nfs-rwm -o wide
NAME          STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS        VOLUMEATTRIBUTESCLASS   AGE   VOLUMEMODE
pvc-nfs-rwm   Bound    pvc-76520289-76f0-498a-8165-f358829750c3   1Gi        RWX            storage-class-nfs   <unset>                 33m   Filesystem
kubectl get volumeattachments -o wide
NAME                                                                   ATTACHER                PV                                         NODE    ATTACHED   AGE
csi-29d573214882e502742c2e079d8f50ec8842b018f96204ff9177591033f5ffc1   csi.trident.netapp.io   pvc-76520289-76f0-498a-8165-f358829750c3   rhel1   true       31m
csi-318e002ea6b1f3a8a5db0bcc837a699af1f65b596c800f98623819ad11101de2   csi.trident.netapp.io   pvc-76520289-76f0-498a-8165-f358829750c3   rhel3   true       31m
csi-f8029c43dd1a903c9bebe82221b15837f2961a482f304ae597a7e4f24f19c863   csi.trident.netapp.io   pvc-76520289-76f0-498a-8165-f358829750c3   rhel2   true       31m

# kubectl describe $(kubectl get pod -o name)
Name:                      test-deployment-57fb685899-66wmp
Namespace:                 default
Priority:                  0
Service Account:           default
Node:                      rhel1/192.168.0.61
Start Time:                Wed, 26 Mar 2025 02:27:10 +0000
Labels:                    app=test
                           pod-template-hash=57fb685899
Annotations:               cni.projectcalico.org/containerID: 1a109ecbe9d743329c23ab2b50ac14b15cfa913b58e60996cc4595d8293201d6
                           cni.projectcalico.org/podIP: 192.168.26.9/32
                           cni.projectcalico.org/podIPs: 192.168.26.9/32
Status:                    Terminating (lasts 105s)
Termination Grace Period:  30s
IP:                        192.168.26.9
IPs:
  IP:           192.168.26.9
Controlled By:  ReplicaSet/test-deployment-57fb685899
Containers:
  alpine:
    Container ID:  cri-o://1c42bb45f739f0a0918c0c7f73ac06cfb640cd1943295026f3c9b28ee2f902cf
    Image:         alpine:3.19.1
    Image ID:      docker.io/library/alpine@sha256:6457d53fb065d6f250e1504b9bc42d5b6c65941d57532c072d929dd0628977d0
    Port:          <none>
    Host Port:     <none>
    Command:
      /bin/sh
      -c
      sleep 7d
    State:          Running
      Started:      Wed, 26 Mar 2025 02:27:18 +0000
    Ready:          True
    Restart Count:  0
    Environment:    <none>
    Mounts:
      /data from iscsi-vol (rw)
      /var/run/secrets/kubernetes.io/serviceaccount from kube-api-access-p4c6s (ro)
Conditions:
  Type                        Status
  PodReadyToStartContainers   True
  Initialized                 True
  Ready                       False
  ContainersReady             True
  PodScheduled                True
  DisruptionTarget            True
Volumes:
  iscsi-vol:
    Type:       PersistentVolumeClaim (a reference to a PersistentVolumeClaim in the same namespace)
    ClaimName:  pvc-iscsi
    ReadOnly:   false
  kube-api-access-p4c6s:
    Type:                    Projected (a volume that contains injected data from multiple sources)
    TokenExpirationSeconds:  3607
    ConfigMapName:           kube-root-ca.crt
    ConfigMapOptional:       <nil>
    DownwardAPI:             true
QoS Class:                   BestEffort
Node-Selectors:              kubernetes.io/os=linux
Tolerations:                 node.kubernetes.io/not-ready:NoExecute op=Exists for 30s
                             node.kubernetes.io/unreachable:NoExecute op=Exists for 30s
Events:
  Type     Reason                  Age    From                     Message
  ----     ------                  ----   ----                     -------
  Normal   Scheduled               23m    default-scheduler        Successfully assigned default/test-deployment-57fb685899-66wmp to rhel1
  Normal   SuccessfulAttachVolume  23m    attachdetach-controller  AttachVolume.Attach succeeded for volume "pvc-2a11e307-1582-4650-bde6-bb9c12e55661"
  Normal   Pulled                  23m    kubelet                  Container image "alpine:3.19.1" already present on machine
  Normal   Created                 23m    kubelet                  Created container alpine
  Normal   Started                 23m    kubelet                  Started container alpine
  Warning  NodeNotReady            2m51s  node-controller          Node is not ready

# kubectl describe pod test-deployment-66f546f986-vmwzc
Name:             test-deployment-66f546f986-vmwzc
Namespace:        default
Priority:         0
Service Account:  default
Node:             rhel2/192.168.0.62
Start Time:       Tue, 01 Apr 2025 06:50:47 +0000
Labels:           app=test
                  pod-template-hash=66f546f986
Annotations:      cni.projectcalico.org/containerID: c2e98a475997972ae0811182662aca674765b0c6a421272411fbcce3d7e5564f
                  cni.projectcalico.org/podIP: 192.168.28.127/32
                  cni.projectcalico.org/podIPs: 192.168.28.127/32
Status:           Running
IP:               192.168.28.127
IPs:
  IP:           192.168.28.127
Controlled By:  ReplicaSet/test-deployment-66f546f986
Containers:
  alpine:
    Container ID:  cri-o://c21a9fc807af2cc617541e5a816ed37c8f588b349c61ac324a4d67a20d99dc9e
    Image:         alpine:3.19.1
    Image ID:      docker.io/library/alpine@sha256:6457d53fb065d6f250e1504b9bc42d5b6c65941d57532c072d929dd0628977d0
    Port:          <none>
    Host Port:     <none>
    Command:
      /bin/sh
      -c
      sleep 7d
    State:          Running
      Started:      Tue, 01 Apr 2025 06:50:48 +0000
    Ready:          True
    Restart Count:  0
    Environment:    <none>
    Mounts:
      /data from nfs-vol (rw)
      /var/run/secrets/kubernetes.io/serviceaccount from kube-api-access-9dkb2 (ro)
Conditions:
  Type                        Status
  PodReadyToStartContainers   True
  Initialized                 True
  Ready                       True
  ContainersReady             True
  PodScheduled                True
Volumes:
  nfs-vol:
    Type:       PersistentVolumeClaim (a reference to a PersistentVolumeClaim in the same namespace)
    ClaimName:  pvc-nfs-rwm
    ReadOnly:   false
  kube-api-access-9dkb2:
    Type:                    Projected (a volume that contains injected data from multiple sources)
    TokenExpirationSeconds:  3607
    ConfigMapName:           kube-root-ca.crt
    ConfigMapOptional:       <nil>
    DownwardAPI:             true
QoS Class:                   BestEffort
Node-Selectors:              kubernetes.io/os=linux
Tolerations:                 node.kubernetes.io/not-ready:NoExecute op=Exists for 30s
                             node.kubernetes.io/unreachable:NoExecute op=Exists for 30s
Events:
  Type    Reason     Age    From               Message
  ----    ------     ----   ----               -------
  Normal  Scheduled  3m35s  default-scheduler  Successfully assigned default/test-deployment-66f546f986-vmwzc to rhel2
  Normal  Pulled     3m32s  kubelet            Container image "alpine:3.19.1" already present on machine
  Normal  Created    3m32s  kubelet            Created container alpine
  Normal  Started    3m32s  kubelet            Started container alpine
```

#### 5. Apply Taint to the Failed Node

```
# kubectl taint nodes rhel1 node.kubernetes.io/out-of-service=nodeshutdown:NoExecute
node/rhel1 tainted

# kubectl get nodes -o json | jq '.items[] | {name: .metadata.name, taints: .spec.taints}'
{
  "name": "rhel1",
  "taints": [
    {
      "effect": "NoExecute",
      "key": "node.kubernetes.io/out-of-service",
      "value": "nodeshutdown"
    },
    {
      "effect": "NoSchedule",
      "key": "node.kubernetes.io/unreachable",
      "timeAdded": "2025-04-01T06:50:10Z"
    },
    {
      "effect": "NoExecute",
      "key": "node.kubernetes.io/unreachable",
      "timeAdded": "2025-04-01T06:50:15Z"
    }
  ]
}
{
  "name": "rhel2",
  "taints": null
}
{
  "name": "rhel3",
  "taints": null
}

# tridentctl get node rhel1 -o yaml -n trident | grep publicationState
  publicationState: dirty
# tridentctl get node rhel2 -o yaml -n trident | grep publicationState
  publicationState: clean
# tridentctl get node rhel3 -o yaml -n trident | grep publicationState
  publicationState: clean
```

#### 6. Verify PVC Detachment from the Failed Node

Verify that the pod has been rescheduled onto another node and that it has successfully mounted the NFS PVC.

```
# ./verify_status.sh
kubectl get deployment.apps/test-deployment -o wide
NAME              READY   UP-TO-DATE   AVAILABLE   AGE   CONTAINERS   IMAGES          SELECTOR
test-deployment   3/3     3            3           37m   alpine       alpine:3.19.1   app=test
kubectl get replicaset.apps/test-deployment-66f546f986 -o wide
NAME                         DESIRED   CURRENT   READY   AGE   CONTAINERS   IMAGES          SELECTOR
test-deployment-66f546f986   3         3         3       37m   alpine       alpine:3.19.1   app=test,pod-template-hash=66f546f986
kubectl get pod -o wide
NAME                               READY   STATUS    RESTARTS   AGE     IP               NODE    NOMINATED NODE   READINESS GATES
test-deployment-66f546f986-57kkl   1/1     Running   0          37m     192.168.25.107   rhel3   <none>           <none>
test-deployment-66f546f986-sdfgt   1/1     Running   0          37m     192.168.28.67    rhel2   <none>           <none>
test-deployment-66f546f986-vmwzc   1/1     Running   0          6m56s   192.168.28.127   rhel2   <none>           <none>
kubectl get persistentvolumeclaim/pvc-nfs-rwm -o wide
NAME          STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS        VOLUMEATTRIBUTESCLASS   AGE   VOLUMEMODE
pvc-nfs-rwm   Bound    pvc-76520289-76f0-498a-8165-f358829750c3   1Gi        RWX            storage-class-nfs   <unset>                 39m   Filesystem
kubectl get volumeattachments -o wide
NAME                                                                   ATTACHER                PV                                         NODE    ATTACHED   AGE
csi-318e002ea6b1f3a8a5db0bcc837a699af1f65b596c800f98623819ad11101de2   csi.trident.netapp.io   pvc-76520289-76f0-498a-8165-f358829750c3   rhel3   true       37m
csi-f8029c43dd1a903c9bebe82221b15837f2961a482f304ae597a7e4f24f19c863   csi.trident.netapp.io   pvc-76520289-76f0-498a-8165-f358829750c3   rhel2   true       37m

# nfs connected-clients show

     Node: cluster1-01
  Vserver: nassvm
  Data-Ip: 192.168.0.131
Client-Ip      Protocol Volume    Policy   Idle-Time    Local-Reqs Remote-Reqs
-------------- -------- --------- -------- ------------ ---------- ----------
Trunking
-------
192.168.0.61   nfs4.2   nassvm_root default 38m 40s     9        0     false
192.168.0.61   nfs4.2   trident_pvc_76520289_76f0_498a_8165_f358829750c3 trident_pvc_76520289_76f0_498a_8165_f358829750c3 10m 34s 44    0 false
192.168.0.62   nfs4.2   nassvm_root default 8m 13s      18       0     false
192.168.0.62   nfs4.2   trident_pvc_76520289_76f0_498a_8165_f358829750c3 trident_pvc_76520289_76f0_498a_8165_f358829750c3 9s 73    0 false
192.168.0.63   nfs4.2   nassvm_root default 38m 44s     9        0     false
192.168.0.63   nfs4.2   trident_pvc_76520289_76f0_498a_8165_f358829750c3 trident_pvc_76520289_76f0_498a_8165_f358829750c3 53s 60    0 false
6 entries were displayed.

# kubectl describe pod test-deployment-66f546f986-vmwzc
Name:             test-deployment-66f546f986-vmwzc
Namespace:        default
Priority:         0
Service Account:  default
Node:             rhel2/192.168.0.62
Start Time:       Tue, 01 Apr 2025 06:50:47 +0000
Labels:           app=test
                  pod-template-hash=66f546f986
Annotations:      cni.projectcalico.org/containerID: c2e98a475997972ae0811182662aca674765b0c6a421272411fbcce3d7e5564f
                  cni.projectcalico.org/podIP: 192.168.28.127/32
                  cni.projectcalico.org/podIPs: 192.168.28.127/32
Status:           Running
IP:               192.168.28.127
IPs:
  IP:           192.168.28.127
Controlled By:  ReplicaSet/test-deployment-66f546f986
Containers:
  alpine:
    Container ID:  cri-o://c21a9fc807af2cc617541e5a816ed37c8f588b349c61ac324a4d67a20d99dc9e
    Image:         alpine:3.19.1
    Image ID:      docker.io/library/alpine@sha256:6457d53fb065d6f250e1504b9bc42d5b6c65941d57532c072d929dd0628977d0
    Port:          <none>
    Host Port:     <none>
    Command:
      /bin/sh
      -c
      sleep 7d
    State:          Running
      Started:      Tue, 01 Apr 2025 06:50:48 +0000
    Ready:          True
    Restart Count:  0
    Environment:    <none>
    Mounts:
      /data from nfs-vol (rw)
      /var/run/secrets/kubernetes.io/serviceaccount from kube-api-access-9dkb2 (ro)
Conditions:
  Type                        Status
  PodReadyToStartContainers   True
  Initialized                 True
  Ready                       True
  ContainersReady             True
  PodScheduled                True
Volumes:
  nfs-vol:
    Type:       PersistentVolumeClaim (a reference to a PersistentVolumeClaim in the same namespace)
    ClaimName:  pvc-nfs-rwm
    ReadOnly:   false
  kube-api-access-9dkb2:
    Type:                    Projected (a volume that contains injected data from multiple sources)
    TokenExpirationSeconds:  3607
    ConfigMapName:           kube-root-ca.crt
    ConfigMapOptional:       <nil>
    DownwardAPI:             true
QoS Class:                   BestEffort
Node-Selectors:              kubernetes.io/os=linux
Tolerations:                 node.kubernetes.io/not-ready:NoExecute op=Exists for 30s
                             node.kubernetes.io/unreachable:NoExecute op=Exists for 30s
Events:
  Type    Reason     Age   From               Message
  ----    ------     ----  ----               -------
  Normal  Scheduled  20m   default-scheduler  Successfully assigned default/test-deployment-66f546f986-vmwzc to rhel2
  Normal  Pulled     20m   kubelet            Container image "alpine:3.19.1" already present on machine
  Normal  Created    20m   kubelet            Created container alpine
  Normal  Started    20m   kubelet            Started container alpine
```

#### 8. Verify Trident Node State after Recovery (Optional)

Power up the failed node and verify the node is in Ready state

```
# kubectl get node
NAME    STATUS   ROLES           AGE    VERSION
rhel1   Ready    <none>          338d   v1.29.4
rhel2   Ready    <none>          338d   v1.29.4
rhel3   Ready    control-plane   338d   v1.29.4
```

Remove the taint and verify the Trident node state is Clean state

```
# kubectl taint nodes rhel1 node.kubernetes.io/out-of-service=nodeshutdown:NoExecute-
node/rhel1 untainted

# kubectl get nodes -o json | jq '.items[] | {name: .metadata.name, taints: .spec.taints}'
{
  "name": "rhel1",
  "taints": null
}
{
  "name": "rhel2",
  "taints": null
}
{
  "name": "rhel3",
  "taints": null
}

#  tridentctl get node rhel1 -o yaml -n trident | grep publicationState
  publicationState: clean
#  tridentctl get node rhel2 -o yaml -n trident | grep publicationState
  publicationState: clean
#  tridentctl get node rhel3 -o yaml -n trident | grep publicationState
  publicationState: clean
```

#### 9. Redistribute the Pod Manually (Optional)

```
# k get pod -o wide
NAME                               READY   STATUS    RESTARTS   AGE   IP               NODE    NOMINATED NODE   READINESS GATES
test-deployment-66f546f986-57kkl   1/1     Running   0          66m   192.168.25.107   rhel3   <none>           <none>
test-deployment-66f546f986-sdfgt   1/1     Running   0          66m   192.168.28.67    rhel2   <none>           <none>
test-deployment-66f546f986-vmwzc   1/1     Running   0          35m   192.168.28.127   rhel2   <none>           <none>

# kubectl delete pod test-deployment-66f546f986-vmwzc
pod "test-deployment-66f546f986-vmwzc" deleted

# kubectl get pod -o wide
NAME                               READY   STATUS    RESTARTS   AGE   IP               NODE    NOMINATED NODE   READINESS GATES
test-deployment-66f546f986-57kkl   1/1     Running   0          68m   192.168.25.107   rhel3   <none>           <none>
test-deployment-66f546f986-lhgvn   1/1     Running   0          41s   192.168.26.15    rhel1   <none>           <none>
test-deployment-66f546f986-sdfgt   1/1     Running   0          68m   192.168.28.67    rhel2   <none>           <none>
```




