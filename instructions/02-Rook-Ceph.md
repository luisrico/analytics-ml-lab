# Rook Deployment

Make sure all the following tasks are executed where the yml files are:
```
cd analytics-ml-lab/rook/
```

## Setup security contexts and deploy rook-ceph operator

First, we have to create a Security Context Contraints (SCC) called "rook-ceph", 
to let Rook operator have extra capabilities with several Service accounts. That is done 
using a common.yaml to provide all things required in OpenShift for Rook operator.
Then, we can create operator Rook.io. It will create: 
* one rook-ceph-operator pod in the newly created rook-ceph project/namespace
* plus 2 pods per Openshift worker node: 1 rook-ceph-agent + 1 rook-discover.

In total there will be 7 pods running, because we have 3 worker nodes.
 
```
oc create -f common.yaml
oc create -f operator-openshift.yaml
watch oc get pods -n rook-ceph -o wide
```

## Create Ceph cluster, with 3 monitors and with OSDs on every worker node

Now, let's define the Ceph cluster we want Rook operator to deploy using cluster.yml. Explore the
cluster.yml to see the definition of Ceph Cluster. As per limitations of lab environment, Ceph pods 
will use HostPath resources in worker nodes at /var/lib/rook, instead of dedicated disks. For production
environments, dedicated disks will be required for Monitor rocksdb databases and for Object Storage Daemon 
devices OSDs. It will create:
* All pods: 3 monitor, 1 manager, 4 osds in the rook-ceph namespace. 

Watch until all components have been created, producing a Ceph Cluster completely deployed by Rook.
>_Note: the `rook-ceph-osd-prepare` pods are READY: 0/2, which is OK._

```
cat cluster.yaml
oc create -f cluster.yaml
watch oc get pods -n rook-ceph -o wide
```

## Create toolbox deployment for Ceph CLI interaction

To get access to common CLI Ceph tools, it's required to deploy a "toolbox" pod.
Once created, rsh on it and execute some common ceph commands to see Ceph cluster health.

```
oc create -f toolbox.yaml
oc get pods -n rook-ceph | grep tool
oc rsh -n rook-ceph rook-ceph-tools-<pod-uuid-your-env>

  ceph health
  ceph status
  ceph osd status
  ceph osd tree

sh-4.2# exit ## Remember to exit from your toolbox pod
```

## Create object storage service

For Ceph Object storage, it's needed to have Rados Gateway (RGW) components, able to translate access 
through http/https using S3 API to Ceph RADOS. With object-openshift.yaml create your RGW pod. 
* 1 RGW pod is deployed.

Watch and wait until it is in Running state.

```
oc create -f object-openshift.yaml
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
              egrep -v 'namespace|selfLink|creation|resourceVersion' | \
              sed 's/rook-ceph-object-user-my-store-demo/s3-user-demo/g' | \
              oc create -f -
oc get secret s3-user-demo -o yaml
```

## Install 's3cmd' & 'nmap' tools to test and play with the Object Storage gateway access

An easy way to interact with an S3 Object Storage interface is to use the `s3cmd` tool. There are plenty of tools available out there, that you can explore on your own another time. The same thing with `nmap`, that's just one convenient way to check which ports a remote service is listening on.

First of all, we expose the RGW service to be able to access it from our client-vm.
```
oc get svc -n rook-ceph | grep rgw
```
_Take note: your RGW service should be namned: `rook-ceph-rgw-my-store`_
Then expose your RGW service.
```
oc -n rook-ceph expose svc/rook-ceph-rgw-my-store
```
_Verify and take note of your exposed route, as it will be used when configuring `s3cmd`._
_It will look similar to this: `rook-ceph-rgw-my-store-rook-ceph.apps.cluster-vienna-a965.vienna-a965.openshiftworkshop.com`_
```
oc get routes -n rook-ceph
```

Become superuser to install `nmap` and `s3cmd` tools.
```
sudo su -
yum -y install nmap
yum -y install python-pip python-wheel
pip install s3cmd
exit
```
_from now on continue as the normal user again_

Check with `nmap` that the RGW service is listening and are available on the default ports '80' and '443'.
_Use the route that you exposed in the previous step._
```
nmap -v rook-ceph-rgw-my-store-rook-ceph.apps.cluster-vienna-a965.vienna-a965.openshiftworkshop.com
```
_You should see your RGW service responding on ports 80 and 443._

Get the S3 'AccessKey' and 'SecretKey' that is randomly created for your RGW service.
```
oc get secret -n rook-ceph | grep user
```
_You should see an output similar to: `rook-ceph-object-user-my-store-demo`_

```
oc -n rook-ceph get secret rook-ceph-object-user-my-store-demo -o yaml | grep AccessKey | awk '{print $2}' | base64 --decode
```
`3LYD5EG2D55W4ULR3UOL`
* _This AccessKey is used when you configure your `s3cmd` tool. Your key will look different._ 

```
oc -n rook-ceph get secret rook-ceph-object-user-my-store-demo -o yaml | grep SecretKey | awk '{print $2}' | base64 --decode
```
`dqPmcdYOOhhR5NF0XAyMgLsAGpadL3iEobJJ7iyk`
* _This SecretKey is used when you configure your `s3cmd` tool. Your key will look different._

Configure the `s3cmd` tool to use your RGW Object Storage service with your AccessKey and SecretKey
```
s3cmd --configure
```
_Enter your, `AccessKey` from above_

_Enter your, `SecretKey` from above_

_For Default Region, Just hit `ENTER`_

_S3 Endpoint, `Enter your RGW service route from above`_

_DNS-style bucket+hostname, `Enter your RGW service route from above`_

_Encryption password, Just hit `ENTER`_

_Path to GPG program [/usr/bin/gpg], Just hit `ENTER`_

_Use HTTPS protocol, `No`_

_HTTP Proxy server name, Just hit `ENTER`_



###### NOTE: The `s3cmd` configuration file will be stored in your home-dir with the hidden name `.s3cfg`

Create your first S3 Object Storage bucket
```
s3cmd mb s3://mybucket
```

```
s3cmd ls ## list available buckets
```

Upload your `/etc/hosts` file as an object to the Ceph S3 Object Store

```
s3cmd put /etc/hosts s3://mybucket
```

_Copies /etc/hosts as the object hosts to the bucket_

```
s3cmd ls s3://mybucket
```

_lists the objects available in the bucket_

```
s3cmd get s3://mybucket/hosts getfile
```

_retrieves the object hosts and stores it as a file named getfile_

Upload the contents of a whole directory, note that the files are automatically chopped into smaller objects.

```
mkdir put_test
cd put_test
for i in 0 1 2 3 4 5 6 7 8 9 10; do sudo tar cf  etc$i.tar /etc ; done
du -sh *
```
```
cd ..
s3cmd put -r put_test/ s3://mybucket
```

_recursively puts all files from the put_test directory as objects in the Ceph S3 Object Storage bucket_

```
s3cmd ls s3://mybucket
```

## Upgrade seamlessly Ceph Cluster with Rook operator 

Take benefit of Rook.io operator management lifecycle and request Rook to execute an automatic
rolling upgrade to your Ceph Cluster. For that, you have to edit the CRD of your CephCluster 
to change the tag number of the version. When editing Ceph cluster replace in the vi editor 
`image: ceph/ceph:v14.2.1-20190430` with **`image: ceph/ceph:v14.2.2-20190722`**
Check the current Ceph version in the toolbox, before and after the upgrade. 
After changing version, watch the process of how Rook upgrades pods one by one starting with `rook-ceph-mon` then `rook-ceph-mgr` , `rook-ceph-osd` and finally `rook-ceph-rgw`.


```
oc rsh -n rook-ceph rook-ceph-tools-<pod-uuid-your-env>
  ceph version
  exit
oc edit CephCluster -n rook-ceph

watch oc get pods -n rook-ceph

oc rsh -n rook-ceph rook-ceph-tools-<pod-uuid-your-env>
  ceph version
  exit
```

## Use Ceph upstream dashboard to monitor health and performance of your Ceph Cluster

Ceph cluster deployed with upstream Rook operator provides a Ceph dashboard to monitor 
alerts and real time performance metrics.
To use it, we have to configure a RGW user to give access to dashboard to our object 
storage part, expose the route of the dashboard and find out the password of **`admin`** user

```

oc rsh -n rook-ceph rook-ceph-tools-<pod-uuid-your-env>
  radosgw-admin user create --uid=dashboard --display-name=dashboard --access-key 123456 --secret-key 123456 --system
  ceph dashboard set-rgw-api-user-id dashboard
  ceph dashboard set-rgw-api-ssl-verify False
  ceph dashboard set-rgw-api-scheme http
  ceph dashboard set-rgw-api-access-key 123456
  ceph dashboard set-rgw-api-secret-key 123456
  exit
  
oc get secret -n rook-ceph rook-ceph-dashboard-password -o yaml | grep "password:" | awk '{print $2}' | base64 --decode
`BBs2F48IDf`# use this password to login to the dashboard (your password will look different)

oc create route passthrough rook-ceph-dashboard -n rook-ceph --service=rook-ceph-mgr-dashboard --port https-dashboard
oc get route -n rook-ceph

```
Use this route to login to the Ceph dashboard from a browser. i.e. https://rook-ceph-dashboard-rook-ceph.apps.cluster-7bde.7bde.sandbox140.opentlc.com (remember to replace with your route)

##### Now continue to the next lab
Instructions 03-Jupyter-Notebook https://github.com/luisrico/analytics-ml-lab/blob/master/instructions/03-Jupyter-Notebook.md
