# EMEA RHTE 2019: Learn how to run analytics and ML in OCP4 and Ceph Object storage with Rook.io
JR
## Prequisites

* OpenShift 4 environment
* Administrative access for Rook-Ceph Operator

## Initial steps

## Remove cpu and memory limits from default quota

```
oc edit userquota default
```

Log in the clientvm through ssh with the credentials provided and git clone this 
repository to get all yml files for creating rook operator, ceph clusters, and jupyter notebook.

```
git clone https://github.com/luisrico/analytics-ml-lab.git
``` 

## Base Images

The first task is to build a set of images to use for deploying Apache Spark
clusters and Jupyter Notebooks. Instructions for building images from scratch
can be found [here](instructions/01-Base-Images.md).

## Ceph Cluster

The second task is to deploy the Rook-Ceph operator and use it to deploy a Ceph
cluster with an object storage service. Instructions [here](instructions/02-Rook-Ceph.md).

## Jupyter Notebook

The final task is to deploy a Jupyter Notebook. Instructions [here](instructions/03-Jupyter-Notebook.md).
