## Remove any limits on the project / namespace my-analytics

Some cluster admin default restrictions could make our jupyter pod to have memory allocation
problems. If there is any limit, let's remove it before deploying.

```
$ oc get limits -o yaml
```
_```
apiVersion: v1
items:
- apiVersion: v1
  kind: LimitRange
  metadata:
    creationTimestamp: "2019-10-18T07:59:02Z"
    name: my-analytics-core-resource-limits
    namespace: my-analytics
    resourceVersion: "175894"
    selfLink: /api/v1/namespaces/my-analytics/limitranges/my-analytics-core-resource-limits
    uid: 2618c27a-f17d-11e9-86b0-02862496ab38
  spec:
    limits:
    - default:
        cpu: 500m
        memory: 1536Mi
      defaultRequest:
        cpu: 50m
        memory: 256Mi
      max:
        cpu: "2"
        memory: 6Gi
      type: Container
    - max:
        cpu: "2"
        memory: 12Gi
      type: Pod
kind: List
metadata:
  resourceVersion: ""
  selfLink: ""
```_

```
$ oc delete limits my-analytics-core-resource-limits
```
_`limitrange "my-analytics-core-resource-limits" deleted`_




## Create and Deploy Jupyter Notebook

Now, let's create a new application that will hold the Jupyter Notebook we are going 
to use and embedded Apache Spark and TensorFlow:

```
oc new-app -i jupyter-notebook:latest \
           -e JUPYTER_NOTEBOOK_PASSWORD=developer \
           -e S3_ENDPOINT="http://rook-ceph-rgw-my-store.rook-ceph.svc.cluster.local:8000"
```

## Make RGW Credentials available to Jupyter Notebook environment

We have to tell this deployment config to use the secret just created in the previous 
section with the S3 credentials: Access Key and Secret Key to be used to access our
Ceph Object cluster

```
oc set env --from=secret/s3-user-demo dc/jupyter-notebook
```

## Expose route to Jupyter Notebook Service

Once, our app and pod are ready and running, we can create a route to access the app Jupyter Notebook. 
And copy the route created to a browser.

```
watch oc get pods # wait until jupyter-notebook pod is running
oc expose svc/jupyter-notebook
oc get route jupyter-notebook
```

## Use browser to execute the Jupyter Notebook

Use the password: `developer` to access the files, and double click in the file named:
`sparksql-tensorflow.ipynb`. Then a new browser tab will open and you will be able to 
push button `Run`  (or click Shift + Enter) to execute all the actions in the notebook and see the results 
as a Data Scientist pro.
Please wait for the steps to be completed before running the next step, steps will show **[\*]**  when running and show the step number when finished.

At step **[10]** data will be ingested into the ceph cluster and as this will take several minutes you have the ability to check activity in the ceph-dashboard

After step **[10]** you can verify the contents in the bucket:

```
s3cmd ls s3://ceph-bucket/kube-metrics/
.....
2019-09-26 07:23         0   s3://ceph-bucket/kube-metrics/_SUCCESS
2019-09-26 07:23   1074414   s3://ceph-bucket/kube-metrics/part-00000-4421b6da-d349-454a-b37f-7ef68f755b6f-c000.json.bz2
2019-09-26 07:23   1075392   s3://ceph-bucket/kube-metrics/part-00001-4421b6da-d349-454a-b37f-7ef68f755b6f-c000.json.bz2
2019-09-26 07:23   1079779   s3://ceph-bucket/kube-metrics/part-00002-4421b6da-d349-454a-b37f-7ef68f755b6f-c000.json.bz2
.....
```

... Enjoy the experience!


