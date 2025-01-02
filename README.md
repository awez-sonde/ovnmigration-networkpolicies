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

```

Delete any NNCP and make sure the `migration` field is set to `none`.



```

```
