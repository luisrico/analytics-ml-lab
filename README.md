# EMEA RHTE 2019: Learn how to run analytics and ML in OCP4 and Ceph Object storage with Rook.io

## Prequisites

* OpenShift 4 environment
* Administrative access for Rook-Ceph Operator

## Base Images

The first task is to build a set of images to use for deploying Apache Spark
clusters and Jupyter Notebooks. Instructions for building images from scratch
can be found [here](instructions/01-Base-Images.md).

## Ceph Cluster

The second task is to deploy the Rook-Ceph operator and use it to deploy a Ceph
cluster with an object storage service. Instructions [here](instructions/02-Rook-Ceph.md).

## Jupyter Notebook

The final task is to deploy a Jupyter Notebook. Instructions [here](instructions/03-Jupyter-Notebook.md).
