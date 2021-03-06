# Get Started With Fluid

This document mainly describes how to deploy Fluid with Helm, and use Fluid to create a dataset and speed up your application.  

## Requirements  

1. Kubernetes 1.14+

    If you don't have a Kubernetes now, we highly recommend you use a cloud Kubernetes service. Usually, with a few steps, you can get your own Kubernetes Cluster. Here's some of the certified cloud Kubernetes services: 
    - [Amazon Elastic Kubernetes Service](https://aws.amazon.com/eks/)
    - [Google Kubernetes Engine](https://cloud.google.com/kubernetes-engine/)
    - [Azure Kubernetes Service](https://docs.microsoft.com/en-us/azure/aks/tutorial-kubernetes-deploy-cluster)
    - [Aliyun Container Service for Kubernetes](https://www.aliyun.com/product/kubernetes)

    > Note: While convenient, Minikube is not recommended to deploy Fluid due to its limited functionalities.

2. Kubectl 1.14+

    Please make sure your kubectl is properly configured to interact with your Kubernetes environment.

3. [Helm 3](https://helm.sh/docs/intro/install/)

    In the following steps, we'll deploy Fluid with Helm 3

## Deploy Fluid  
1. Create namespace for Fluid 
    ```shell
    $ kubectl create ns fluid-system
    ```  
2. Download the latest Fluid from Github [release page](https://github.com/fluid-cloudnative/fluid/releases)

3. Deploy Fluid with Helm
    ```shell
    $ helm install fluid fluid.tgz
    NAME: fluid
    LAST DEPLOYED: Tue Jul  7 11:22:07 2020
    NAMESPACE: default
    STATUS: deployed
    REVISION: 1
    TEST SUITE: None
    ```

4. Check running status of Fluid
    ```shell
    $ kubectl get po -n fluid-system
    NAME                                  READY     STATUS    RESTARTS   AGE
    controller-manager-6b864dfd4f-995gm   1/1       Running   0          32h
    csi-nodeplugin-fluid-c6pzj            2/2       Running   0          32h
    csi-nodeplugin-fluid-wczmq            2/2       Running   0          32h
    ```

## Create a Dataset  
Fluid provides cloud-native data acceleration and management capabilities, and use *dataset* as a high-level abstraction to facilitate user management. Here we will show you how to create a dataset with Fluid. 

1. Create a Dataset object through the CRD file, which describes the source of the dataset.  
    ```yaml
    apiVersion: data.fluid.io/v1alpha1
    kind: Dataset
    metadata:
      name: demo
    spec:
      mounts:
        - mountPoint: https://mirror.bit.edu.cn/apache/spark/spark-3.0.0/
          name: spark
    ```  
    Create dataset with kubectl
    
    ```shell
    kubectl create -f dataset.yaml
    ```

2. Create an `AlluxioRuntime` CRD object to support the dataset we created. We use [Alluxio](https://www.alluxio.io/) as its runtime here.
    ```yaml
    apiVersion: data.fluid.io/v1alpha1
    kind: AlluxioRuntime
    metadata:
      name: demo
    spec:
      replicas: 1
      tieredstore:
        levels:
          - mediumtype: MEM
            path: /dev/shm
            quota: 2Gi
            high: "0.95"
            low: "0.7"
      properties:
        alluxio.user.file.writetype.default: MUST_CACHE
        alluxio.master.journal.folder: /journal
        alluxio.master.journal.type: UFS
        alluxio.user.block.size.bytes.default: 256MB
        alluxio.user.streaming.reader.chunk.size.bytes: 256MB
        alluxio.user.local.reader.chunk.size.bytes: 256MB
        alluxio.worker.network.reader.buffer.size: 256MB
        alluxio.user.streaming.data.timeout: 300sec
      master:
        jvmOptions:
          - "-Xmx4G"
      worker:
        jvmOptions:
          - "-Xmx4G"
      fuse:
        jvmOptions:
          - "-Xmx4G "
          - "-Xms4G "
        args:
          - fuse
          - --fuse-opts=direct_io,ro,max_read=131072
    ```
    
    Create *Alluxio* Runtime with `kubectl`
    
    ```shell
    kubectl create -f runtime.yaml  
    ``` 

3. Next, we create an application to access this dataset. Here we will access the same data multiple times and compare the time consumed by each access.

    ```yaml
    apiVersion: v1
    kind: Pod
    metadata:
      name: demo-app
    spec:
      containers:
        - name: demo
          image: nginx
          volumeMounts:
            - mountPath: /data
              name: demo
      volumes:
        - name: demo
          persistentVolumeClaim:
            claimName: demo
    ```
    
    Create Pod with `kubectl`
    
    ```shell
    kubectl create -f app.yaml
    ```

4. Dive into the container to access data, the first access will take longer.
    ```
    kubectl exec -it demo-app -- bash
    #  du -sh /data/spark/spark-3.0.0-bin-without-hadoop.tgz
    150M	/data/spark/spark-3.0.0-bin-without-hadoop.tgz
    # time cp /data/spark/spark-3.0.0-bin-without-hadoop.tgz /dev/null
    real	0m13.171s
    user	0m0.002s
    sys	0m0.028s
    ```

5. In order to avoid the influence of other factors like page cache, we will delete the previous container, create the same application, and try to access the same file. Since the file has been cached by alluxio at this time, you can see that it takes significantly less time now.
    ```
    kubectl delete -f app.yaml && kubectl create -f app.yaml
    ...
    # time cp /data/spark/spark-3.0.0-bin-without-hadoop.tgz /dev/null
    real	0m0.344s
    user	0m0.002s
    sys	0m0.020s
    ```

We've created a dataset and did some management in a very simple way. For more detail about Fluid, we provide several sample docs for you:
- [Speed Up Accessing Remote Files](../samples/accelerate_data_accessing.md)
- [Cache Co-locality for Workload Scheduling](../samples/data_co_locality.md)
- [Accelerate Machine Learning Training with Fluid](../samples/machinelearning.md)