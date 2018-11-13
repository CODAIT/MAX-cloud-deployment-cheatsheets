# Deploy a MAX model-serving microservice on Google Cloud Platform

This document describes how to deploy the MAX model-serving microservices on Google Cloud Platform's [Kubernetes Engine](https://cloud.google.com/kubernetes-engine/).

> Note: The Kubernetes Engine service is not free.

### Prerequisites

1. Install [Docker Desktop](https://www.docker.com/products/docker-desktop).
2. Install the [Google Cloud SDK](https://cloud.google.com/sdk/).
3. Open a terminal window. 
   > Run commands prefixed with `$` in this terminal window.
4. Configure Docker to use `gcloud` as a credential helper.
   ```
   $ gcloud auth configure-docker
   ```
5. Create a new project.

   If you have already access to a Google Kubernetes Engine cluster use the project in which the cluster is defined and skip this step. Otherwise create a project and set it as default (to keep things simple).

   ```
   $ gcloud projects create max-deployments
   $ gcloud config set project max-deployments
   ```

### Publish the model container image in the Container Registry

Google Cloud Platform maintains a private [Docker repository](https://cloud.google.com/container-registry/), to which you have to upload the MAX model Docker container. If you are not familiar with the Container Registry or require more detailed instructions, refer to the [How-to Guides](https://cloud.google.com/container-registry/docs/how-to).

1. Clone the MAX model repository.

   ```
   $ git clone https://github.com/IBM/MAX-Object-Detector.git
   ```

2. Tag the Docker image.

   Tag the model Docker image using the [`Docker tag`](https://docs.docker.com/engine/reference/commandline/tag/) command, following the `[REGISTRY-HOSTNAME]/[PROJECT-ID]/[IMAGE]` naming convention as described [here](https://cloud.google.com/container-registry/docs/pushing-and-pulling).
 
   The command shown below tags the `max-object-detector` Docker image using REGISTRY-HOSTNAME `us.gcr.io` (US-based storage), the previsouly created `max-deployments` project id, and image name `max-object-detector`. 

   ```
   $ docker tag max-object-detector us.gcr.io/max-deployments/max-object-detector
   ```

3. Push the tagged Docker image to the Google Container Registry.

   Push the tagged image to the Container Registry using the [`Docker push`](https://docs.docker.com/engine/reference/commandline/push/) command.
 
   ```
   $ docker push us.gcr.io/max-deployments/max-object-detector
    The push refers to repository [us.gcr.io/max-deployments/max-object-detector]
    a90cd3170409: Pushed  
    ...

   $ gcloud container images list --repository us.gcr.io/max-deployments
    NAME
    us.gcr.io/max-deployments/max-object-detector
   ```

You can now deploy the image on the GCP Kubernetes Engine service.

### Deploy the container on Google Kubernetes Engine


#### Create a Kubernetes cluster

If you already have access to a Google Kubernetes cluster skip this section.

1. Create a Kubernetes cluster. 

   For _illustrative purposes_ we are creating a small Kubernetes cluster (1vCPU, 1.7GB RAM, 1 node). [Learn more about clusters](https://cloud.google.com/kubernetes-engine/docs/how-to/creating-a-cluster)

   > Note that some MAX model-services require additional resources (e.g. RAM). Check the model documentation for details.

    ```
    $ gcloud container clusters create max-cluster --machine-type=g1-small --max-nodes-per-pool=100 --num-nodes=1
    ```



#### Deploy the container 

1. Create a deployment from the container image on the Container Registry.

   ```
   $ kubectl run max-object-detector --image=us.gcr.io/max-deployments/max-object-detector --replicas=1
   ```

2. Expose the workload as a service.

   ```
   $ kubectl expose deploy --selector=run=max-object-detector --port=80 --target-port=5000 --name=max-model-service --type=LoadBalancer --load-balancer-ip=''
   ```

