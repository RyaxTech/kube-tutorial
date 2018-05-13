# Kubernetes Orchestration Engine Training
This tutorial provides a fast training on selected features of Kubernetes orchestration engine. It allows users to deploy a Kubernetes cluster upon AWS using their account and provides the necessary steps to discover basic and advanced functionalities of Kubernetes. It goes through the activation and usage of features such as dashboard, monitoring, advanced scheduling, execution of Big Data jobs with Spark, cpu management and pod auto-scaling.

## Prerequisites

- An AWS account and IAM user, with `AmazonEC2FullAccess` and `IAMReadOnlyAccess`.
- [AWS CLI](https://aws.amazon.com/cli/) installed
- [Parallel SSH](https://code.google.com/archive/p/parallel-ssh/) installed for the configuration script
  

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
By default the script and the following steps will deploy 4 VMs of AWS t2.medium class with Ubuntu succeeded by a deployment of Kubernetes cluster with 1 master and 3 compute nodes. For customization you can change `settings/kube101.yaml` and `lib/commands.sh`.  


### Run the script to bring up the AWS VMs and install Docker 

    $ ./workshopctl start 4
    $ ./workshopctl deploy $TAG settings/kube101.yaml

### Run the script to deploy Kubernetes on the VMs and prepare the environment for the training
 
    $ ./workshopctl kube $TAG

At the end of the procedure you will get a list of IPs. These are the external IP addresses that you can use to connect on the VMs. You can save them because we will need them afterwards. The last one will be the master of your Kubernetes cluster, named NODE1_IP for simplicity. Connect on this last NODE1_IP with the following command:

    $ ssh ubuntu@NODE1_IP

#### Teardown the VMs

You can stop the VMs and deployment at any time using the following command:

    $ ./workshopctl stop $TAG

## Activate the Dashboard with Heapster and Prometheus monitoring with Grafana on your Kubernetes cluster

### Kubernetes Dashboard with Heapster

    $kubectl apply -f https://raw.githubusercontent.com/kubernetes/dashboard/master/src/deploy/alternative/kubernetes-dashboard.yaml

Then expose the service to access it through an external browser. Change 'type: ClusterIP' to 'type: NodePort' and save

    $kubectl -n kube-system edit service kubernetes-dashboard

Grant cluster-admin permissions to Dashboard
    
    $kubectl create clusterrolebinding add-on-cluster-admin --clusterrole=cluster-admin --serviceaccount=kube-system:kubernetes-dashboard

Install Heapster for resources monitoring
    
    $git clone https://github.com/kubernetes/heapster.git
    $cd heapster/deploy/kube-config/influxdb
    $kubectl create -f influxdb.yaml
    $kubectl create -f heapster.yaml
    $cd ../rbac/
    $kubectl create -f heapster-rbac.yaml
    $kubectl delete pod kubernetes-dashboard -n kube-system

Find out the exposed port where dashboard is running, named DASHBOARD_PORT for simplicity.
    
    $kubectl -n kube-system get service kubernetes-dashboard

Access Dashboard from your local browser through the link - "http://NODE1_IP:DASHBOARD_PORT".

### Prometheus monitoring with Grafana

    $git clone https://github.com/RyaxTech/prometheus-operator.git
    $cd prometheus-operator/contrib/kube-prometheus/
    $git checkout release-0.19
    $cd contrib/kube-prometheus/
    $./hack/cluster-monitoring/deploy

Find out the exposed port where dashboard is running, named GRAFANA_PORT for simplicity.

    $kubectl get svc grafana -n monitoring

Access Grafana from your local browser through the link - "http://NODE1_IP:GRAFANA_PORT".

## Execute Big Data job with Spark on the Kubernetes Cluster

Retrieve the API-server IP address:
    
    $kubectl cluster-info

Execute the spark job through the following command subsituting the $API_SERVER variable with the IP address collected from the previous step:

    $cd ~/spark-2.3.0-bin-hadoop2.7/
    $bin/spark-submit --master k8s://https://$API_SERVER:6443 --deploy-mode cluster --name spark-pi --class org.apache.spark.examples.SparkPi --conf spark.kubernetes.authenticate.driver.serviceAccountName=spark --conf spark.executor.instances=2 --conf spark.kubernetes.container.image=gohnwej/spark:2.3.0 --conf spark.kubernetes.driver.pod.name=spark-pi-driver1 local:///opt/spark/examples/jars/spark-examples_2.11-2.3.0.jar 10000

While the job is running you can connect from another terminal and follow the execution of the job. You can vary the time of execution by increasing the last variable set to 10000 for the example
    
    $kubectl get pods -n default
    $kubectl describe pods spark-pi-driver1 -n default

Note that the driver launches the executors and each one of them is launched on a different node
You can retreve the result of the execution through the command that gives the logs of the pod:
    
    $kubectl logs spark-pi-driver3 -n default  

## Activate an advanced scheduling policy and test its usage

Clone the repo:

    $cd ~/
    $git clone https://github.com/RyaxTech/heat-scheduler.git
    $cd ~/heat-scheduler/

Prepare the nodes with annotations:

    $kubectl annotate nodes node1 nerdalize/temp=40
    $kubectl annotate nodes node2 nerdalize/temp=30
    $kubectl annotate nodes node3 nerdalize/temp=35
    $kubectl annotate nodes node4 nerdalize/temp=20

Give admin permissions to the default serviceaccount:

    $kubectl create clusterrolebinding serviceaccounts-cluster-admin --clusterrole=cluster-admin --serviceaccount=kube-system:default

Deploy the new scheduler:
        
    $kubectl create -f deployments/scheduler-policy-config.yaml
    $kubectl create -f deployments/heat-scheduler.yaml

Launch a job that will use the new scheduler:
    
    $kubectl create -f deployments/podpi.yaml

Check on which node it is executed by following the logs of the scheduler:

    $kubectl logs

## Activate and use Exclusive CPU allocation policy

Connect on one of the nodes and activate `cpu-manager-policy` parameter on kubelet

    $KUBEADM_SYSTEMD_CONF=/etc/systemd/system/kubelet.service.d/10-kubeadm.conf
    $sudo sed -e "s/--allow-privileged=true/--allow-privileged=true --cpu-manager-policy=static --kube-reserved=cpu=400m/" -i $KUBEADM_SYSTEMD_CONF
    $sudo systemctl stop kubelet
    $rm /var/lib/kubelet/cpu_manager_state
    $sudo systemctl daemon-reload
    $sudo systemctl start kubelet

Launch a job and see how cgroups are set in a way to allow exclusive reservation of the job on the allocated CPU.

    $kubectl create -f ~/heat-scheduler/deployments/podpi.yaml

## Enable and use Pod Autoscaling

    $cd ~/
    $export GOROOT=/usr/lib/go-1.10
    $git clone https://github.com/stefanprodan/k8s-prom-hpa
    $cd k8s-prom-hpa/
    $kubectl create -f ./metrics-server
    $kubectl create -f ./podinfo/podinfo-svc.yaml,./podinfo/podinfo-dep.yaml
    $kubectl create -f ./podinfo/podinfo-hpa.yaml
    $export PATH=$GOPATH/bin:$GOROOT/bin:$PATH
    $go get -u github.com/rakyll/hey
    $hey -n 10000 -q 10 -c 5 http://external-IPnode1:31198/








