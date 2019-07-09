MY Experiment with Kubernetes:
===============================
===============================


Docker Installation and Overview
=================================
Docker is a containerization technology that allows you to package and run 
applications in loosely isolated environments called containers. For our 
Kubernetes cluster, we will be using Docker as our container runtime. In this
lesson, we will go through a brief overview of Docker and then install
Docker CE 18.06.1 to our cluster nodes.

Installation Steps:
Add GPG key:

# curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add -

Add Docker repository:

# sudo add-apt-repository "deb [arch=amd64] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable"

Update packages:

# sudo apt-get update

Install Docker:

# sudo apt-get install -y docker-ce=18.06.1~ce~3-0~ubuntu


Installing Kubeadm, Kubelet, and Kubectl
===========================================
**Important: Due to an issue introduced by a recent security update, the 
kubernetes binaries need to use version 1.12.7-00 instead of 1.12.2-00. Please 
see instalation instructions below.

Kubeadm, kubelet, and kubectl are important components for our implementation 
of Kubernetes. Kubeadm provides a streamlined way to bootstrap a kubernetes 
cluster. Kubelet is the agent that runs on the cluster nodes and performs 
various actions on the components of the cluster. Then we have kubectl, which
is the command line utility for interacting and managing the cluster. 

Installation Instructions to be exexucted on all the master and cluster nodes:
------------------------------------------------------------------------------
Add the GPG key:
# curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo apt-key add -

Add the Kubernetes repository:
# cat << EOF | sudo tee /etc/apt/sources.list.d/kubernetes.list
deb https://apt.kubernetes.io/ kubernetes-xenial main
EOF

Resynchronize the package index:
# sudo apt-get update

Install the required packages:
# sudo apt-get install -y kubelet=1.12.7-00 kubeadm=1.12.7-00 kubectl=1.12.7-00

Prevent packages from being automatically updated:
# sudo apt-mark hold kubelet kubeadm kubectl


Bootstrap the Kubernetes Cluster
=================================
The kubeadm command was developed to provide best practices for initializing a
Kubernetes cluster and for joining nodes to the cluster. In this lesson, we 
will be using the kubeadm command to bootstrap our kubernetes cluster and then
join our cloud servers to the cluster.

Installation Steps:
-------------------
Initialize the Cluster on the Master:
# sudo kubeadm init --pod-network-cidr=10.244.0.0/16

Set up kubeconfig for a Local User on the Master
# mkdir -p $HOME/.kube

# sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config

# sudo chown $(id -u):$(id -g) $HOME/.kube/config

Join Nodes to the Cluster
# sudo kubeadm join $controller_private_ip:6443 --token $token --discovery-token-ca-cert-hash $hash



Configure Cluster Network with Flannel
=======================================
In Kubernetes, the communication between pods occurs on the cluster network. 
To set up the cluster network, install the network add-on after bootstrapping
the cluster. In this lesson, we will prepare the cluster nodes for the cluster
network and then install the Flannel network add on.

Install the Flannel Network Addon

(on all nodes) Add net.bridge.bridge-nf-call-iptables=1 to sysctl.conf.
# echo "net.bridge.bridge-nf-call-iptables=1" | sudo tee -a /etc/sysctl.conf

(on all nodes) Apply the change made to sysctl.conf
# sudo sysctl -p

(on Master) Use kubectl to install Flannel using YAML template.
# kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/bc79dd1505b0c8681ece4de4c0d86c5cd2643275/Documentation/kube-flannel.yml


Installing a Single Application to the Cluster:
===============================================
Understanding Kubernetes Pods
-----------------------------
. A pod is the basic building block of Kubernetes
. Pods are the smallest units that can be created or deployed in the Kubernetes object model.
. Pods normally have one container per pod, but there can be more.
. Each pod has its own unique IP address an storage resources.
. Kubernetes schedules pods to run on servers in the cluster.

Understanding Namespaces:
------------------------
- A namespace is a virtual cluster that is backed by the physical Kubernetes cluster.
- Namespaces provide a logical way to split up cluster resources among multiple users.
- Kubernetes initially starts with three namespaces.
  . default - The default namespace for all objects that do not have a namespace.
  . kube-system - The namespace for objects that were created by the kubernetes system.
  . kube-public - The namespace is created automatically and is readable by all users by default.
  

Working with Namespaces:
------------------------

1. Create a namespace:
  kubectl create namespace <namespace>
  
2. List namespaces:
  kubectl get namespaces
  
3. Get additional information about a namespace.
  kubectl describe namespaces <namespace>
  
4. Delete a namespace
  kubectl delete namespace <namespace>
  
  
Working with Pods
================
1. Create a Pod
  a) Using kubectl run (deprecated), However this will create a deployment 
     kubectl run <name> --image=<image>
	 
	 E.g kubectl run test --image=nginx 
This will create a deployment object under which a test pod is created
like deployment.apps/test under the default namespace, as we have not passed 
any namespace.
	 
	 To verify pods are created:
	 kubectl get pods
	 
  b) Using kubectl create deployment:
  kubectl create deployment <name> --image=<image>
  
  E.g 
  kubectl create deployment test2 --image=nginx
  
  kubectl get pods
  This will return pod name starting with test2 following with unique identifier.
  
  
  c) Using kubectl create with STDIN:
  cat << EOF | kubectl create -f -
  > apiVersion: v1
  > kind: Pod
  > metadata:
  >   name: nginx
  > spec:
  >   containers:
  >   - name: nginx
  >     image: nginx
  > EOF
  
2. Get additional information about Pod:
  kubectl describe pod <pod_name>
  
3. Delete a Pod:
  kubectl delete pod <pod_name>
  
Note: If a deployment was created for the pod, the deployment must be deleted in order to remove the pod,
otherwise, if we try to delete a pod without deleting the deployment, then a new pod will be automatically
generated, within the deployment.

Installing a Microservice Application to the Cluster.
=====================================================
This microservice is made available on GitHub by WeaveWorks. It is created to show a use case for the microservice architecture and be a reference for anyone trying to learn or implement a microservice architecture.

Install Git and Clone the Microservice Repository
-------------------------------------------------
1. Install Git:
sudo apt-get install git

2. Switch to the user's home directory
cd ~

3. Clone the Microservice repo:
git clone https://github.com/linuxacademy/microservices-demo

Install the Microservice Application to the Cluster
---------------------------------------------------
1. Create a namespace for the application:
kubectl create namespace sock-shop

2. Install the microservice application under the "sock-shop" namespace.
kubectl -n sock-shop create -f microservices-demo/deploy/kubernetes/complete-demo.yaml

3. List the pods for the newly created application
kubectl get pods -n sock-shop

Note: Using -w allows you to see view the pods as they start up in the real time:
kubectl get pods -n sock-shop -w

Working with the Microservice Application
------------------------------------------
Access 	the microservice application
1. List the services in the application:
  kubectl get services -n sock-shop
  
2. Get more information about the NodePort Service:
  kubectl services -n sock-shop front-end
  
3. Access the application using the IP of one of the cluster nodes and the port from the NodePort service.
  http://<IP ADDRESS>:30080

4. Create an account an login.
5. Browse products.
6. Add products to cart
7. View cart and checkout.

Each of these individual components are managed by a separate service and can be updated, deployed, and scaled independently.

Kubernetes takes ab otherwise very complex application and makes the management of it more simple and straightforward.


Kubernetes API
==============
The Kubernetes API is the main gateway for interacting with all the components in
the cluster. This a RESTful API that supports interactions provided through HTTP
verbs viz GET, POST, DELETE, PUT, PATCH

Of the three components on the master that make up the control plane- the API
Server. The Controller Manager, and the Scheduler - the API Server is the only
component that directly interacts with the distributed key-value store, etcd.

The Kubernetes API is exposed by the kube-apiserver, which runs on the master. This serves as the frontend for the Kubernetes control plane.

API terminology
------------------
- Resource Type: The name used in the URL(pods,services,etc).
- Kind: The concrete representation in JSON of a resource type:
  - Objects - Represents a a persistent entity in the system.
  - Lists - Collections of resources of one or more kinds.
  - Simple - used for Specific actions on objects and non-persistent entities.

- Collection: A list of instances of a resource type.
- Resource: A single instance of a resource type.

Note: All resource types are scoped by the cluster or a namespace.
  

HTTP Verbs for RESTful Resources:
----------------------------------
- GET : Retrieve a resource or list of resources.
- POST: Create a new resource.
- DELETE: Delete a resource.
- PUT: Mainly used to update a resource but can be used to create.
- PATCH: Selectively modify the specified fields of a resource.


Interacting with the API:
--------------------------
Using command-line tools such kubectl and kubeadm.
 Example: kubectl get pods --all-namespaces.
 
Calling the API directly using REST calls.
  - With kubectl proxy - Run 
    
	kubectl proxy --port=8080 & 
  
  then use curl, wget or a browser to interact with the API:
 curl http://localhost:8080/api/

  - Without kubectl proxy- Use 
  
  kubectl describe secret
   
   to get the token for the default service account and make a call to the API server passing the token.
Example: curl $APISERVER/api --header "Authorization: Bearer $TOKEN" --insecure 



Service Discovery:
===================
Service discovery is the process of automatically detecting available services and
how to connect to them. 

The two main modes of service discovery in Kubernetes are using environment
variables or using DNS:
- A set of environment variables will be assigned by kubelet whenever a Pod is run
on a node. In order to function properly the host and port variables must be set (at
a minimum). Environment variables can be displayed by running.

kubectl get pods -n sock-shop ===> Get any pod and use it below.

kubectl exec <pod> -n sock-shop -- env

- DNS is the default means of service discovery in Kubernetes and is highly
recommended. The DNS server is a cluster add-on that watches the Kubernetes API for
any new Services and automatically creates DNS records for them. Enabling DNS across
the cluster allows all the Pods to perform name resolution automatically for all Services.

Staring in Kubernetes version 1.13 and later CoreDNS is used by defualt, replacing the previous DNS Server, kube-dns.


CoreDNS: CoreDNS is an incredibly flexible cluster DNS server. It integrates with
Kubernetes providing DNS and service discovery. By modifying the Corefile, CoreDNS
can support more use cases than kube-dns which is one of the reasons that it has
become the default DNS server for Kubernetes.

The Corefile is kept in a configmap and can be updated to add plugins which provie new functionality for CoreDNS and change the way service discovery works:

-- Display current configmaps.
  kubectl get configmap --all-namespaces.
-- View CoreDNS configmap:
  kubectl describe configmap coredns -n kube-system
-- Display current deployments:
  kubectl get deployments --all-namespaces
-- View coredns deployment:
  kubectl describe deployments -n kube-system coredns

Note: The coredns configmap is referenced in the coredns deployment when it is
installed. The Core file in the configmap can be updated to modify the behaviour of
CoreDNS.

Querying the Cluster DNS:
--------------------------
1. Install a Pod running curl to query other Pods in the cluster, in a default namespace(so no -n option provided):
  kubectl run curl --image=radial/busyboxpluz:curl -i --tty 
  
2. Attach to running curlPod:
  kubectl attach <name of pod> -id

3. Querying Services from curlPod
  - Services in the same namespace
    curl <service>
  - Services in a different namespace:
    curl <service>.<namespace>
	
	Even nslookup <service>.<namespace> 
	here: nslookup front-end.sock-shop will give the name resolution
	
For Leaving the Pod terminal give the following command, "Ctrl + P and Ctrl + Q." But if 'exit' command is executed then the new Pod is regenerated.

Note: All services within the clusetr are given A records by the DNS server. pods
will only be assigned A records if the hostname is set for the Pod. When deploying a
Pod, additional DNS configurations can be set (dnsConfig) such as the name server,
search domains, etc.


Replication
===========
Kubernetes uses replication to create multiple instances of an application 
across the cluster. One of the great features of Kubernetes is the ability to
replocate pods(and their underlying containers) across the cluster. Originally
in Kubernetes, the ReplicationController performed this action. ReplicaSets 
used in conjunction with Deployments have largely replaced this role.

ReplicationController
---------------------
The ReplicationController exists to ensure that the right amount of pods exits
across the cluster. So, if there are too many pods, the ReplicationController 
will terminate the excess number of pods. Conversely,if there are too few pods,
the ReplicationController will create pods until the defined amount is reached.
This is particulary important in the case of node failure or a service 
disruption. Even if the desired pod amount is one for given service, the 
ReplicationController will ensure that the pod is always available.

ReplicaSet
----------
ReplicationController and ReplicaSets are almost the same. The main way they differ is that
ReplicaSets support the new set-based selector requirements as oppose to ReplicationControllers,
which only support equality-based selector requirements. The use of ReplicaSets alone is 
discouraged in favour of using Deployments(which leverage ReplicaSet functionality).

Deployments
------------
Deployments have effectively replaced ReplicationControllers as the preferred method of
deploying and updating a set of Pods with replication(through ReplicaSets). The desired
amount of Pods (replicas) are defined in a deployment which leverages ReplicaSets to ensure
their creation. Deployments manage their own ReplicaSets, so there is no need to manage the
ReplicaSets outside of the deployment.


deployment.yaml
---------------
apiVersion: apps/v1
kind : Deployment
metadata:
  name: nginx-deployment
  labels:
    app: nginx
spec:
  replicas: 3
  selector:
    matchlabels:
	  app: nginx
  template:
    metadata:
	  labels:
	    app: nginx
    spec:
	  containers:
	  - name: nginx
	    image: nginx:1.15.4
		ports:
		- containerPort: 80


In order to create the deployment, execute the below command
kubectl create -f deployment.yaml 

Then look for the deployment in all the namespaces.
Since we have not created any namespaces, and assigned this nginx-deployment, so by default it will
be created under 'default' namespace.

kubectl get deployments --all-namespaces


kubectl get pods 
will result three pods running in the default namespace.


Ingress
=========
In reference to Kubernetes, Ingres is an API object that manages external access to services in the cluster typically through HTTP. Simply put this is the mechanism or the entry way by which the services in a private cluster are accessed. Ingress can be configured to provide services with externally-reachable URLs, load balanced 
traffic, SSL termination, and name-based virtual hosting.

Ingress is currently in beta resource and requires an ingres controller to function. The supported controllers as of this point are GCE and nginx. The type of controller that needs to be implemented will depend heavily on the platform being used. Please refer to the Kubernetes documentaion for specific implementations.

In real-world example, the main ingres will be provided by an edge router or a gateway from a cloud provider which will then route traffic to the ingres controller in the Kubernetes cluster. For our example, we will be using a NodePort service which will expose a static port on our cluster nodes which can be accessed outside of the
internal cluster IPs.

Services:
----------
 - View services: kubectl get services --all-namespaces
 - Describe services: kubectl describe services front-end -n sock-shop
 
Deployments:
-------------
 - View services: kubectl get deployments --all-namespaces
 - Describe services: kubectl describe deployments front-end -n sock-shop
 
Pods:
-----
 - View resources: kubectl get pods --all-namespaces
 - Describe resources: kubectl describe pods <pod-name> -n sock-shop
 
Accessing the Microservice Front End:
-------------------------------------
- http://clusterIp:<NodePort>
 - Simple elinks browser
   1. Install package: sudo apt-get install elinks
   2. Access front-end: elinks http://localhost:<NodePort>
   
 - Web browser with SSh tunnel
   1. Set up the SSH tunnel: ssh -L <any fancy port>:localhost:<NodePort> cloud_user@<IPAdddress of Master Node>
   2. Access front end url from browser on local machine.
     http://localhost:<same fancy port>
   
   
Scaling Microservices
=====================

Kubernetes provides the ability to scale the individual components of a microservice application independently. 
this is incredibly useful if one of the pods in your cluster requires more resources, or if it has more user
traffic than the other pods in your cluster. By individually scaling services, reosurces are not wasted on scaling
every service in the microservice application.

Manual Scaling:
---------------
1. List deployments:
  kubectl get deployments -n <namespace>
  
2. Get additional information about deployments:
  kubectl describe deployments <deployment> -n <namespace>
  
3. Increase the replicas in the deployment file and apply the changes.
  <editor> deployment.yml
  
  kubectl apply -f deployment.yml
  
4. List deployment and pods:

  kubectl get deployments -n <namespace>
  
5. Scale directly from the commandline

  kubectl scale deployment/<deployment> --replicas=3 -n <namespace>
  For e.g kubectl scale deployment/payment --replicas=3 -n sock-shop
  
6. List deployment and pods:
  kubectl get deployments -n <namespace>
  
  
The number of Pods can also be scaled down by reducing the amount
of replicas via the command line or the deployment file.

Autoscaling
------------
As mentioned previously, pods can be scaled manually by updating the number of replicas in a deployment,
replication controller, or replica set. Kubernetes also allows automatic scaling by implementing a Horizontal
Pod scaleror HPA for short. The HPA is implemented as a Kubernetes API resource and a controller which will
automatically scale the number of pods based on the observed CPU utilization(or other provided custom metrics).

To autoscale:

1. List the current deployments
  kubectl get deployments -n sock-shop
  
2. Create a horizontal Pod Autoscaler for a deployment:
  kubectl autoscale deployment <deployment> -n <namespace> --min 2 --max 6 --cpu-percent 65
  Ex. kubectl autoscale deployment front-end -n sock-shop --min 2 --max 6 --cpu-percent 65
  
3. List Horizontal Pod Autoscaler.
  kubectl get hpa -n <namespace>

4. Delete Horizontal pod Autoscaler.
  kubectl delete hpa -n <namespace> <hpa>
  E.g kubectl delete hpa -n sock-shop front-end
  
  
Vertical and Horizontal Scaling.
-------------------------------
Vertical scaling increases the resource limits on an individual pod.

Horizontal Scaling increases the number of pods across a cluster.



Self Healing:
=============
Kubernetes ensures that the desired state of the cluster and the actual state of the clsuter are always in sync.
This is made possible through continuous monitoring within the Kubernetes cluster. Whenever the state of a cluster 
changes from what has been defined, the various components of Kubernetes work to bring it back to its defined state. This type of automated recovery is often referred to as Self Healing.

Automatic Recovery
------------------
1. List deployments and pods
kubectl get deployments -n <namespace>
kubectl get pods -n <namespace>

2. Delete a pod
kubectl delete pod <pod_name> -n <namespace>

3. List deployments and pods
kubectl get deployments -n <namespace>
kubectl get pods -n <namespace>


Install application without replication
-----------------------------------------

1. Install application without deployment
  kubectl create -f application.yml

application.yml 
--------------
apiVersion: v1
kind: Pod
metadata:
  name: nginx
spec:
  containers:
  - name: nginx
    image: nginx
  
2. Delete pod
kubectl delete pod <pod_name> -n <namespace>

Note: Pods that are not created through a deployment
or other replication method (i.e ReplicationController or ReplicaSet)
will not automatically recover.

Node Failure:
-------------
1. List Nodes:
  kubectl get nodes

2. List deployments:
  kubectl get deployments -n <namespace>
  
3. List pods:
  kubectl get pods -n <namespace>
  
4. Gain additional information about pods, like on which cluster node it is running:
  kubectl describe pods -n <namespace> <pod_name>
  
5. Shutdown one of the cluster node, by executing the poweroff command.

5.a) In the master node check the "Status" of the cluster node.
   kubectl get nodes
   This should report the "Status" of the cluster node as "NotReady"

6. List pods and deployments:
  kubectl get pods -n <namespace>
  kubectl get deployments -n <namespace>
  
7. Start up cluster node.
7.a) In the master node check the "Status" of the cluster node.
   kubectl get nodes
   This should report the "Status" of the cluster node as "Ready"


8. List nodes, deployments and pods:
  kubectl get nodes
  kubectl get pods -n <namespace> ===> This will report one of the shutdown cluster node having the pod status
  as Unknown, but a new pod is created on one of the master or cluster node.
  kubectl get deployments -n <namespace>
