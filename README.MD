# Event driven autoscaling of Kubernetes workloads using KEDA
![enter image description here](https://raw.githubusercontent.com/krvarma/event-driven-autoscaling-using-keda/master/images/keda.png)
[In my previous article](https://medium.com/swlh/rabbitmq-trigger-and-azure-functions-8826633bf54c), we explored how to use RabbitMQ Extension for Azure Functions. We discussed how to create an Azure Function using RabbitMQ Extension. The function gets triggered when we send a message to a RabbitMQ Queue. We also discussed how to generate a Dockerfile so that we can use it in production environments.

In this article, we will explore how to deploy this function in a Kubernetes cluster and autoscale it using KEDA.

## What is KEDA?

KEDA is Kubernetes Event-Driven Autoscaling. KEDA is a lightweight component that determines when a container should be scaled. KEDA works along with Kubernetes's Horizontal Pod Autoscaler. KEDA can scale any container in Kubernetes from 0 to n based on how many events are waiting to be processed.

KEDA can process Kubernetes Deployments and Jobs. Deployments are a common way to scale the workloads. When there is a long-running task, we have to use Kubernetes Jobs.

Some of the crucial concepts in KEDA are:

1.  **Scaling Agent**: The primary goal of the Scaling Agent is to activate and deactivate deployments
2.  **Metrics Server**: The Metrics Server acts as a Kubernetes Metrics Server. Metrics Server sends scaling parameters like Queue Length, Number of messages in a stream to HPA so that HPA initiates scaling
3.  **Scaler**:  Scalers are responsible for collecting metrics for an event source. The scale handler talks to the scaler to get the event metrics. Depends on the reported event metrics by the scaler, the scale handler scales the deployment or job. Scaler acts as a bridge between KEDA and external event sources.
4.  **ScaledObject**: ScaledObject is a Kubernetes Custom Resource Definition (CRD). We should deploy a ScaledaObject to scale a Deployment. ScaledObject defines the deployment name, minimum replicas,  maximum replicas, metadata for scaler.

## Deploy KEDA to the cluster

KEDA provides Helm charts to deploy to the cluster. First, we need to add the Helm repo.

    helm repo add kedacore https://kedacore.github.io/charts
    helm repo update

Then we need to create a namespace (this is not a mandatory step, you can install KEDA to the default namespace). After that, we can install KEDA using Helm. I suppose you are having Helm 3.

    kubectl create namespace keda
    helm install keda kedacore/keda --namespace keda

Now we have deployed KEDA to our cluster. We can verify it by getting the pods.

    kubectl get pods --namespace=keda

You can see the KEDA operator and metrics server pods are running.

    NAME 												READY 	STATUS 	RESTARTS 	AGE
    keda-operator-678fb469f5-xlz6q 						1/1 	Running	3 			29h
    keda-operator-metrics-apiserver-5d6c9c96d9-drst4	1/1 	Running	2			29h

Now we need to deploy our function and enable KEDA.

## Deploy Azure Function on Kubernetes and enable KEDA

To deploy our Azure Function on Kubernetes, we need to generate the YAML file with Deployment resources. We can manually create the YAML file or use the Azure Functions Core Tools to create the file. We will use the Azur Functions Core Tools to create the file. We can use the `func kubernetes` command to generate the YAML file. To generate the YAML file, go to the project folder, and run the following command:

    func kubernetes deploy --name <name-of-function-deployment> --registry <container-registry-username>

This command will create a YAML file and deploy it to the cluster. The `name` parameter specifies the name of the deployment. The `registry` parameter specifies the container registry. The `registry` parameter tells the tool to run a docker build and push the image to the registry/name. Alternatively, you can use `image-name` to pull an image. The `registry` and `image-name` parameters are mutually exclusive.

The above command will deploy the function to the Kubernetes cluster. The `dry-run` parameter will display the YAML. Use output redirection operator to save the content.

    func kubernetes deploy --name <name-of-function-deployment> --registry <container-registry-username> > func-deployment.yml

This command will generate a Deployment resource, ScaledObject resource, and a Secrets resource. The Secrets resource is generated using the contents in the local.settings.json file. Here is the generated resource definition.

    data:
      AzureWebJobsStorage: VXNlRGV2ZWxvcG1lbnRTdG9yYWdlPWZhbHNl
      FUNCTIONS_WORKER_RUNTIME: ZG90bmV0
      RabbitMqConnection: <Base64 of your connection string>
    apiVersion: v1
    kind: Secret
    metadata:
      name: mqfn
      namespace: default
    ---
    apiVersion: apps/v1
    kind: Deployment
    metadata:
      name: mqfn
      namespace: default
      labels:
        app: mqfn
    spec:
      selector:
        matchLabels:
          app: mqfn
      template:
        metadata:
          labels:
            app: mqfn
        spec:
          containers:
          - name: mqfn
            image: <image-name>
            env:
            - name: AzureFunctionsJobHost__functions__0
              value: fn
            envFrom:
            - secretRef:
                name: mqfn
    ---
    apiVersion: keda.k8s.io/v1alpha1
    kind: ScaledObject
    metadata:
      name: mqfn
      namespace: default
      labels:
        deploymentName: mqfn
    spec:
      scaleTargetRef:
        deploymentName: mqfn
      pollingInterval: 5   # Optional. Default: 30 seconds
      cooldownPeriod: 30   # Optional. Default: 300 seconds
      minReplicaCount: 0   # Optional, Default 0
      maxReplicaCount: 30  # Optional. Default: 100
      triggers:
      - type: rabbitmq
        metadata:
          type: rabbitMQTrigger
          queueName: sampleq
          name: inputMessage
          host: RabbitMqConnection
    ---

We need to make one more change manually to deploy to the cluster. We need to add a metadata queueLength. The queueLength represents the number of messages in the RabbitMQ Queue, the HPA scales pods based on this number. For example, if the queueLength is 10 and there are 30 messages in the RabbitMQ queue, the HPA will scale to 3 pods. Here is the final template.

    data:
      AzureWebJobsStorage: VXNlRGV2ZWxvcG1lbnRTdG9yYWdlPWZhbHNl
      FUNCTIONS_WORKER_RUNTIME: ZG90bmV0
      RabbitMqConnection: <Base64 of your connection string>
    apiVersion: v1
    kind: Secret
    metadata:
      name: mqfn
      namespace: default
    ---
    apiVersion: apps/v1
    kind: Deployment
    metadata:
      name: mqfn
      namespace: default
      labels:
        app: mqfn
    spec:
      selector:
        matchLabels:
          app: mqfn
      template:
        metadata:
          labels:
            app: mqfn
        spec:
          containers:
          - name: mqfn
            image: <image-name>
            env:
            - name: AzureFunctionsJobHost__functions__0
              value: fn
            envFrom:
            - secretRef:
                name: mqfn
    ---
    apiVersion: keda.k8s.io/v1alpha1
    kind: ScaledObject
    metadata:
      name: mqfn
      namespace: default
      labels:
        deploymentName: mqfn
    spec:
      scaleTargetRef:
        deploymentName: mqfn
      pollingInterval: 5   # Optional. Default: 30 seconds
      cooldownPeriod: 30   # Optional. Default: 300 seconds
      minReplicaCount: 0   # Optional, Default 0
      maxReplicaCount: 30  # Optional. Default: 100
      triggers:
      - type: rabbitmq
        metadata:
          type: rabbitMQTrigger
          queueName: sampleq
          name: inputMessage
          host: RabbitMqConnection
          queueLength  : '20'
    ---
    
Let's deploy the function to the cluster.

    kubectl apply -f func-deployment.yml

Let's check the deployment.

    kubectl get pods -w

You cannot see the function pod since KEDA scaled it to zero.

Now let's run the RabbitMQ producer in another command shell to send a message to the queue.

    node send.js TemperatureReading 19.2

If everything goes well, you can see the function pod running.

![enter image description here](https://raw.githubusercontent.com/krvarma/event-driven-autoscaling-using-keda/master/images/getpods.svg)

## The ScaledObject

Let's look at the ScledObject in detail and understand it. Given below is our ScaledObject.

    apiVersion: keda.k8s.io/v1alpha1
    kind: ScaledObject
    metadata:
      name: mqfn
      namespace: default
      labels:
        deploymentName: mqfn
    spec:
      scaleTargetRef:
        deploymentName: mqfn
      pollingInterval: 5   # Optional. Default: 30 seconds
      cooldownPeriod: 30   # Optional. Default: 300 seconds
      minReplicaCount: 0   # Optional, Default 0
      maxReplicaCount: 30  # Optional. Default: 100
      triggers:
      - type: rabbitmq
        metadata:
          type: rabbitMQTrigger
          queueName: sampleq
          name: inputMessage
          host: RabbitMqConnection
          queueLength  : '200'
    

This object contains apiVersion, kind, metadata as usual. Then comes some of the interesting elements:

1.  **scaleTargetRef** - tells the HPA which deployment to scale.
2.  **minReplicaCount** - Minimum number of pods to maintain, in our case it is 0
3.  **minReplicaCount** - Maximum number of pods to create
4.  **pollingInterval** - Polling Interval, how often to contact the event source for new metrics. We should select the value judiciously, otherwise, there will be a performance hit.
5.  **cooldownPeriod** - Cool Down period specifies how long the system should wait before terminating the pod.
6.  **type** - Type specifies the scaler, in our case it is rabbitmq.
7.  **metadata** - Metadata is scaler specific, it will be passed on to the selected scalar.

In the next article, we will explore how to create a custom scaler.



