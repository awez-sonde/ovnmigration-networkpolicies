# Network policies post OVN migration in OCP 4
Effects on network policies post OVN migration

## CLuster Details


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
