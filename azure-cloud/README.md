# Deploy a MAX model-serving microservice on Azure Kubernetes Service

This document describes how to deploy the MAX model-serving microservices on [Azure Kubernetes Service](https://azure.microsoft.com/en-us/services/kubernetes-service/) (AKS).

### Prerequisites

1. Install [Docker Desktop](https://www.docker.com/products/docker-desktop).
2. Install the [Azure CLI](https://docs.microsoft.com/en-us/cli/azure/?view=azure-cli-latest).
3. Open a terminal window. 
   > Run commands prefixed with `$` in this terminal window.

### Publish the model container image in the Container Registry

Google Cloud Platform maintains a private [Docker repository](https://cloud.google.com/container-registry/), to which you have to upload the MAX model Docker container image. If you are not familiar with the Container Registry or require more detailed instructions, refer to the [How-to Guides](https://cloud.google.com/container-registry/docs/how-to).

#### Create a Container Registry


X. Create a resource group if none is defined yet.

  ```
  $ az group create --name default --location westus
  ```

X. Create a Container Registry.

  ```
  $ az acr create --resource-group default --name maxdeployments --sku Basic
  ```

X.Log in to the Container Registry

  ```
  $ az acr login --name maxdeployments
  ```

1. Clone the desired MAX model repository and build the Docker image using the [`docker build`](https://docs.docker.com/engine/reference/commandline/build/) command. 

    ```
    $ git clone https://github.com/IBM/MAX-Object-Detector.git
    $ cd MAX-Object-Detector
    $ docker build -t max-object-detector .
    ```

   > The example instructions use the MAX-Object-Detector model for illustrative purposes. 


2. Tag the Docker image for uploading.

   Tag the Docker image using the [`docker tag`](https://docs.docker.com/engine/reference/commandline/tag/) command, following the `[REGISTRY-HOSTNAME]/[PROJECT-ID]/[IMAGE]` naming convention as described [here](https://cloud.google.com/container-registry/docs/pushing-and-pulling).
 
   The command shown below tags the `max-object-detector` Docker image using REGISTRY-HOSTNAME `us.gcr.io` (US-based storage), the previsouly created `max-deployments` project id, and image name `max-object-detector`. 

   ```
   $ docker tag max-object-detector maxdeployments.azurecr.io/max-object-detector

   $ docker images
    REPOSITORY                                      TAG                 IMAGE ID        ...    
    max-object-detector                             latest              0ba72cc8c62b    ...    
    maxdeployments.azurecr.io/max-object-detector   latest  
   ```

3. Push the tagged Docker image to the Container Registry.

   Push the tagged image to the Container Registry using the [`Docker push`](https://docs.docker.com/engine/reference/commandline/push/) command.
 
   ```
   $ docker push maxdeployments.azurecr.io/max-object-detector
    The push refers to repository [maxdeployments.azurecr.io/max-object-detector]
    d04398a13d59: Pushed 
    ...

   $ az acr repository list --name maxdeployments --output table
    Result
    -------------------
    max-object-detector
   ```

You can now deploy the image on the Azure Kubernetes Service.

### Deploy the container on Azure Kubernetes Service


#### Create a Kubernetes cluster

If you already have access to a Kubernetes cluster skip this section. For detailed information about how to create an Azure Kubernetes Service cluster refer to the [tutorial](https://docs.microsoft.com/en-us/azure/aks/tutorial-kubernetes-deploy-cluster).

1. Create a Kubernetes cluster. 

   Many MAX model-serving microservices require at a minimim 1 CPU and 2GB of RAM.
   > Note that some MAX model-serving microservices require additional resources. Check the model documentation for details.

    ```
    $ az ad sp create-for-rbac --skip-assignment
    $ az acr show --resource-group default --name maxdeployments --query "id" --output tsv

    $ az role assignment create --assignee 7adc2f2f-be0a-4819-bebc-2a014f1cd6af --scope /subscriptions/70fa6736-1c48-4cbc-af70-b9beebe42b40/resourceGroups/default/providers/Microsoft.ContainerRegistry/registries/maxdeployments --role Reader

    $ az aks create \
    --resource-group default \
    --name maxCluster \
    --node-count 1 \
    --service-principal 7adc2f2f-be0a-4819-bebc-2a014f1cd6af \
    --client-secret a51d3ac3-5085-4535-a702-1b12b939cedf \
    --generate-ssh-keys \
    --node-vm-size Standard_DS1
     ...

    $ az aks list --resource-group default
     NAME                            ZONE        MACHINE_TYPE  PREEMPTIBLE  INTERNAL_IP  EXTERNAL_IP  STATUS
     gke-max-cluster-...             us-west1-a  g1-small                   1...2        3...0        RUNNING
    ```

2. Connect to the Kubernetes cluster.

   You use `kubectl` to connect to the Kubernetes cluster. 
   
   > If necessary install it by running `az aks install-cli` before continuing.

   ```
   $ az aks get-credentials --resource-group default --name maxCluster
   ```

3. List cluster nodes.
   ```
   $ kubectl get nodes
    NAME                STATUS    ROLES     AGE       VERSION
    aks-nodepool...-0   Ready     agent     14m       v1.9.11
   ```

#### Deploy the container and expose it as a service


1. Create a deployment from the container image on the Container Registry. In this example we are creating only a single pod. (The node size in your cluster limits how many replicas you can run at any point in time.)

   ```
   $ kubectl run max-object-detector --image=maxdeployments.azurecr.io/max-object-detector --replicas=1
    deployment.apps "max-object-detector" created
   ```

3. Wait until the desired number of pods was started.

   ```
   $ kubectl get pods
    NAME                                READY     STATUS        RESTARTS   AGE
    max-object-detector-...h            1/1       Running       0          4m
   ```

3. Expose the workload as a service.

   ```
   $ kubectl expose deployment max-object-detector --port=80 --target-port=5000 --name=max-model-service --type=LoadBalancer --load-balancer-ip=''
    ...
   ```

4. Retrieve the service's external IP address.

   Service creation can take a couple minutes. Wait until an external IP address is displayed for the `max-model-service`.

   ```
   $ kubectl get service --watch
    NAME                TYPE           CLUSTER-IP     EXTERNAL-IP   PORT(S)        AGE
    kubernetes          ClusterIP      10.0.0.1       <none>        443/TCP        38m
    max-model-service   LoadBalancer   1...5          <pending>     80:30997/TCP   1m
    max-model-service   LoadBalancer   1...5          1...2         80:30997/TCP   1m
   ```

5. Verify that you can access the model-serving microservice using the displayed external IP address.

   ```
   $ curl http://1...2:80/model/metadata
    {"id": "ssd_mobilenet_v1_coco_2017_11_17-tf-mobilenet", 
     "name": "ssd_mobilenet_v1_coco_2017_11_17 TensorFlow Model", 
     "description": "ssd_mobilenet_v1_coco_2017_11_17 TensorFlow model trained on MobileNet", 
     "license": "ApacheV2"}
   ```

6. Explore the service endpoints.

   Open `http://1...2:80/` in a browser window to explore the service endpoints using the Swagger specification.