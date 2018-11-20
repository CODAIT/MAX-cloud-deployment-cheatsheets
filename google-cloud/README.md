# Deploy a MAX model-serving microservice on Google Cloud Platform

This document describes how to deploy deep learning models from the [Model Asset Exchange](https://developer.ibm.com/exchanges/models/) as model-serving microservices on [Google Kubernetes Engine](https://cloud.google.com/kubernetes-engine/).

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

Google Cloud Platform maintains a private [Docker repository](https://cloud.google.com/container-registry/), to which you have to upload the MAX model Docker container image. If you are not familiar with the Container Registry or require more detailed instructions, refer to the [How-to Guides](https://cloud.google.com/container-registry/docs/how-to).

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
   $ docker tag max-object-detector us.gcr.io/max-deployments/max-object-detector

   $ docker images
    REPOSITORY                                      TAG                 IMAGE ID        ...    
    max-object-detector                             latest              0ba72cc8c62b    ...    
    us.gcr.io/max-deployments/max-object-detector   latest              0ba72cc8c62b    ...    
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

   > Note that some MAX model-serving microservices require additional resources (e.g. RAM). Check the model documentation for details.

    ```
    $ gcloud container clusters create max-cluster --machine-type=g1-small --max-nodes-per-pool=100 --num-nodes=1
     ...

    $ gcloud compute instances list
     NAME                            ZONE        MACHINE_TYPE  PREEMPTIBLE  INTERNAL_IP  EXTERNAL_IP  STATUS
     gke-max-cluster-...             us-west1-a  g1-small                   1...2        3...0        RUNNING
    ```

#### Deploy the container and expose it as a service

1. Create a deployment from the container image on the Container Registry. In this example we are creating only a single pod.

   ```
   $ kubectl run max-object-detector --image=us.gcr.io/max-deployments/max-object-detector --replicas=1
    deployment.apps "max-object-detector" created
   ```

2. Wait until the desired number of pods was started.

   ```
   $ kubectl get pods
    NAME                                     READY     STATUS        RESTARTS   AGE
    max-object-detector-7458679574-mv4d5     1/1       Running       0          1m
   ```

3. Expose the workload as a service.

   ```
   $ kubectl expose deployment max-object-detector --port=80 --target-port=5000 --name=max-model-service --type=LoadBalancer --load-balancer-ip=''
    ...
   ```

4. Retrieve the service's external IP address.

   ```
   $ kubectl get service
    NAME                TYPE           CLUSTER-IP     EXTERNAL-IP      PORT(S)        AGE
    kubernetes          ClusterIP      1...1          <none>           443/TCP        2d
    max-model-service   LoadBalancer   1...9          3...0            80:30164/TCP   2m
   ```

5. Verify that you can access the model-serving microservice using the displayed IP address.

   ```
   $ curl http://3...0:80/model/metadata
     {"id": "facenet-tensorflow", 
      "name": "facenet TensorFlow Model", 
      "description": "facenet TensorFlow model trained on LFW data to detect faces and generate embeddings",
      "license": "MIT"}

6. Explore the service endpoints.

   Open `http://3...0:80/` in a browser window to explore the service endpoints using the Swagger specification.

   ![model-serving microservice endpoints](images/swagger.png)
