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

## Create two projects for facilitation network policies

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



## Create application in `projecta` and `projectb` to test the connectivity between them



```
awezsonde@Awezs-Mac-Studio ~ % oc new-app rails-postgresql-example -n projecta --name rails-postgresql-projecta
--> Deploying template "projecta/rails-postgresql-example" to project projecta

     Rails + PostgreSQL (Ephemeral)
     ---------
     An example Rails application with a PostgreSQL database. For more information about using this template, including OpenShift considerations, see https://github.com/sclorg/rails-ex/blob/master/README.md.
     
     WARNING: Any data stored will be lost upon pod destruction. Only use this template for testing.

     The following service(s) have been created in your project: rails-postgresql-example, postgresql.
     
     For more information about using this template, including OpenShift considerations, see https://github.com/sclorg/rails-ex/blob/master/README.md.

     * With parameters:
        * Name=rails-postgresql-example
        * Namespace=openshift
        * Memory Limit=512Mi
        * Memory Limit (PostgreSQL)=512Mi
        * Git Repository URL=https://github.com/sclorg/rails-ex.git
        * Git Reference=
        * Context Directory=
        * Application Hostname=
        * GitHub Webhook Secret=0RXOErDCKTQPKw7fEpSECQ0gTKKin0jsLUqWSIaO # generated
        * Secret Key=2rci16ldj3g22ullvym0pc7csurjt5u2ghv3www0q3ido2rsfop4pfafh0152ug18wk0l426h2mbgn786clq32pvvfpecw3xp0nbchvp8b3lvslpbh1djj5ej332icd # generated
        * Application Username=openshift
        * Application Password=secret
        * Rails Environment=production
        * Database Service Name=postgresql
        * Database Username=userXCY # generated
        * Database Password=wnb1Emrd # generated
        * Database Name=root
        * Maximum Database Connections=100
        * Shared Buffer Amount=12MB
        * Custom RubyGems Mirror URL=

--> Creating resources ...
    secret "rails-postgresql-example" created
    service "rails-postgresql-example" created
    route.route.openshift.io "rails-postgresql-example" created
    imagestream.image.openshift.io "rails-postgresql-example" created
    buildconfig.build.openshift.io "rails-postgresql-example" created
Warning: apps.openshift.io/v1 DeploymentConfig is deprecated in v4.14+, unavailable in v4.10000+
    deploymentconfig.apps.openshift.io "rails-postgresql-example" created
    service "postgresql" created
    deploymentconfig.apps.openshift.io "postgresql" created
--> Success
    Access your application via route 'rails-postgresql-example-projecta.apps.ovnmigration.lab.upshift.rdu2.redhat.com' 
    Build scheduled, use 'oc logs -f buildconfig/rails-postgresql-example' to track its progress.
    Run 'oc status' to view your app.
```



```
awezsonde@Awezs-Mac-Studio ~ % oc new-app rails-postgresql-example -n projectb --name rails-postgresql-projectb
--> Deploying template "projectb/rails-postgresql-example" to project projectb

     Rails + PostgreSQL (Ephemeral)
     ---------
     An example Rails application with a PostgreSQL database. For more information about using this template, including OpenShift considerations, see https://github.com/sclorg/rails-ex/blob/master/README.md.
     
     WARNING: Any data stored will be lost upon pod destruction. Only use this template for testing.

     The following service(s) have been created in your project: rails-postgresql-example, postgresql.
     
     For more information about using this template, including OpenShift considerations, see https://github.com/sclorg/rails-ex/blob/master/README.md.

     * With parameters:
        * Name=rails-postgresql-example
        * Namespace=openshift
        * Memory Limit=512Mi
        * Memory Limit (PostgreSQL)=512Mi
        * Git Repository URL=https://github.com/sclorg/rails-ex.git
        * Git Reference=
        * Context Directory=
        * Application Hostname=
        * GitHub Webhook Secret=X728NM5RVSVK56PViexP0nKayMQmWJ8iBTNDrC8U # generated
        * Secret Key=n8vyqutsi74wp76olkmsrtvumuwqfpuse66wkstlx0hf4oonhtyqhrycwlch7tp7wia5u1fakwt3jxyg87ykrd7fd8v7tm8afnogcv0opi5cknvhk3eh0yf01qvg7rb # generated
        * Application Username=openshift
        * Application Password=secret
        * Rails Environment=production
        * Database Service Name=postgresql
        * Database Username=userQRF # generated
        * Database Password=hcoFeG2u # generated
        * Database Name=root
        * Maximum Database Connections=100
        * Shared Buffer Amount=12MB
        * Custom RubyGems Mirror URL=

--> Creating resources ...
    secret "rails-postgresql-example" created
    service "rails-postgresql-example" created
    route.route.openshift.io "rails-postgresql-example" created
    imagestream.image.openshift.io "rails-postgresql-example" created
    buildconfig.build.openshift.io "rails-postgresql-example" created
Warning: apps.openshift.io/v1 DeploymentConfig is deprecated in v4.14+, unavailable in v4.10000+
    deploymentconfig.apps.openshift.io "rails-postgresql-example" created
    service "postgresql" created
    deploymentconfig.apps.openshift.io "postgresql" created
--> Success
    Access your application via route 'rails-postgresql-example-projectb.apps.ovnmigration.lab.upshift.rdu2.redhat.com' 
    Build scheduled, use 'oc logs -f buildconfig/rails-postgresql-example' to track its progress.
    Run 'oc status' to view your app.
```



