# Network policies post OVN migration in OCP 4
Effects on network policies post OVN migration

## Cluster Details


```
awezsonde@Awezs-Mac-Studio ~ % oc get clusterversion         
NAME      VERSION   AVAILABLE   PROGRESSING   SINCE   STATUS
version   4.14.31   True        False         6d22h   Cluster version is 4.14.31



awezsonde@Awezs-Mac-Studio ~ % oc get nodes                  
NAME                                                STATUS   ROLES                  AGE     VERSION
master-0.ovnmigration.lab.upshift.rdu2.redhat.com   Ready    control-plane,master   6d22h   v1.27.14+7852426
master-1.ovnmigration.lab.upshift.rdu2.redhat.com   Ready    control-plane,master   6d22h   v1.27.14+7852426
master-2.ovnmigration.lab.upshift.rdu2.redhat.com   Ready    control-plane,master   6d22h   v1.27.14+7852426
worker-0.ovnmigration.lab.upshift.rdu2.redhat.com   Ready    worker                 6d22h   v1.27.14+7852426
worker-1.ovnmigration.lab.upshift.rdu2.redhat.com   Ready    worker                 6d22h   v1.27.14+7852426



awezsonde@Awezs-Mac-Studio ~ % oc get network cluster -o yaml
apiVersion: config.openshift.io/v1
kind: Network
metadata:
  creationTimestamp: "2024-12-26T10:38:05Z"
  generation: 2
  name: cluster
  resourceVersion: "5160"
  uid: c3628f35-000d-4316-8860-cd007087bdda
spec:
  clusterNetwork:
  - cidr: 10.128.0.0/14
    hostPrefix: 23
  externalIP:
    policy: {}
  networkType: OpenShiftSDN
  serviceNetwork:
  - 172.30.0.0/16
status:
  clusterNetwork:
  - cidr: 10.128.0.0/14
    hostPrefix: 23
  clusterNetworkMTU: 1450
  networkType: OpenShiftSDN
  serviceNetwork:
  - 172.30.0.0/16

```

## Create two projects for the facilitation of network policies

Two project `projecta` and `projectb` 

```
awezsonde@Awezs-Mac-Studio ~ % oc new-project projecta
Now using project "projecta" on server "https://api.ovnmigration.lab.upshift.rdu2.redhat.com:6443".

You can add applications to this project with the 'new-app' command. For example, try:

    oc new-app rails-postgresql-example

to build a new example application in Ruby. Or use kubectl to deploy a simple Kubernetes application:

    kubectl create deployment hello-node --image=registry.k8s.io/e2e-test-images/agnhost:2.43 -- /agnhost serve-hostname




awezsonde@Awezs-Mac-Studio ~ % oc new-project projectb
Now using project "projectb" on server "https://api.ovnmigration.lab.upshift.rdu2.redhat.com:6443".

You can add applications to this project with the 'new-app' command. For example, try:

    oc new-app rails-postgresql-example

to build a new example application in Ruby. Or use kubectl to deploy a simple Kubernetes application:

    kubectl create deployment hello-node --image=registry.k8s.io/e2e-test-images/agnhost:2.43 -- /agnhost serve-hostname

```



## Create an application in `projecta` and `projectb` to test the connectivity between them



```
awezsonde@Awezs-Mac-Studio ~ % oc create -f applicationa.yaml -n projecta
deployment.apps/ovn-migration-network-policy-test created
```



```
awezsonde@Awezs-Mac-Studio ~ % oc create -f applicationb.yaml -n projectb
deployment.apps/ovn-migration-network-policy-test created
```


## Content of the deployment file 

```
awezsonde@Awezs-Mac-Studio ~ % cat applicationa.yaml 
apiVersion: apps/v1
kind: Deployment
metadata:
  name: ovn-migration-network-policy-test
spec:
  replicas: 1
  selector:
    matchLabels:
      app: projecta-app
  template:
    metadata:
      labels:
        app: projecta-app
    spec:
      containers:
      - name: applicationa
        image: registry.redhat.io/openshift4/network-tools-rhel9@sha256:5329a0b15459029b955c0347e45ff0e01d3433623f33121575dcf4ece0552bca
        command: ["/bin/sh", "-ec", "sleep 1000"]









awezsonde@Awezs-Mac-Studio ~ % cat applicationb.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: ovn-migration-network-policy-test
spec:
  replicas: 1
  selector:
    matchLabels:
      app: projectb-app
  template:
    metadata:
      labels:
        app: projectb-app
    spec:
      containers:
      - name: applicationb
        image: registry.redhat.io/openshift4/network-tools-rhel9@sha256:5329a0b15459029b955c0347e45ff0e01d3433623f33121575dcf4ece0552bca
        command: ["/bin/sh", "-ec", "sleep 1000"]

```


## Verify the pod IPs on `projecta` and `projectb`

```
awezsonde@Awezs-Mac-Studio ~ % oc get pods -o wide -n projecta 
NAME                                                 READY   STATUS    RESTARTS   AGE     IP            NODE                                                NOMINATED NODE   READINESS GATES
ovn-migration-network-policy-test-5d6d5d9fff-8xbxd   1/1     Running   0          4m33s   10.128.2.49   worker-1.ovnmigration.lab.upshift.rdu2.redhat.com   <none>           <none>






awezsonde@Awezs-Mac-Studio ~ % oc get pods -o wide -n projectb 
NAME                                                 READY   STATUS    RESTARTS   AGE    IP            NODE                                                NOMINATED NODE   READINESS GATES
ovn-migration-network-policy-test-55c4854885-vc846   1/1     Running   0          109s   10.128.2.50   worker-1.ovnmigration.lab.upshift.rdu2.redhat.com   <none>           <none>

```



## Ping from `projecta` pod to `projectb`

### Ping is successfull from 10.128.2.49 to 10.128.2.50

```
awezsonde@Awezs-Mac-Studio ~ % oc rsh ovn-migration-network-policy-test-5d6d5d9fff-8xbxd         
sh-5.1$ 
sh-5.1$ ping 10.128.2.50
PING 10.128.2.50 (10.128.2.50) 56(84) bytes of data.
64 bytes from 10.128.2.50: icmp_seq=1 ttl=64 time=0.434 ms
64 bytes from 10.128.2.50: icmp_seq=2 ttl=64 time=0.065 ms
64 bytes from 10.128.2.50: icmp_seq=3 ttl=64 time=0.048 ms
64 bytes from 10.128.2.50: icmp_seq=4 ttl=64 time=0.081 ms
64 bytes from 10.128.2.50: icmp_seq=5 ttl=64 time=0.053 ms
^C
--- 10.128.2.50 ping statistics ---
5 packets transmitted, 5 received, 0% packet loss, time 4118ms
rtt min/avg/max/mdev = 0.048/0.136/0.434/0.149 ms

```

## Introduce network policy

### The network policy will be applied on `projectb` and prevent traffic from `projecta` 

```
awezsonde@Awezs-Mac-Studio ~ % cat networkpolicy-projectb.yaml 
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: deny-projecta-traffic
  namespace: projectb
spec:
  podSelector: {} # Apply to all pods in ProjectB
  ingress:
    - from:
        - namespaceSelector:
            matchLabels:
              name: projecta # Only allow traffic from namespaces other than ProjectA




awezsonde@Awezs-Mac-Studio ~ % oc create -f networkpolicy-projectb.yaml -n projectb
networkpolicy.networking.k8s.io/deny-projecta-traffic created
```

## Ping test again from projecta to projectb

### Ping not successful , suggests network policy is in place

```
awezsonde@Awezs-Mac-Studio ~ % oc rsh ovn-migration-network-policy-test-5d6d5d9fff-8xbxd 
sh-5.1$ ping 10.128.2.50
PING 10.128.2.50 (10.128.2.50) 56(84) bytes of data.


^C
--- 10.128.2.50 ping statistics ---
20 packets transmitted, 0 received, 100% packet loss, time 19477ms
```


# Start  OVN migration 


## Prerequisites

### Backup 

```
awezsonde@Awezs-Mac-Studio ~ % oc get Network.config.openshift.io cluster -o yaml > cluster-openshift-sdn.yaml


awezsonde@Awezs-Mac-Studio ~ % cat cluster-openshift-sdn.yaml 
apiVersion: config.openshift.io/v1
kind: Network
metadata:
  creationTimestamp: "2024-12-26T10:38:05Z"
  generation: 2
  name: cluster
  resourceVersion: "5160"
  uid: c3628f35-000d-4316-8860-cd007087bdda
spec:
  clusterNetwork:
  - cidr: 10.128.0.0/14
    hostPrefix: 23
  externalIP:
    policy: {}
  networkType: OpenShiftSDN
  serviceNetwork:
  - 172.30.0.0/16
status:
  clusterNetwork:
  - cidr: 10.128.0.0/14
    hostPrefix: 23
  clusterNetworkMTU: 1450
  networkType: OpenShiftSDN
  serviceNetwork:
  - 172.30.0.0/16

```
### Script to verify if the timeout for SDN is set to `0`

```
awezsonde@Awezs-Mac-Studio ~ % cat timeout.sh 
#!/bin/bash

if [ -n "$OVN_SDN_MIGRATION_TIMEOUT" ] && [ "$OVN_SDN_MIGRATION_TIMEOUT" = "0s" ]; then
    unset OVN_SDN_MIGRATION_TIMEOUT
fi

#loops the timeout command of the script to repeatedly check the cluster Operators until all are available.

co_timeout=${OVN_SDN_MIGRATION_TIMEOUT:-1200s}
timeout "$co_timeout" bash <<EOT
until
  oc wait co --all --for='condition=AVAILABLE=True' --timeout=10s && \
  oc wait co --all --for='condition=PROGRESSING=False' --timeout=10s && \
  oc wait co --all --for='condition=DEGRADED=False' --timeout=10s;
do
  sleep 10
  echo "Some ClusterOperators Degraded=False,Progressing=True,or Available=False";
done
EOT



awezsonde@Awezs-Mac-Studio ~ %  chmod +x ./timeout.sh


awezsonde@Awezs-Mac-Studio ~ %  ./timeout.sh 
clusteroperator.config.openshift.io/authentication condition met
clusteroperator.config.openshift.io/baremetal condition met
clusteroperator.config.openshift.io/cloud-controller-manager condition met
clusteroperator.config.openshift.io/cloud-credential condition met
clusteroperator.config.openshift.io/cluster-autoscaler condition met
clusteroperator.config.openshift.io/config-operator condition met
clusteroperator.config.openshift.io/console condition met
clusteroperator.config.openshift.io/control-plane-machine-set condition met
clusteroperator.config.openshift.io/csi-snapshot-controller condition met
clusteroperator.config.openshift.io/dns condition met
clusteroperator.config.openshift.io/etcd condition met
clusteroperator.config.openshift.io/image-registry condition met
clusteroperator.config.openshift.io/ingress condition met
clusteroperator.config.openshift.io/insights condition met
clusteroperator.config.openshift.io/kube-apiserver condition met
clusteroperator.config.openshift.io/kube-controller-manager condition met
clusteroperator.config.openshift.io/kube-scheduler condition met
clusteroperator.config.openshift.io/kube-storage-version-migrator condition met
clusteroperator.config.openshift.io/machine-api condition met
clusteroperator.config.openshift.io/machine-approver condition met
clusteroperator.config.openshift.io/machine-config condition met
clusteroperator.config.openshift.io/marketplace condition met
clusteroperator.config.openshift.io/monitoring condition met
clusteroperator.config.openshift.io/network condition met
clusteroperator.config.openshift.io/node-tuning condition met
clusteroperator.config.openshift.io/openshift-apiserver condition met
clusteroperator.config.openshift.io/openshift-controller-manager condition met
clusteroperator.config.openshift.io/openshift-samples condition met
clusteroperator.config.openshift.io/operator-lifecycle-manager condition met
clusteroperator.config.openshift.io/operator-lifecycle-manager-catalog condition met
clusteroperator.config.openshift.io/operator-lifecycle-manager-packageserver condition met
clusteroperator.config.openshift.io/service-ca condition met
clusteroperator.config.openshift.io/storage condition met
clusteroperator.config.openshift.io/authentication condition met
clusteroperator.config.openshift.io/baremetal condition met
clusteroperator.config.openshift.io/cloud-controller-manager condition met
clusteroperator.config.openshift.io/cloud-credential condition met
clusteroperator.config.openshift.io/cluster-autoscaler condition met
clusteroperator.config.openshift.io/config-operator condition met
clusteroperator.config.openshift.io/console condition met
clusteroperator.config.openshift.io/control-plane-machine-set condition met
clusteroperator.config.openshift.io/csi-snapshot-controller condition met
clusteroperator.config.openshift.io/dns condition met
clusteroperator.config.openshift.io/etcd condition met
clusteroperator.config.openshift.io/image-registry condition met
clusteroperator.config.openshift.io/ingress condition met
clusteroperator.config.openshift.io/insights condition met
clusteroperator.config.openshift.io/kube-apiserver condition met
clusteroperator.config.openshift.io/kube-controller-manager condition met
clusteroperator.config.openshift.io/kube-scheduler condition met
clusteroperator.config.openshift.io/kube-storage-version-migrator condition met
clusteroperator.config.openshift.io/machine-api condition met
clusteroperator.config.openshift.io/machine-approver condition met
clusteroperator.config.openshift.io/machine-config condition met
clusteroperator.config.openshift.io/marketplace condition met
clusteroperator.config.openshift.io/monitoring condition met
clusteroperator.config.openshift.io/network condition met
clusteroperator.config.openshift.io/node-tuning condition met
clusteroperator.config.openshift.io/openshift-apiserver condition met
clusteroperator.config.openshift.io/openshift-controller-manager condition met
clusteroperator.config.openshift.io/openshift-samples condition met
clusteroperator.config.openshift.io/operator-lifecycle-manager condition met
clusteroperator.config.openshift.io/operator-lifecycle-manager-catalog condition met
clusteroperator.config.openshift.io/operator-lifecycle-manager-packageserver condition met
clusteroperator.config.openshift.io/service-ca condition met
clusteroperator.config.openshift.io/storage condition met
clusteroperator.config.openshift.io/authentication condition met
clusteroperator.config.openshift.io/baremetal condition met
clusteroperator.config.openshift.io/cloud-controller-manager condition met
clusteroperator.config.openshift.io/cloud-credential condition met
clusteroperator.config.openshift.io/cluster-autoscaler condition met
clusteroperator.config.openshift.io/config-operator condition met
clusteroperator.config.openshift.io/console condition met
clusteroperator.config.openshift.io/control-plane-machine-set condition met
clusteroperator.config.openshift.io/csi-snapshot-controller condition met
clusteroperator.config.openshift.io/dns condition met
clusteroperator.config.openshift.io/etcd condition met
clusteroperator.config.openshift.io/image-registry condition met
clusteroperator.config.openshift.io/ingress condition met
clusteroperator.config.openshift.io/insights condition met
clusteroperator.config.openshift.io/kube-apiserver condition met
clusteroperator.config.openshift.io/kube-controller-manager condition met
clusteroperator.config.openshift.io/kube-scheduler condition met
clusteroperator.config.openshift.io/kube-storage-version-migrator condition met
clusteroperator.config.openshift.io/machine-api condition met
clusteroperator.config.openshift.io/machine-approver condition met
clusteroperator.config.openshift.io/machine-config condition met
clusteroperator.config.openshift.io/marketplace condition met
clusteroperator.config.openshift.io/monitoring condition met
clusteroperator.config.openshift.io/network condition met
clusteroperator.config.openshift.io/node-tuning condition met
clusteroperator.config.openshift.io/openshift-apiserver condition met
clusteroperator.config.openshift.io/openshift-controller-manager condition met
clusteroperator.config.openshift.io/openshift-samples condition met
clusteroperator.config.openshift.io/operator-lifecycle-manager condition met
clusteroperator.config.openshift.io/operator-lifecycle-manager-catalog condition met
clusteroperator.config.openshift.io/operator-lifecycle-manager-packageserver condition met
clusteroperator.config.openshift.io/service-ca condition met
clusteroperator.config.openshift.io/storage condition met
```

Delete any NNCP and make sure the `migration` field is set to `none`.


```
awezsonde@Awezs-Mac-Studio ~ % oc patch Network.operator.openshift.io cluster --type='merge' --patch '{"spec":{"migration":null}}' 
network.operator.openshift.io/cluster patched (no change)
```

### Start the prerequisites for migration

```
awezsonde@Awezs-Mac-Studio ~ % oc patch Network.operator.openshift.io cluster --type='merge' --patch '{ "spec": { "migration": { "networkType": "OVNKubernetes" } } }' 
network.operator.openshift.io/cluster patched

```

### Verify the process

```
awezsonde@Awezs-Mac-Studio ~ % oc get mcp   
NAME     CONFIG                                             UPDATED   UPDATING   DEGRADED   MACHINECOUNT   READYMACHINECOUNT   UPDATEDMACHINECOUNT   DEGRADEDMACHINECOUNT   AGE
master   rendered-master-111e65b19e5e673d02a596329266a022   False     True       False      3              0                   0                     0                      7d6h
worker   rendered-worker-ca4c7dc88351e223445c226a15ea52de   False     True       False      2              0                   0                     0                      7d6h


awezsonde@Awezs-Mac-Studio ~ % oc get nodes
NAME                                                STATUS                     ROLES                  AGE    VERSION
master-0.ovnmigration.lab.upshift.rdu2.redhat.com   Ready,SchedulingDisabled   control-plane,master   7d6h   v1.27.14+7852426
master-1.ovnmigration.lab.upshift.rdu2.redhat.com   Ready                      control-plane,master   7d6h   v1.27.14+7852426
master-2.ovnmigration.lab.upshift.rdu2.redhat.com   Ready                      control-plane,master   7d6h   v1.27.14+7852426
worker-0.ovnmigration.lab.upshift.rdu2.redhat.com   Ready                      worker                 7d6h   v1.27.14+7852426
worker-1.ovnmigration.lab.upshift.rdu2.redhat.com   Ready                      worker                 7d6h   v1.27.14+7852426



awezsonde@Awezs-Mac-Studio ~ % oc get pod -n openshift-machine-config-operator

NAME                                                                     READY   STATUS    RESTARTS   AGE
kube-rbac-proxy-crio-master-0.ovnmigration.lab.upshift.rdu2.redhat.com   1/1     Running   4          7d6h
kube-rbac-proxy-crio-master-1.ovnmigration.lab.upshift.rdu2.redhat.com   1/1     Running   4          7d6h
kube-rbac-proxy-crio-master-2.ovnmigration.lab.upshift.rdu2.redhat.com   1/1     Running   4          7d6h
kube-rbac-proxy-crio-worker-0.ovnmigration.lab.upshift.rdu2.redhat.com   1/1     Running   7          7d6h
kube-rbac-proxy-crio-worker-1.ovnmigration.lab.upshift.rdu2.redhat.com   1/1     Running   7          7d6h
machine-config-controller-56fc59fdcf-pv49c                               2/2     Running   0          17m
machine-config-daemon-7h489                                              2/2     Running   2          7d6h
machine-config-daemon-h98hl                                              2/2     Running   2          7d6h
machine-config-daemon-l7kx2                                              2/2     Running   2          7d6h
machine-config-daemon-nvnnv                                              2/2     Running   2          7d6h
machine-config-daemon-z6kfn                                              2/2     Running   2          7d6h
machine-config-operator-575bdfffc6-v5hfb                                 2/2     Running   0          13m
machine-config-server-db272                                              1/1     Running   1          7d6h
machine-config-server-h5vtq                                              1/1     Running   1          7d6h
machine-config-server-wz28q                                              1/1     Running   1          7d6h


```


## Start the migration

```
awezsonde@Awezs-Mac-Studio ~ % oc patch Network.config.openshift.io cluster --type='merge' --patch '{ "spec": { "networkType": "OVNKubernetes" } }' 
network.config.openshift.io/cluster patched


```

## Verify the multus daemonset has rolled out

```
awezsonde@Awezs-Mac-Studio ~ % oc -n openshift-multus rollout status daemonset/multus

daemon set "multus" successfully rolled out
```

Wait for all the operators to be available

```
NAME                                       VERSION   AVAILABLE   PROGRESSING   DEGRADED   SINCE   MESSAGE
authentication                             4.14.31   True        False         False      3s
baremetal                                  4.14.31   True        False         False	  7d6h
cloud-controller-manager                   4.14.31   True        False         False	  7d6h
cloud-credential                           4.14.31   True        False         False	  7d7h
cluster-autoscaler                         4.14.31   True        False         False	  7d6h
config-operator                            4.14.31   True        False         False	  7d6h
console                                    4.14.31   False       False         False	  44s     RouteHealthAvailable: console route is not admitted
control-plane-machine-set                  4.14.31   True        False         False	  7d6h
csi-snapshot-controller                    4.14.31   True        False         False	  7d6h
dns                                        4.14.31   True        True          True	  7d6h    DNS default is degraded
etcd                                       4.14.31   True        False         False	  7d6h
image-registry                             4.14.31   False	 True          True	  17m     Available: The deployment does not have available replicas...
ingress                                    4.14.31   True        False         False	  38m
insights                                   4.14.31   True        False         False	  7d6h
kube-apiserver                             4.14.31   True        False         False	  7d6h
kube-controller-manager                    4.14.31   True        False         False      7d6h
kube-scheduler                             4.14.31   True        False         False	  7d6h
kube-storage-version-migrator              4.14.31   True        False         False	  37m
machine-api                                4.14.31   True        False         False	  7d6h
machine-approver                           4.14.31   True        False         False	  7d6h
machine-config                             4.14.31   True        False         False	  7d6h
marketplace                                4.14.31   True        False         False	  7d6h
monitoring                                 4.14.31   False       True          True 	  2m27s   reconciling Prometheus Operator Admission Webhook Deployment failed: updat
ing Deployment object failed: waiting for DeploymentRollout of openshift-monitoring/prometheus-operator-admission-webhook: context deadline exceeded
network                                    4.14.31   True        True          False	  7d6h    DaemonSet "/openshift-network-diagnostics/network-check-target" is not ava
ilable (awaiting 2 nodes)
node-tuning                                4.14.31   True        False         False	  7d6h
openshift-apiserver                        4.14.31   True 	 False         False	  50s
openshift-controller-manager               4.14.31   True        False         False	  7d6h
openshift-samples                          4.14.31   True        False         False	  7d6h
operator-lifecycle-manager                 4.14.31   True        False         False	  7d6h
operator-lifecycle-manager-catalog         4.14.31   True        False         False	  7d6h
operator-lifecycle-manager-packageserver   4.14.31   True        False         False	  7d6h
service-ca                                 4.14.31   True        False         False	  7d6h
storage                                    4.14.31   True        False         False	  7d6h


```





The above operators seems to be in Degraded state for a long time.Starting the reboot script

```

[root@test ~]# cat reboot-post-migration.sh 
#!/bin/bash
readarray -t POD_NODES <<< "$(oc get pod -n openshift-machine-config-operator -o wide| grep daemon|awk '{print $1" "$7}')"

for i in "${POD_NODES[@]}"
do
  read -r POD NODE <<< "$i"
  until oc rsh -n openshift-machine-config-operator "$POD" chroot /rootfs shutdown -r +1
    do
      echo "cannot reboot node $NODE, retry" && sleep 3
    done
done






[root@test ~]# chmod +x reboot-post-migration.sh 
[root@test ~]# ./reboot-post-migration.sh 
Defaulted container "machine-config-daemon" out of: machine-config-daemon, kube-rbac-proxy
Reboot scheduled for Thu 2025-01-02 17:51:23 UTC, use 'shutdown -c' to cancel.
Defaulted container "machine-config-daemon" out of: machine-config-daemon, kube-rbac-proxy
Reboot scheduled for Thu 2025-01-02 17:51:53 UTC, use 'shutdown -c' to cancel.
Defaulted container "machine-config-daemon" out of: machine-config-daemon, kube-rbac-proxy
Reboot scheduled for Thu 2025-01-02 17:51:53 UTC, use 'shutdown -c' to cancel.
Defaulted container "machine-config-daemon" out of: machine-config-daemon, kube-rbac-proxy
Reboot scheduled for Thu 2025-01-02 17:51:54 UTC, use 'shutdown -c' to cancel.
Defaulted container "machine-config-daemon" out of: machine-config-daemon, kube-rbac-proxy
Reboot scheduled for Thu 2025-01-02 17:51:54 UTC, use 'shutdown -c' to cancel.
```

Its normal to lose the connectivity of the cluster for few minutes

After the reboot script the operators should be fine

```
NAME                                       VERSION   AVAILABLE   PROGRESSING   DEGRADED   SINCE   MESSAGE
authentication                             4.14.31   True        False         False      7m36s
baremetal                                  4.14.31   True        False         False	  7d7h
cloud-controller-manager                   4.14.31   True        False         False	  7d7h
cloud-credential                           4.14.31   True        False         False	  7d7h
cluster-autoscaler                         4.14.31   True        False         False	  7d7h
config-operator                            4.14.31   True        False         False	  7d7h
console                                    4.14.31   True        False         False	  30s
control-plane-machine-set                  4.14.31   True        False         False	  7d7h
csi-snapshot-controller                    4.14.31   True        False         False	  7d7h
dns                                        4.14.31   True        False         False	  3m29s
etcd                                       4.14.31   True        False         False	  7d7h
image-registry                             4.14.31   True 	 False         False      57s
ingress                                    4.14.31   True        False         False	  2m54s
insights                                   4.14.31   True        False         False	  7d7h
kube-apiserver                             4.14.31   True        False         False	  7d7h
kube-controller-manager                    4.14.31   True        False         False	  7d7h
kube-scheduler                             4.14.31   True        False         False	  7d7h
kube-storage-version-migrator              4.14.31   True        False         False	  53m
machine-api                                4.14.31   True        False         False	  7d7h
machine-approver                           4.14.31   True        False         False	  7d7h
machine-config                             4.14.31   True        False         False	  7d7h
marketplace                                4.14.31   True        False         False	  7d7h
monitoring                                 4.14.31   True        False         False	  2m59s
network                                    4.14.31   True        False         False	  7d7h
node-tuning                                4.14.31   True        False         False	  7d7h
openshift-apiserver                        4.14.31   True        False         False	  7m53s
openshift-controller-manager               4.14.31   True        False         False	  7d7h
openshift-samples                          4.14.31   True        False         False	  7d7h
operator-lifecycle-manager                 4.14.31   True        False         False	  7d7h
operator-lifecycle-manager-catalog         4.14.31   True        False         False	  7d7h
operator-lifecycle-manager-packageserver   4.14.31   True        False         False	  3m29s
service-ca                                 4.14.31   True        False         False	  7d7h
storage                                    4.14.31   True        False         False	  7d7h

```


## Verify the migration

```
[root@test ~]# oc get network.config/cluster -o jsonpath='{.status.networkType}{"\n"}'
OVNKubernetes



[root@test ~]# oc get nodes
NAME                                                STATUS   ROLES                  AGE    VERSION
master-0.ovnmigration.lab.upshift.rdu2.redhat.com   Ready    control-plane,master   7d7h   v1.27.14+7852426
master-1.ovnmigration.lab.upshift.rdu2.redhat.com   Ready    control-plane,master   7d7h   v1.27.14+7852426
master-2.ovnmigration.lab.upshift.rdu2.redhat.com   Ready    control-plane,master   7d7h   v1.27.14+7852426
worker-0.ovnmigration.lab.upshift.rdu2.redhat.com   Ready    worker                 7d6h   v1.27.14+7852426
worker-1.ovnmigration.lab.upshift.rdu2.redhat.com   Ready    worker                 7d6h   v1.27.14+7852426
```

## Finish the migration

```
[root@test ~]# oc patch Network.operator.openshift.io cluster --type='merge' --patch '{ "spec": { "migration": null } }' 
network.operator.openshift.io/cluster patched


[root@test ~]# oc patch Network.operator.openshift.io cluster --type='merge' --patch '{ "spec": { "defaultNetwork": { "openshiftSDNConfig": null } } }' 
network.operator.openshift.io/cluster patched (no change)


[root@test ~]# oc delete namespace openshift-sdn
namespace "openshift-sdn" deleted


```

# Verify the Network Policy post migration

List the network policy and the pods from `projecta` and `projectb` as they may have new IPs

```
[root@test ~]# oc get networkpolicy -A
NAMESPACE   NAME                    POD-SELECTOR   AGE
projectb    deny-projecta-traffic   <none>         92m



[root@test ~]# oc get pods -o wide -n projecta
NAME                                                 READY   STATUS    RESTARTS   AGE   IP           NODE                                                NOMINATED NODE   READINESS GATES
ovn-migration-network-policy-test-5d6d5d9fff-jqfww   1/1     Running   3          59m   10.131.0.6   worker-0.ovnmigration.lab.upshift.rdu2.redhat.com   <none>           <none>



[root@test ~]# oc get pods -o wide -n projectb
NAME                                                 READY   STATUS    RESTARTS   AGE   IP            NODE                                                NOMINATED NODE   READINESS GATES
ovn-migration-network-policy-test-55c4854885-28ffw   1/1     Running   3          59m   10.131.0.15   worker-0.ovnmigration.lab.upshift.rdu2.redhat.com   <none>           <none>


```

## Ping test 

```
[root@test ~]# oc project projecta
Now using project "projecta" on server "https://api.ovnmigration.lab.upshift.rdu2.redhat.com:6443".



[root@test ~]# oc rsh ovn-migration-network-policy-test-5d6d5d9fff-jqfww
sh-5.1$ ping 10.131.0.15
PING 10.131.0.15 (10.131.0.15) 56(84) bytes of data.
^C
--- 10.131.0.15 ping statistics ---
5 packets transmitted, 0 received, 100% packet loss, time 4121ms



````
Network Policies work well


## Another ping test after removing Network Policy

```
[root@test ~]# oc delete networkpolicy deny-projecta-traffic  -n projectb
networkpolicy.networking.k8s.io "deny-projecta-traffic" deleted




[root@test ~]# oc rsh ovn-migration-network-policy-test-5d6d5d9fff-jqfww
sh-5.1$ ping 10.131.0.15
PING 10.131.0.15 (10.131.0.15) 56(84) bytes of data.
64 bytes from 10.131.0.15: icmp_seq=1 ttl=64 time=0.991 ms
64 bytes from 10.131.0.15: icmp_seq=2 ttl=64 time=0.638 ms
64 bytes from 10.131.0.15: icmp_seq=3 ttl=64 time=0.080 ms
^C
--- 10.131.0.15 ping statistics ---
3 packets transmitted, 3 received, 0% packet loss, time 2057ms
rtt min/avg/max/mdev = 0.080/0.569/0.991/0.375 ms
```


So to conclude nothing substantial needs to be changed for migration atleast with network policies.


