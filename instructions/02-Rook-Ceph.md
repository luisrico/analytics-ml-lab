# Rook Deployment

Make sure all the following tasks are executed where the yml files are:
```
cd analytics-ml-lab/rook/
```

## Setup security contexts and deploy rook-ceph operator

First, we have to create a Security Context Contraints (SCC) called "rook-ceph", 
to let Rook operator have extra capabilities with several Service accounts. That is done 
using a common.yaml to provide all things required in OpenShift for Rook operator.
Then, we can create operator Rook.io. It will create one rook-ceph-operator pod in the 
newly created rook-ceph project/namespace, plus 2 pods per Openshift worker node: 
1 rook-ceph-agent + 1 rook-discoverer.
 
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

sh-4.2# exit ## Remember to exit from your toolbox pod
```

## Create object storage service

For Ceph Object storage, it's needed to have Rados Gateway (RGW) components, able to translate access 
through http/https using S3 API to Ceph RADOS. With object-openshift.yaml, 1 RGW pod is deployed. Watch and wait
until it is in Running state.

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
<style
  type="text/css">
p {color:blue;}
</style>
<p>rook-ceph-rgw-my-store    ClusterIP   172.30.51.4      <none>        8000/TCP   23m
</p>

oc -n rook-ceph expose svc/rook-ceph-rgw-my-store
route.route.openshift.io/rook-ceph-rgw-my-store exposed

oc get routes -n rook-ceph
NAME                     HOST/PORT                                                                                     PATH   SERVICES                 PORT   TERMINATION   WILDCARD
rook-ceph-rgw-my-store   rook-ceph-rgw-my-store-rook-ceph.apps.cluster-vienna-a965.vienna-a965.openshiftworkshop.com          rook-ceph-rgw-my-store   http                 None
```

Install 'nmap' and 's3cmd' tools.
```
sudo su -
yum -y install nmap
yum -y install python-pip python-wheel
pip install s3cmd
exit ## from now on continue as the normal user again
```

Check with 'nmap' that the RGW service is listening and are available on the default ports '80' and '443'.
Use the route that you exposed in the previous step.
```
nmap -v  <Use the route you got in the previous step oc get routes>
...
PORT    STATE SERVICE
80/tcp  open  http
443/tcp open  https
```

Get the S3 'AccessKey' and 'SecretKey' that is randomly created for your RGW service.
```
oc get secret -n rook-ceph | grep user
rook-ceph-object-user-my-store-demo   kubernetes.io/rook                    2      77m

oc -n rook-ceph get secret rook-ceph-object-user-my-store-demo -o yaml | grep AccessKey | awk '{print $2}' | base64 --decode
3LYD5EG2D55W4ULR3UOL
## This AccessKey is used when you configure your 's3cmd' tool. Your key will look different.

oc -n rook-ceph get secret rook-ceph-object-user-my-store-demo -o yaml | grep SecretKey | awk '{print $2}' | base64 --decode
dqPmcdYOOhhR5NF0XAyMgLsAGpadL3iEobJJ7iyk
## This SecretKey is used when you configure your 's3cmd' tool. Your key will look different.
```

Configure the 's3cmd' tool to use your RGW Object Storage service with your AccessKey and SecretKey
```
s3cmd --configure
...
Access Key []: ## Enter your key from above
Secret Key []: ## Enter your key from above
Default Region [US]: ## Just hit ENTER
...
S3 Endpoint []: ## Enter your RGW service route: rook-ceph-rgw-my-store-rook-ceph.apps.cluster-vienna-a965.vienna-a965.openshiftworkshop.com
...
DNS-style bucket+hostname:port template for accessing a bucket []: rook-ceph-rgw-my-store-rook-ceph.apps.cluster-vienna-a965.vienna-a965.openshiftworkshop.com
...
Encryption password: ## Just hit ENTER
Path to GPG program [/usr/bin/gpg]: ## Just hit ENTER
...
Use HTTPS protocol [Yes]: No
...
HTTP Proxy server name: ## Just hit ENTER


## NOTE: The 's3cmd' configuration file will be stored in your home-dir with the hidden name .s3cfg
```

Create your first S3 Object Storage bucket
```
s3cmd mb s3://mybucket
Bucket 's3://mybucket/' created

s3cmd ls ## list available buckets
2019-08-15 12:35  s3://mybucket
```

Upload your /etc/hosts file as an object to the Ceph S3 Object Store

```
s3cmd put /etc/hosts s3://mybucket
upload: '/etc/hosts' -> 's3://mybucket/hosts'  [1 of 1]
 159 of 159   100% in    0s     2.55 kB/s  done
## Copies /etc/hosts as the object hosts to the bucket

s3cmd ls s3://mybucket
2019-08-15 12:39       159   s3://mybucket/hosts
## lists the objects available in the bucket

s3cmd get s3://mybucket/hosts getfile
download: 's3://mybucket/hosts' -> 'getfile'  [1 of 1]
 159 of 159   100% in    0s    12.15 kB/s  done
## retrieves the object hosts and stores it as a file named getfile
```

Upload the contents of a whole directory, note that the files are automatically chopped into smaller objects.
```
mkdir put_test
cd put_test
for i in 0 1 2 3 4 5 6 7 8 9 10; do  tar cf  etc$i.tar /etc ; done
du -sh *
18M	etc0.tar
18M	etc10.tar
18M	etc1.tar
18M	etc2.tar
18M	etc3.tar
18M	etc4.tar
18M	etc5.tar
18M	etc6.tar
18M	etc7.tar
18M	etc8.tar
18M	etc9.tar

cd ..
s3cmd put -r put_test/ s3://mybucket
## recursively puts all files from the put_test directory as objects in the Ceph S3 Object Storage bucket

upload: 'put_test/etc0.tar' -> 's3://mybucket/etc0.tar'  [part 1 of 2, 15MB] [1 of 11]
 15728640 of 15728640   100% in    0s    41.29 MB/s  done
upload: 'put_test/etc0.tar' -> 's3://mybucket/etc0.tar'  [part 2 of 2, 2MB] [1 of 11]
 2457600 of 2457600   100% in    0s    30.57 MB/s  done
upload: 'put_test/etc1.tar' -> 's3://mybucket/etc1.tar'  [part 1 of 2, 15MB] [2 of 11]
 15728640 of 15728640   100% in    0s    41.50 MB/s  done
upload: 'put_test/etc1.tar' -> 's3://mybucket/etc1.tar'  [part 2 of 2, 2MB] [2 of 11]
 2457600 of 2457600   100% in    0s    26.49 MB/s  done
upload: 'put_test/etc10.tar' -> 's3://mybucket/etc10.tar'  [part 1 of 2, 15MB] [3 of 11]
 15728640 of 15728640   100% in    0s    38.88 MB/s  done
upload: 'put_test/etc10.tar' -> 's3://mybucket/etc10.tar'  [part 2 of 2, 2MB] [3 of 11]
 2457600 of 2457600   100% in    0s    26.21 MB/s  done
upload: 'put_test/etc2.tar' -> 's3://mybucket/etc2.tar'  [part 1 of 2, 15MB] [4 of 11]
 15728640 of 15728640   100% in    0s    30.37 MB/s  done
upload: 'put_test/etc2.tar' -> 's3://mybucket/etc2.tar'  [part 2 of 2, 2MB] [4 of 11]
 2457600 of 2457600   100% in    0s    10.57 MB/s  done
upload: 'put_test/etc3.tar' -> 's3://mybucket/etc3.tar'  [part 1 of 2, 15MB] [5 of 11]
 15728640 of 15728640   100% in    0s    28.75 MB/s  done
upload: 'put_test/etc3.tar' -> 's3://mybucket/etc3.tar'  [part 2 of 2, 2MB] [5 of 11]
 2457600 of 2457600   100% in    0s    14.79 MB/s  done
upload: 'put_test/etc4.tar' -> 's3://mybucket/etc4.tar'  [part 1 of 2, 15MB] [6 of 11]
 15728640 of 15728640   100% in    0s    28.24 MB/s  done
upload: 'put_test/etc4.tar' -> 's3://mybucket/etc4.tar'  [part 2 of 2, 2MB] [6 of 11]
 2457600 of 2457600   100% in    0s    14.78 MB/s  done
upload: 'put_test/etc5.tar' -> 's3://mybucket/etc5.tar'  [part 1 of 2, 15MB] [7 of 11]
 15728640 of 15728640   100% in    0s    29.65 MB/s  done
upload: 'put_test/etc5.tar' -> 's3://mybucket/etc5.tar'  [part 2 of 2, 2MB] [7 of 11]
 2457600 of 2457600   100% in    0s    15.51 MB/s  done
upload: 'put_test/etc6.tar' -> 's3://mybucket/etc6.tar'  [part 1 of 2, 15MB] [8 of 11]
 15728640 of 15728640   100% in    0s    25.04 MB/s  done
upload: 'put_test/etc6.tar' -> 's3://mybucket/etc6.tar'  [part 2 of 2, 2MB] [8 of 11]
 2457600 of 2457600   100% in    0s    11.26 MB/s  done
upload: 'put_test/etc7.tar' -> 's3://mybucket/etc7.tar'  [part 1 of 2, 15MB] [9 of 11]
 15728640 of 15728640   100% in    0s    28.27 MB/s  done
upload: 'put_test/etc7.tar' -> 's3://mybucket/etc7.tar'  [part 2 of 2, 2MB] [9 of 11]
 2457600 of 2457600   100% in    0s    14.67 MB/s  done
upload: 'put_test/etc8.tar' -> 's3://mybucket/etc8.tar'  [part 1 of 2, 15MB] [10 of 11]
 15728640 of 15728640   100% in    0s    30.77 MB/s  done
upload: 'put_test/etc8.tar' -> 's3://mybucket/etc8.tar'  [part 2 of 2, 2MB] [10 of 11]
 2457600 of 2457600   100% in    0s    16.95 MB/s  done
upload: 'put_test/etc9.tar' -> 's3://mybucket/etc9.tar'  [part 1 of 2, 15MB] [11 of 11]
 15728640 of 15728640   100% in    0s    25.07 MB/s  done
upload: 'put_test/etc9.tar' -> 's3://mybucket/etc9.tar'  [part 2 of 2, 2MB] [11 of 11]
 2457600 of 2457600   100% in    0s    13.81 MB/s  done
 
s3cmd ls s3://mybucket
2019-08-15 12:45  18186240   s3://mybucket/etc0.tar
2019-08-15 12:45  18186240   s3://mybucket/etc1.tar
2019-08-15 12:45  18186240   s3://mybucket/etc10.tar
2019-08-15 12:45  18186240   s3://mybucket/etc2.tar
2019-08-15 12:45  18186240   s3://mybucket/etc3.tar
2019-08-15 12:45  18186240   s3://mybucket/etc4.tar
2019-08-15 12:45  18186240   s3://mybucket/etc5.tar
2019-08-15 12:45  18186240   s3://mybucket/etc6.tar
2019-08-15 12:45  18186240   s3://mybucket/etc7.tar
2019-08-15 12:45  18186240   s3://mybucket/etc8.tar
2019-08-15 12:45  18186240   s3://mybucket/etc9.tar
2019-08-15 12:39       159   s3://mybucket/hosts
```

## Upgrade seamlessly Ceph Cluster with Rook operator 

Take benefit of Rook.io operator management lifecycle and request Rook to execute an automatic
rolling upgrade to your Ceph Cluster. For that, you have to edit the CRD of your CephCluster 
to change the tag number of the version. When editing Ceph cluster replace in the vi editor 
image: ceph/ceph:v14.2.1-20190430 with image: ceph/ceph:v14.2.2-20190722
Check the current Ceph version in the toolbox, before and after the upgrade. 
After changing version, watch the process of how Rook upgrades pods one by one starting with Monitors.


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
storage part, expose the route of the dashboard and find out the password of admin user

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

oc create route passthrough rook-ceph-dashboard --service=rook-ceph-mgr-dashboard --port https-dashboard
oc get route

```
