# Kubernetes Orchestration Engine Training
This tutorial provides a fast training on selected features of Kubernetes orchestration engine. It allows users to deploy a Kubernetes cluster upon AWS using their account and provides the necessary steps to discover basic and advanced functionalities of Kubernetes. It goes through the activation and usage of features such as dashboard, monitoring, advanced scheduling, execution of Big Data jobs with Spark, cpu management and pod auto-scaling.

## Prerequisites

- An AWS account and IAM user, with `AmazonEC2FullAccess` and `IAMReadOnlyAccess`.
- [AWS CLI](https://aws.amazon.com/cli/) installed
- [Parallel SSH](https://code.google.com/archive/p/parallel-ssh/) installed for the configuration script
  

## 1. Deploy the AWS VMs to host our Kubernetes cluster

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

At the end of the procedure you will get a list of IPs. These are the external IP addresses that you can use to connect on the VMs. Save them on a file such as `KubeTraining_Nodes.txt` because we will need them afterwards.

The last one will be the master of your Kubernetes cluster, named NODE1_IP for simplicity. Connect on this last NODE1_IP with the following command:

    $ ssh ubuntu@NODE1_IP

#### Teardown the AWS deployment

You can stop the VMs and deployment at any time using the following command:

    $ ./workshopctl stop $TAG

## 2. Activate the Dashboard with Heapster and Prometheus monitoring with Grafana on your Kubernetes cluster

From now on all the commands that are mentioned have to be executed on NODE1 of your AWS deployment (which will be the master of your Kubernetes cluster) except if another node is mentioned explicitly.

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

## 3. Execute Big Data job with Spark on the Kubernetes Cluster

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

## 4. Activate an advanced scheduling policy and test its usage

We test a particular scheduling policy that considers the temperature of the nodes and performs the placement on the node with the lowest temperature. We test the policy explained and provided by Nerdalize. More details on this [presentation](https://schd.ws/hosted_files/kccnceu18/4e/KubeCon%202018%20-%20Advanced%20Scheduling%20for%20Heating%20Shows.pdf). 

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

Find the pod that has been deployed for the new scheduling policy: 

    $kubectl get pods --all-namespaces
    $kubectl describe pods heat-scheduler-SPECIFIC-HASH -n kube-system

Launch a job that will use the new scheduler:
    
    $kubectl create -f deployments/podpi.yaml

Check on which node it is executed by following the logs of the scheduler extender:

    $kubectl logs heat-scheduler-SPECIFIC-HASH -n kube-system -c extender

Adapt the temperature of nodes:

    $kubectl annotate --overwrite nodes node2 nerdalize/temp=10

Launch a new job. We can launch the same file as before but before launching we need to change the name of the pod to avoid getting a submission error:

    $sed -e "s/podpi9/podpi10/g" -i deployments/podpi.yaml
    $kubectl create -f deployments/podpi.yaml

Follow the scheduling decision by getting the logs of the scheduler extender as before:

    $kubectl logs heat-scheduler-SPECIFIC-HASH -n kube-system -c extender

## 5. Activate and use Exclusive CPU Management

From another terminal connect on one of the compute nodes of your Kubernetes cluster and activate `cpu-manager-policy` parameter on kubelet.

    $KUBEADM_SYSTEMD_CONF=/etc/systemd/system/kubelet.service.d/10-kubeadm.conf
    $sudo sed -e "s/--allow-privileged=true/--allow-privileged=true --cpu-manager-policy=static --kube-reserved=cpu=400m/" -i $KUBEADM_SYSTEMD_CONF
    $sudo systemctl stop kubelet
    $rm /var/lib/kubelet/cpu_manager_state
    $sudo systemctl daemon-reload
    $sudo systemctl start kubelet

Launch a job and see how cgroups are set in a way to allow exclusive reservation of the job on the allocated CPU.

    $kubectl create -f ~/heat-scheduler/deployments/podpi.yaml

## 6. Enable and use Pod Autoscaling

This exercise will provide the necessary steps to configurie HPA v2 for Kubernetes, use it and observe its behaviour. The following commands allow us ot install the Metrics Server add-on that supplies the core metrics and use a demo app to showcase pod autoscaling based on CPU and memory usage.

    $git clone https://github.com/stefanprodan/k8s-prom-hpa
    $cd k8s-prom-hpa/

Deploy the Metrics Server in the kube-system namespace:

    $kubectl create -f ./metrics-server

Deploy podinfo to the default namespace:

    $kubectl create -f ./podinfo/podinfo-svc.yaml,./podinfo/podinfo-dep.yaml

Next define a HPA that maintains a minimum of two replicas and scales up to ten if the CPU average is over 80% or if the memory goes over 200Mi by submitting the yaml file:

    $kubectl create -f ./podinfo/podinfo-hpa.yaml

After a couple of seconds the HPA controller contacts the metrics server and then fetches the CPU and memory usage. You can see the evolution of HPA through the following command:

    $kubectl get hpa

In order to increase the CPU usage, run a load test with rakyll/hey:

    $cd ~/
    $mkdir golang
    $export GOPATH=~/golang/
    $export GOROOT=/usr/lib/go-1.10
    $export PATH=$GOPATH/bin:$GOROOT/bin:$PATH
    $go get -u github.com/rakyll/hey
    $hey -n 10000 -q 10 -c 5 http://NODE1_IP:31198/

You can monitor the HPA events with:

    $kubectl describe hpa

The autoscaler doesn't react immediately to usage spikes. By default the metrics sync happens once every 30 seconds. Also scaling up/down can only happen if there was no rescaling within the last 3-5 minutes. This ensures the HPA prevents rapid execution of conflicting decisions and gives time for the Cluster Autoscaler to kick in.

The procedure is explained in detail and with more examples here: https://www.weave.works/blog/kubernetes-horizontal-pod-autoscaler-and-prometheus





