# Quickstart

This tutorial shows to run ML model serving at scale on AWS using Seldon Core.

## Pre-Requisites
### Kubernetes cluster
To install Seldon Core you'll need a Kubernetes cluster version equal or higher than 1.12.

Follow the latest [AWS EKS docs](https://docs.aws.amazon.com/eks/latest/userguide/create-cluster.html) to find the setup option that best suits your needs.

Below is an example of how to create EKS cluster using `eksctl`:
1. [Install AWS CLI version 2](https://docs.aws.amazon.com/cli/latest/userguide/install-cliv2.html)
2. [Configure the AWS CLI](https://docs.aws.amazon.com/cli/latest/userguide/cli-chap-configure.html)
3. [Install eksctl](https://docs.aws.amazon.com/eks/latest/userguide/eksctl.html)
4. Run in terminal:
    ```bash
    eksctl create cluster \
    --name eks-model-serving \
    --version 1.21 \
    --region us-east-2 \
    --nodegroup-name linux-nodes \
    --node-type t3.medium \
    --nodes 2 \
    --nodes-min 2 \
    --nodes-max 2 \
    --managed
    ```
5. [Install kubectl](https://kubernetes.io/docs/tasks/tools/#kubectl)
    
    For example, to install on macOS using Homebrew:
    ```bash
    brew install kubectl
    ```
6. Configure kubectl:
   ```bash
   aws eks --region us-east-2 update-kubeconfig --name eks-model-serving
   ```
7. Verify kubectl is properly configured:
   ```bash
   kubectl cluster-info
   ```
### Helm
You'll need Helm version equal or higher than 3.0.

Follow the latest [Helm docs](https://helm.sh/docs/intro/install/) to find the setup option that best suits your needs.

For example, to install helm with Homebew on macOS run:
```bash
brew install helm
```

### Ingress
In order to route traffic to your models you will need to install Ingress. Seldon Core supports Istio or Ambassador. In this tutorial we will use Ambassador.

Follow the latest [Ambassador docs](https://www.getambassador.io/docs/edge-stack/latest/topics/install/) to find the setup option that best suits your needs.

For example, to install via helm 3 run:
```bash
helm repo add datawire https://www.getambassador.io
kubectl create namespace ambassador
helm install ambassador --namespace ambassador datawire/ambassador
```
Finish the installation by running the following command: 
```bash
edgectl install
```
[Edge Control](https://www.getambassador.io/docs/edge-stack/latest/topics/using/edgectl/edge-control) (edgectl) automatically configures TLS for your instance and provisions a domain name for your Ambassador Edge Stack. This is not necessary if you already have a domain name and certificates.

The output will contain the created DNS, for example:
```bash
Congratulations! You've successfully installed the Ambassador Edge Stack in
your Kubernetes cluster. You can find it at your custom URL:
https://great-shtern-3456.edgestack.me/
```
You can get the hosts name by getting the custom `host` resource from the kubernetes cluster. Save it in a variable for a later use:
```bash
export LOAD_BALANCER_URL=$(kubectl get host -n ambassador -o jsonpath='{.items[0].spec.hostname}')
echo $LOAD_BALANCER_URL
```

## Installing Seldon Core
Follow the latest [Seldon install docs](https://docs.seldon.io/projects/seldon-core/en/latest/workflow/install.html).

For example, to install via helm 3 run:
```bash
kubectl create namespace seldon-system
helm install seldon-core seldon-core-operator \
    --repo https://storage.googleapis.com/seldon-charts \
    --set usageMetrics.enabled=true \
    --namespace seldon-system \
    --set ambassador.enabled=true
```

## Deploying a model using a pre-packaged inference server
1. Create a namespace for your models:
    ```bash
    kubectl create namespace my-models
    ```
2. Apply a SeldonDeployment to deploy your model:
```bash
kubectl apply -f - << END
apiVersion: machinelearning.seldon.io/v1
kind: SeldonDeployment
metadata:
  name: iris-model
  namespace: my-models
spec:
  name: iris
  predictors:
  - graph:
      implementation: SKLEARN_SERVER
      modelUri: gs://seldon-models/v1.10.0-dev/sklearn/iris
      name: classifier
    name: default
    replicas: 1
END
```
3. Check deployment status. You should get `"state": "Available"`:
    ```bash
    kubectl get sdep iris-model -o json --namespace my-models | jq .status
    ```
4. Now access the OpenAPI UI of your deployed model: `http://<ingress_url>/seldon/<namespace>/<model-name>/api/v1.0/doc/`
   
   Which in our example resolves to:
   ```bash
   echo https://$LOAD_BALANCER_URL/seldon/my-models/iris-model/api/v1.0/doc/
   ```

    You can use the OpenAPI UI to send requests to your model and get prediction results. Try it out with this data:
    ```json
    { "data": { "ndarray": [[1,2,3,4]] } }
    ```

    Alternatively you can use any other client, for example `curl` or `Postman`, as well as [Seldon Python Client](https://docs.seldon.io/projects/seldon-core/en/latest/python/seldon_client.html):
    ```bash
    curl -X POST https://$LOAD_BALANCER_URL/seldon/my-models/iris-model/api/v1.0/predictions \
    -H 'Content-Type: application/json' \
    -d '{ "data": { "ndarray": [[1,2,3,4]] } }'
    ```
    Note: If you haven't set up a SSL certificate properly (which we won't in the scope of this tutorial), you will need to ignore SSL errors in your clients. For example in curl add `-k` argument.

## Scaling
In this section I will show how to handle increasing scale by adding more replicas.
In order to measure the effectiveness of the additional replicas, we will run a load test using tool called `locust`.

### Installing and running locust
1. Create and activate a virtual environment.
    
    For example using virtualenvwrapper:
    ```bash
    mkvirtualenv seldon-core-quickstart-tutorial
    workon seldon-core-quickstart-tutorial
    ```
2. Install locust:
    ```bash
    pip install locust
    ```
3. Run locust:
   ```bash
   locust
   ```
   Then open web UI: http://0.0.0.0:8089

   Or run headless:
   ```bash
   locust --headless --users 20 --spawn-rate 1 -H https://$LOAD_BALANCER_URL
   ```
4. Take note of the median and 95% percentile response times.

### Adding replicas
5. Now let's increase the number of replicas:
   ```bash
   kubectl scale sdep/iris-model -n my-models --replicas=4
   ```
   NOTE: As of the moment of writing this tutorial I couldn't get the above command to work.
   
   We can achieve the desired result by re-applying the SeldonDeployment with 2 replicas instead:  
```bash
kubectl apply -f - << END
apiVersion: machinelearning.seldon.io/v1
kind: SeldonDeployment
metadata:
  name: iris-model
  namespace: my-models
spec:
  name: iris
  predictors:
  - graph:
      implementation: SKLEARN_SERVER
      modelUri: gs://seldon-models/v1.10.0-dev/sklearn/iris
      name: classifier
    name: default
    replicas: 4
END
```
6. Check rollout status:
    ```bash
    kubectl rollout status deploy/$(kubectl get deploy -l seldon-deployment-id=iris-model -n my-models -o jsonpath='{.items[0].metadata.name}') -n my-models
    ```
7. Verify replicas count:
   ```bash
   kubectl get deploy -l seldon-deployment-id=iris-model -n my-models -o jsonpath='{.items[0].status.replicas}'
   ```
8. Finally, check locust to see the change in the median and 95% percentile response times.
