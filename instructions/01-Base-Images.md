# YOU DON'T NEED TO EXECUTE THESE STEPS IF LAB WAS CORRECTLY DEPLOYED

# Notebook Images

Create a new project to put our analytics stuff: 

```
oc new-project my-analytics
```

## Create UBI openshift-spark-py36-inc build

```
oc new-build --name=openshift-spark-py36-inc https://github.com/mmgaggle/openshift-spark#wip-ubi \
             --context-dir=openshift-spark-build-inc-py36 \
             --strategy=docker
```

## Watch openshift-spark-py36-inc build

```
oc logs -f bc/openshift-spark-py36-inc
```

## Create openshift-spark-py36 build

```
oc new-build --name=openshift-spark-py36 \
             -i openshift-spark-py36-inc:latest \
             -e SPARK_URL=http://mmgaggle-bd.s3.amazonaws.com/spark-2.3.2-bin-hadoop-2.8.5.tgz \
             -e SPARK_MD5_URL=http://mmgaggle-bd.s3.amazonaws.com/spark-2.3.2-bin-hadoop-2.8.5.tgz.md5 \
             --binary
oc start-build openshift-spark-py36
```

## Watch openshift-spark-py36 build (around 7 min)
```
oc logs -f bc/openshift-spark-py36
```

## Create Jupyter Notebook build

Using the openshift-spark-py36 image stream as a base, we'll create a jupyter notebook build. The resulting image stream can be utilized by the jupyterhub operator when provisioning notebooks, and will ensure the notebooks have the correct spark client side library versioning necessary to interact with spark clusters provisioned by the spark operator with the openshift-spark-py36 image stream.


```
oc new-build --name=jupyter-notebook \
             https://github.com/luisrico/analytics-ml-lab \
             --context-dir=notebook \
             -i openshift-spark-py36:latest \
             --strategy=docker
```

## Watch notebook build (around 18 min)
```
oc logs -f buildconfig/jupyter-notebook
```
