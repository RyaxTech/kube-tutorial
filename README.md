# Kubernetes Orchestration Engine Training
This tutorial provides a fast training on selected features of Kubernetes orchestration engine. It allows users to deploy a Kubernetes cluster upon AWS using their account and provides the necessary steps to discover basic and advanced functionalities of Kubernetes. It goes through the activation and usage of features such as dashboard, monitoring, advanced scheduling, execution of Big Data jobs with Spark, cpu management and pod auto-scaling.

#### Prerequisites

- An AWS account and IAM user, with `AmazonEC2FullAccess` and `IAMReadOnlyAccess`.
- The [AWS CLI](https://aws.amazon.com/cli/) installed
- [Parallel SSH](https://code.google.com/archive/p/parallel-ssh/) for the configuration script
  

## Deploy the AWS VMs to host our Kubernetes cluster

We will be using the tools provided by another widely used Kubernetes training platform which we have adapted to our needs.

### Clone the repo and prepare to run the deployment script

    $ mkdir kubernetes_training
    $ cd kubernetes_training
    $ git clone https://github.com/RyaxTech/container.training
    $ cd container.training/prepare-vms

Set the following required Environment Variables based on your AWS key along with the default region.

```
export AWS_ACCESS_KEY_ID="$YOUR_AWS_KEY_ID"
export AWS_SECRET_ACCESS_KEY="$YOUR_AWS_KEY_ID"
export AWS_DEFAULT_REGION="us-east-1"
```

#### Details on deployment

For more details on the deployment script you can check here: https://github.com/RyaxTech/container.training/tree/master/prepare-vms 
For customization you can change `settings/kube101.yaml` and `lib/commands.sh`. By default the script and the following steps will deploy 4 VMs of AWS t2.medium class with Ubuntu succeeded by a deployment of Kubernetes cluster with 1 master and 3 compute nodes.  


### Run the script to deploy the VMs

    $ ./workshopctl start 4
    $ ./workshopctl $TAG settings/kube101.yaml
    $ ./workshopctl deploy $TAG

### Run the script to deploy Kubernetes on the VMs
 
    $ ./workshopctl kube $TAG


