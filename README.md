# EMEA RHTE 2019: Learn how to run analytics and ML in OCP4 and Ceph Object storage with Rook.io

## Prequisites

* OpenShift 4 environment
* Administrative access for Rook-Ceph Operator

## Initial steps


Log in the clientvm through ssh with the credentials provided and git clone this 
repository to get all yml files for creating rook operator, ceph clusters, and jupyter notebook.

```
git clone https://github.com/luisrico/analytics-ml-lab.git
``` 

## Base Images

The lab has been deployed with a set of already built images to use for deploying Apache Spark
clusters and Jupyter Notebooks. These images are in the already created project "my-analytics". 
Check that all images have already been correctly created in the project/namespace "my-analytics". 
You should find 3 PODs with Status "Completed":

```
oc get all -n my-analytics 
``` 

For your reference, the commands used for building those images from scratch can be 
found [here](instructions/01-Base-Images.md). But, you DON'T have to execute those lengthy steps
unless there was any error in the lab deployment.

## Ceph Cluster

The main task is to deploy the Rook-Ceph operator and use it to deploy a Ceph
cluster with an object storage service. Instructions [here](instructions/02-Rook-Ceph.md).

## Jupyter Notebook

The final task is to deploy a Jupyter Notebook and run through it. 
Instructions [here](instructions/03-Jupyter-Notebook.md).
