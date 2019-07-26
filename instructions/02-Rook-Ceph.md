# Rook Deployment

Make sure all the following tasks are executed where the yml files are:
```
cd analytics-ml-lab/rook/
```

## Setup security contexts and deploy rook-ceph operator

First, we have to create a Security Context Contraints (SCC) called "rook-ceph", 
to let Rook operator have extra capabilities with several Service accounts.
Then, we can create operator Rook.io. This will create rook-ceph-system namespace/project 
to host rook operator resources. It will create one rook-ceph-operator pod in the 
newly created rook-ceph-system project/namespace, plus 2 pods per Openshift worker node: 
1 rook-ceph-agent + 1 rook-discoverer.
 
```
oc create -f scc.yaml
oc create -f operator.yaml
watch oc get pods -n rook-ceph-system -o wide
```

## Create Ceph cluster, with 3 monitors and with OSDs on every worker node

Now, let's define the Ceph cluster we want Rook operator to deploy using cluster.yml. Explore the
cluster.yml to see the definition of Ceph Cluster. As per limitations of lab environment, Ceph pods 
will use HostPath resources in worker nodes at /var/lib/rook, instead of dedicated disks. For production
environments, dedicated disks will be required for Monitor rocksdb databases and for Object Storage Daemon 
devices OSDs. All pods: 3 monitor, 1 manager, 4 osds are create in new rook-ceph namespace. Watch until all 
components have been created, producing a Ceph Cluster completely deployed by Rook.

```
cat cluster.yml
oc create -f cluster.yaml
watch oc get pods -n rook-ceph -o wide
```

## Create toolbox deployment for Ceph CLI interaction

To get access to common CLI Ceph tools, it's required to deploy a "toolbox" pod.
Once created, rsh on it and execute some common ceph commands to see Ceph cluster health.

```
oc create -f toolbox.yaml
oc get pods -n rook-ceph
oc rsh -n rook-ceph rook-ceph-tools-<pod-uuid-your-env>

ceph health
ceph status
ceph osd status
ceph osd tree
```

## Create object storage service

For Ceph Object storage, it's needed to have Rados Gateway (RGW) components, able to translate access 
through http/https using S3 API to Ceph RADOS. With object.yaml, 2 RGW pods are deployed. Watch and wait
until they are in Running state.

```
oc create -f object.yaml
watch oc get pods -n rook-ceph -o wide
```

## Create demo user
To access our Ceph object storage through S3 is required a valid user with credentials. With object-user.yaml, 
a user called demo is created to be used later for accessing object storage when analytics and ML workloads.

```
oc create -f object-user.yaml
```

## Copy S3 demo user credentials into my-analytics namespace

Let's copy the user demo credentials in a secret in the my-analytics namespace. You can see the AccessKey 
and SecretKey required for S3 access.

```
oc project my-analytics
oc get secret -n rook-ceph rook-ceph-object-user-my-store-demo -o yaml | \
              egrep -v 'uid|namespace|selfLink|creation|resourceVersion' | \
              sed 's/rook-ceph-object-user-my-store-demo/s3-user-demo/g' | \
              oc create -f -
oc get secret s3-user-demo -o yaml
```
