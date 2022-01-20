# POC: Self Hosted Runners on Kubernetes Cluster for Github Actions

## Context

GitHub Actions is a very useful tool for automating development. GitHub Actions jobs are run in the cloud by default, but you may want to run your jobs in your environment. 

An example usecase would be to create a Github Action to create and manage topics and ACLs on non-public Kafka Cluster. Self-hosted runner can be used for such use cases, but requires the provisioning and configuration of a virtual machine instance. Instead if you already have a Kubernetes cluster, it makes more sense to run the self-hosted runner on top of it. [action-runner-controller](https://github.com/actions-runner-controller) makes that possible.

This POC is intended to try out self hosted runners on Kubernetes and covers
1. Setting a Kubernetes Cluster using minikube
2. Deploying minimal Kafka Cluster on minikube which is accessible only within Kuberenets Cluster
3. Deploying self hosted action runners on the minikube
4. Running Github Action Runner on minikube to create topics on Kafka.

## Setting up Github Authnetication

### Types of Action Runners

The Github Hosted Runners can be deployed for a specific repository, organiztion or enterprize. The repository runners can run Github Action of specific repository whereas organization and enterprize runners can be shared across multiple repository across the organization or entereprize respectively.

### Authenctication

There are two ways for actions-runner-controller to authenticate with the GitHub API (only 1 can be configured at a time however):

1. Using a GitHub App (not supported for enterprise level runners due to lack of support from GitHub)
1. Using a Personal Access Token (PAT)

**Note: For the scope of this POC, authentication is done using PAT and repository runners are used. Please refer the [guide](https://github.com/actions-runner-controller/actions-runner-controller#setting-up-authentication-with-github-api) to create PAT and assign required permission to it**

## Preparing Kuberenets Cluster

1. Install and start minikube cluster. Details can be found [here](https://minikube.sigs.k8s.io/docs/start/)

### Install cert-manager 

1. Install cert-manager on the k8s 

    ```
    helm repo add jetstack https://charts.jetstack.io

    kubectl create namespace cert-manager
    helm install --wait --namespace cert-manager cert-manager jetstack/cert-manager --version v1.3.0 --set installCRDs=true

    # Verify Installation
    kubectl --namespace cert-manager get all
    ```

1. Install `action-runner-controller`

    ```
    helm repo add actions-runner-controller https://actions-runner-controller.github.io/actions-runner-controller

    kubectl create namespace actions-runner-system
    helm install --wait --namespace actions-runner-system actions-runner-controller actions-runner-controller/actions-runner-controller

    # Verify Installation
    kubectl --namespace actions-runner-system get all
    ```

1. Install Kafka

    ```
    helm repo add bitnami https://charts.bitnami.com/bitnami

    kubectl create namespace kafka
    helm install --namespace kafka kafka bitnami/kafka

    kubectl --namespace kafka get all
    ```
1. Create a pod to interact with Kafka

    ```
    kubectl run kafka-client --restart='Never' --image docker.io/bitnami/kafka:2.8.1-debian-10-r57 --namespace kafka --command -- sleep infinity
    ```
### Install Action Runners

1. The simple runner configuration is defined in a file [simple-repo-action-runner.yaml](./simple-repo-action-runner.yaml). These runners are tied to the `abhishekvm/self-hosted-runners-gha-poc`. Replace the repository name with your repository.

  ```
  apiVersion: actions.summerwind.dev/v1alpha1
  kind: RunnerDeployment
  metadata:
    name: k8s-action-runner
  spec:
    replicas: 2
    template:
      spec:
        repository: abhishekvm/self-hosted-runners-gha-poc
        workDir: /home/runner
  ```
- **replicas**: Number of runners to spin up
- **workDir**: Default working directory 

2. Deploy Action Runners

  ```
  kubectl create namespace action-runners
  kubectl --namespace action-runners apply -f simple-repo-action-runner.yaml

  # Verify that Action Runners are ready
  kubectl --namespace action-runners get runner
  ```

3. If everything goes well you should see two action runners on the kubernetes and same are registered on the Github. Check under `Settings > Actions > Runner` of your repository.

```
$ kubectl get runner -n  action-runners

NAME                            ENTERPRISE   ORGANIZATION   REPOSITORY                                     LABELS   STATUS    AGE
k8s-action-runner-t29xc-gcqjp                               abhishekvm/self-hosted-runners-gha-poc         Running   3d14h
k8s-action-runner-t29xc-gmmks                               abhishekvm/self-hosted-runners-gha-poc         Running   3d14h
```
<img width="1440" alt="CleanShot 2021-12-14 at 14 30 18@2x" src="https://user-images.githubusercontent.com/93121924/145966490-c1a66ca0-a45f-4b0c-b6a4-1d9de23ff315.png">

### Github Action in Action

1. Define a Github Action. [Here](.github/workflows/topics.yaml) is the example that shows running a Github Action to create topics on non-publically accessible Kafka.

1. `runs-on: [self-hosted]` in the workflow tells Github to run the workflow on the `self-hosted` runners. Github Action matches the tag specified in the `run-on` with the tags of the runner and runs the action on appropriate runner. By default all the runners get default tag as `self-hosted` and you can add more tages if you need.

1. The sample workflow also uses a docker in docker feature to run a `kafka-configurator` docker container on the Action Runner Container.

1. Add following snippet into [topics.yaml](topics.yaml) cerate one more topic on the Kafka. As soon as the file is updated Github Action will be triggered automatically.

  ```
  topic2:
    partitions: 3
    replication: 1
    config:
      cleanup.policy: compact
      delete.retention.ms: 86400000
      min.compaction.lag.ms: 21600000
      retention.ms: 0
      min.insync.replicas: 1
  ```
 
1. Login to the tools pod and verify that the newly added topic is successfully created on Kafka

  ```
  kubectl exec --tty -i kafka-client --namespace kafka -- bash
  
  # Verify Topics
  kafka-topics.sh --bootstrap-server kafka-0.kafka-headless.kafka.svc.cluster.local:9092  --describe
  ```
  
### Install Action Runners with Horizontal Scaling

1. In the previous section number of action runners deployed are fixed to 2. If there are more than 2 Github Actions are triggered the jobs will go in waiting state until any of the runner becomes available and picks up the job. But, there is a way to spin up more workers dynamically based on your requirement.

2. The horizontal scaling can be achieved by deploying `HorizontalRunnerAutoscaler`. You can find the configuration in the file [scaling-repo-action-runner.yaml](./scaling-repo-action-runner.yaml).

3. Important difference is previously the replica was specified as 2 however in new configuration we are setting up `minReplicas` and `maxReplicas`. The Horizontal Scaling can be acheived in two ways:

   1. **Pull Based**: The action controller periodically polls (default: 10 min) the number of queued jobs from Github and scale up/down the number of runners. This is simple technique but scaling will not happen in real-time. Also, lowering the polling period can run into Github API Rate Limiting issue.
   2. **Webhook Based**: In this approach you deploy the API Server in the k8s which is registered in the Github to receive events. For any new job created Github will send event to API Server and based on event action controller will scrale up the runners. With this you can achieve scaling in realtime but this approach is slight complex to implement. 
   3. For the sake of this POC we are using Pull Based approach.

4. Before enabling the Horizontal Scaling adjust the polling time by editign deployment. Open the deployment using below comamnd and adjust the `--sync-period` parameter.

  ```
  kubectl edit deployment actions-runner-controller -n actions-runner-system
  ```
5. Deploy the `HorizontalRunnerAutoscaler`

  ```
  kubectl --namespace action-runners apply -f scaling-repo-action-runner.yaml
  ```
6. With the deployment of `HorizontalRunnerAutoscaler` now the runner will be scaled down from 2 to 1. As soon as first Github Action triggers first job it will be picked up by initial runner. If there are more jobs in queue Action Controller will start scaling up the runners till it reaches to 4 which is `maxReplicas`.

