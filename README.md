# Flink operator on AWS

Managing Apache Flink workloads using Kubernetes operators on AWS.

## Prerequisites

The following must be installed in your local system:

- [aws cli](https://docs.aws.amazon.com/cli/latest/userguide/getting-started-install.html)
- [docker](https://docs.docker.com/get-docker/)
- [kubernetes](https://kubernetes.io/docs/tasks/tools/)
- [helm](https://helm.sh/docs/intro/install/)
- [terraform](https://developer.hashicorp.com/terraform/tutorials/aws-get-started/install-cli)

The AWS CLI must also be [configured](https://docs.aws.amazon.com/cli/latest/userguide/cli-chap-configure.html) with sufficent permissions to allow Terraform to provision and destroy resources.

## Kubernetes Cluster Provisioning using Terraform

Provision the EKS cluster and networking infrastructure:
```terraform init```
```terraform apply --auto-approve```

Configure kubectl to interact with the EKS output. The command inputs are retrieved from the terraform outputs.
```aws eks --region $(terraform output -raw region) update-kubeconfig --name $(terraform output -raw cluster_name)```

Can verify the worker nodes in the cluster:
```kubectl get nodes```

## Flink Operator Deployment

Create the namespace and roles for running the Flink operator.
```kubectl create -f k8s/namespace.yaml```
```kubectl create -f k8s/role.yaml```
```kubectl create -f k8s/bind-role.yaml```

Install the certificate manager on EKS cluster. 
>If the installation fails you can pass the ```--set webhook.create=false``` to the helm install command below.

```kubectl create -f https://github.com/jetstack/cert-manager/releases/download/v1.8.2/cert-manager.yaml```

Use the following Helm chart to install the Flink Kubernetes Operator.
```helm repo add flink-operator-repo https://downloads.apache.org/flink/flink-kubernetes-operator-1.6.0/```
```helm install flink-operator flink-operator-repo/flink-kubernetes-operator```

Once the operator is running you can submit Flink jobs:
```kubectl create -f k8s/flink-job-1.yaml```
> Note: flink-job-2.yaml file needs an env var passed to it to work due to a slight modification of the manifest detailed below

You can follow the logs of the sample Flink job they should look similar to this after a successful startup.
>```kubectl logs deploy/sample-job-1 -f```
>2023-10-20 06:20:38,679 INFO  org.apache.flink.runtime.checkpoint.CheckpointCoordinator    [] - Triggering checkpoint 52 (type=CheckpointType{name='Checkpoint', sharingFilesStrategy=FORWARD_BACKWARD}) @ 1697782838679 for job 2ee7024cd9cdf620bd14ba6a11dd07e7.
>2023-10-20 06:20:38,693 INFO  org.apache.flink.runtime.checkpoint CheckpointCoordinator    [] - Completed checkpoint 52 for job 2ee7024cd9cdf620bd14ba6a11dd07e7 (15387 bytes, checkpointDuration=14 ms, finalizationTime=0 ms).
>2023-10-20 06:20:40,679 INFO  org.apache.flink.runtime.checkpoint.CheckpointCoordinator    [] - Triggering checkpoint 53 (type=CheckpointType{name='Checkpoint', sharingFilesStrategy=FORWARD_BACKWARD}) @ 1697782840679 for job 2ee7024cd9cdf620bd14ba6a11dd07e7.
>2023-10-20 06:20:40,695 INFO  org.apache.flink.runtime.checkpoint.CheckpointCoordinator    [] - Completed checkpoint 53 for job 2ee7024cd9cdf620bd14ba6a11dd07e7 (15270 bytes, checkpointDuration=16 ms, finalizationTime=0 ms).

## Resource Management

###### Configure resource constraints for the Flink operator and jobs

##### Briefly describe how you would handle resource quotas and limits for Flink jobs

## Logging and Monitoring

###### Suggest tools or practices for monitoring the health, performance and logs of the Flink jobs running in the cluster

## Deployment & Management

###### Describe how would you deploy and track updates to Flink jobs
Flink jobs will be stored on the image and more can be added by pushing a newly build image to a image repo. Updates can also be tracked there.

We could use a single flink-job.yaml file to handle different job types by using the command below. The .jar file would need to be built into the image before hand.
```export JOB_URI=local:///opt/flink/examples/streaming/StateMachineExample.jar```
```cat k8s/flink-job-2.yaml | sed "s~{{JOB_URI}}~$JOB_URI~g" | kubectl apply -f -```
Resource management of the taskManager and jobManager can also be allocated this way.

We would need to create a green copy and blue copy of our kubernetes manifests.
Efficient Blue-Green deployment would also require deployment of a loadbalancer to route traffic between the blue deployments and green deployments.

We can track deployment versions with git.

## Cleanup

Teardown resources created in AWS
```terraform destroy --auto-approve```
