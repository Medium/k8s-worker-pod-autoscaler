# Worker Pod Autoscaler

Scale kubernetes pods based on the Queue length of a queue in a Message Queueing Service. Worker Pod Autoscaler automatically scales the number of pods in a deployment based on observed queue length.

Currently the supported Message Queueing Services are AWS SQS and Beanstalk.

# Install the WorkerPodAutoscaler

### Install
Running the below script will create the WPA [CRD](https://kubernetes.io/docs/concepts/extend-kubernetes/api-extension/custom-resources/) and start the controller. The controller watches over all the specified queues in AWS SQS and beanstalk and scales the Kubernetes deployments based on the specification.

```bash
export AWS_ACCESS_KEY_ID='sample-aws-access-key-id'
export AWS_SECRET_ACCESS_KEY='sample-aws-secret-acesss-key'
./hack/install.sh
```

### Verify Installation
Check the wpa resource is accessible using kubectl

```bash
kubectl get wpa
```

# Example
Please install following above before trying the below example.

- Create Deployment that needs to scale based on queue length.
```bash
kubectl create -f artificats/example-deployment.yaml
```

- Create `WPA object (example-wpa)` that will scale the `example-deployment` based on SQS queue length.
```bash
kubectl create -f artifacts/example-wpa.yaml
```

This will start scaling `example-deployment` based on SQS queue length.

# Why make a separate autoscaler CRD ?

Kubernetes does support custom metric scaling using Horizontal Pod Autoscaler. Before making this we were using HPA to scale our worker pods. Below are the reasons for moving away from HPA and making a custom resource:

**TLDR;** Don't want to write and maintain custom metric exporters? Use WPA to quickly start scaling your pods based on queue length with minimum effort (few kubectl commands and you are done !)

1. **No need to write and maintain custom metric exporters**: In case of HPA with custom metrics, the users need to write and maintain the custom metric exporters. This makes sense for HPA to support all kinds of use cases. WPA comes with queue metric exporters(pollers) integrated and the whole setup can start working with 2 kubectl commands.

2. **Different Metrics for Scaling Up and Down**: Scaling up and down metric can be different based on the use case. For example in our case we want to scale up based on SQS `ApproximateNumberOfMessages` length and scale down based on `NumberOfEmptyReceives`. This is because if the worker jobs watching the queue is consuming the queue very fast, `ApproximateNumberOfMessages` would always be zero and you don't want to scale down to 0 in such cases.

3. **Fast Scaling**: We wanted to achieve super fast near real time scaling. As soon as a job comes in queue the containers should scale if needed. The concurrency, speed and interval of sync have been made configurable to keep the API calls to minimum.

4. **On-demand Workers:** min=0 is supported. It's also supported in HPA.

## Configuration

## WPA Resource

```yaml
apiVersion: k8s.practo.dev/v1alpha1
kind: WorkerPodAutoScaler
metadata:
  name: example-wpa
spec:
  minReplicas: 0
  maxReplicas: 10
  targetMessagesPerWorker: 2
  queueURI: https://sqs.ap-south-1.amazonaws.com/{{aws_account_id}}/{{queue_prefix-queue_name-queue_suffix}}
  deploymentName: example-deployment
```

## WPA Controller

```bash
$ bin/darwin_amd64/workerpodautoscaler run --help
Run the workerpodautoscaler

Usage:
  workerpodautoscaler run [flags]

Examples:
  workerpodautoscaler run

Flags:
      --aws-region string             aws region of SQS (default "ap-south-1")
  -h, --help                          help for run
      --kube-config string            path of the kube config file, if not specified in cluster config is used
      --resync-period int             sync period for the worker pod autoscaler (default 20)
      --sqs-long-poll-interval int    the duration (in seconds) for which the sqs receive message call waits for a message to arrive (default 20)
      --sqs-short-poll-interval int   the duration (in seconds) after which the next sqs api call is made to fetch the queue length (default 20)
      --wpa-threads int               wpa threadiness, number of threads to process wpa resources (default 10)
```
