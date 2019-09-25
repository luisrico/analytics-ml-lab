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
watch oc get pods # wait until jupyter-notebook pod is created
oc expose svc/jupyter-notebook
oc get route jupyter-notebook
```

## Use browser to execute the Jupyter Notebook

Use the password: `developer` to access the files, and double click in the file named:
`sparksql-tensorflow.ipynb`. Then a new browser tab will open and you will be able to 
push button `Run`  (or click Shift + Enter) to execute all the actions in the notebook and see the results 
as a Data Scientist pro... Enjoy the experience!

