# My Experiment with Kubernetes Cluster
The basic requirement for craeting the Kubernetes cluster are:
Master Node should have atleast 2 Core CPU and 4 GB RAM
and Node should have atleast 1 core CPU and 4 GB RAM

On the master node execute the following commands
1) sudo kubeadm init --pod-network-cidr=[Calico CNI | Flannel CNI] --apiserver-advertise-address=[Master node IPAddress]
 
// For starting a Calico CNI(container network interface), with the network range for a particular pod 192.168.0.0/16 
and For starting a Flannel CNI the network range is: 10.244.0.0/16

Then execute the following set of commands as suggested by the output of the first command.

2) mkdir -p  $HOME/.kube
3) sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/configuration
4) sudo chown $(id -u):$(id -g) $HOME/.kube/config

5) For creating a POD based on Calico

 kubectl apply -f https://docs.projectcalico.org/v3.0/getting-started/kubernetes/installation/hosted/kubeadm/1.7/calico.yml

6) Execute the command to know whether your master node is up and running.
kubectl get nodes

kubectl get pods --all-namespaces -o wide

This will tell all the pods which are started by default.
like calico-etcd, kube-controller, etcd-master, dns server  

7) Bring up the dashboard before starting any nodes

kubectl create -f https://raw.githubusercontent.com/kubernetes/dashboard/master/src/deploy/recommended/kubernetes-dashboard.yaml

8) Dashboard once created need to be up and running and the command to bring the dashboard up and running is:

kubectl proxy

9) Now to access the dashboard GUI up Execute the folloing three steps
  a) create a service account using the following command
  
     kubectl create serviceaccount dashboard -n default
  
     Here -n is the namespace which is default.
  
  b) Then you have to make that account to be the admin user by using the clusterbinding with default namespace  dashboard and the command is:
  
     kubectl create clusterrolebinding dashboard-admin -n default --clusterrole=cluster-admin --serviceaccount=default:dashboard

  c) Get the secret key to be pasted into the dashboard token pod. Copy the outcoming secret key.
  
     kubectl get secret $(kubectl get serviceaccount dashboard -o jsonpath="{.secret[0].name}") -o jsonpath="{.data.token}" | base64 --decode

The 9 c) step will give you a token which you can use it in dashboard access.

10) Then navigate to this link

http://localhost:8001/api/v1/namespace/kub-system/services/https:kubernetes-dashboard:/proxy/#!login 

to acces the dashboard.

11) Then Open a terminal on the node to allow the node join to the cluster-admin
and the command to join was provided when the master node was initialized.
 And the command to be executed on the node is :
 
    sudo kubeadm join 192.168.56.101:6443 --token rsv7eq.f6uqods283j61ew3 --discovery-token-ca-cert-hash sha256:c5444e5cbd24149f70dff3e2d466f6057494c611904cb1d8e7e99c3b828fe62e

12) Now for deployment of any application, click on the Create button on the 
  right corner of the dashboard, select the "create an app" link, then provide 
  the necessary details. It is easy to create deployment via GUI.
  like 
     a) app name
     b) Container image
     c) No of pods
     d) Service which could be external for exposing to the localhost

13) On Master node to deploy an application through terminal
a) Create a deployment

    kubectl create deployment nginx --image=nginx

b) Verify the deployment

    kubectl get deployments

c) More details about the deployment

    kubectl describe deployment nginx

d) Create the service on the nodes

     kubectl create service nodeport niginx --tcp=80:80
e) Check which deployment is running on which node

     kubectl get svc
f) To delete the deployment 

     kubectl delete deployment <name>

