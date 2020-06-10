
<!--
title: "Rancher 2.4 on K3s with an external MySQL Database in Azure using only the Cloud Shell"
author: Mike Bookham
date:
description: "This tutorial will provide a step by step look at how to deploy Rancher HA using K3s as the Kubernetes provider with an external MySQL database in Azure."
type: "blog"
tags: ["Tutorial"]
categories: [blog]

 -->
# Run Rancher 2.4 in Azure with K3s and MySQL  
## Introduction

This article will describe the process of installing Rancher 2.4 on a highly available K3s Kubernetes cluster in Microsoft Azure. It will also take advantage of Microsoft Azure Database for MySQL, which removes the dependency on etcd and provides us all the additional features that Azure delivers with this service.

You will learn how to deploy the infrastructure to support this pattern using only the Azure Cloud Shell. The benefit of using the Cloud Shell is that you need zero infrastructure to get started -- just access to the Azure portal. A number of the CLI tools we need are already preinstalled, significantly reducing the effort needed to complete the install.

Once you have deployed the infrastructure, you will learn how to deploy Rancher 2.4 on a Kubernetes cluster using K3s. In Rancher 2.4, we've added a new supported deployment pattern: Rancher 2.4 on two nodes running K3s with an external database. One of the benefits of using this pattern is that we can treat the nodes as ephemeral. We are able to do this as a result of K3s supporting an external MySQL database.

K3s is a lightweight, fully compliant Kubernetes distribution that takes the form of a single 50MB binary. It is more modern that the alternative for deploying Rancher, Rancher Kubernetes Engine (RKE), and has the following enhancements:

1. An embedded SQLite database has replaced etcd as the default datastore. It also supports external datastores such as PostgreSQL, MySQL and etcd. (We are using MySQL in the article.)
2. We've added simple but powerful “batteries included” features, such as a local storage provider, a service load balancer, a Helm controller and the Traefik ingress controller.
3. Operation of all Kubernetes control plane components is encapsulated in a single binary and process. This allows K3s to automate and manage complex cluster operations like distributing certificates.
4. We've removed in-tree cloud providers and storage plugins.
5. We've minimized external dependencies (just a modern kernel and cgroup mounts needed). K3s packages require dependencies, including: Containerd, Flannel, CoreDNS and host utilities (iptables, socat, etc.)

If you are thinking of trying Rancher for the first time, consider this deployment pattern. It is likely that this will eventually become the preferred method of deploying Rancher, so get a head start on the crowd -- especially if you are running your data center in Azure.


## Prerequisites
In order to complete this tutorial, you will need the following:

* **Microsoft Account:** Login credentials for Microsoft. These could be your corperate Azure Active Directory credentials or just a normal Outlook account
      [Create a Microsoft Account](https://account.microsoft.com/account/Account?refd=login.live.com&ru=https%3A%2F%2Faccount.microsoft.com%2F%3Frefd%3Dlogin.live.com&destrt=home-index)
* **Access to an Azure Subscription:** This can be a Free Trial/Pay as you go/or an Enterprise Subscription 
      [Free Trial Subscription](https://azure.microsoft.com/en-us/free/)
* **Access to the [Azure Portal](https://portal.azure.com)** 

## Architecture
This high-level diagram shows the resources that will be created in Azure. 

<!--  ![[TO DO Insert Image of Azure Architecture]https://granitecomputing.sharepoint.com/:i:/s/MarketingGlobal/EbgeACbms0BHsQsJ36TF_GEBX8ozJyymMPwKNOQycuydsg?e=Wri1Rr]-->

The two nodes will be placed on their own vNet in a single subnet. These will be fronted by an Azure Load Balancer. The MySQL database will be delivered from outside the vNet as it is managed by Microsoft. The nodes will be secured with a single Network Security Group (NSG) attached to the subnet.

## Azure Cloud Shell

We'll use only the Azure Cloud Shell to provision all of the elements required to run Rancher on K3s in Azure. In the portal, click on the **Azure Cloud Shell** button in the top right hand corner. It's the icon with the greater than and underscore symbol.

<!-- [To Do Insert Image of Azure Cloud Shell][(https://granitecomputing.sharepoint.com/:i:/s/MarketingGlobal/EfaULSUDRGlGhn_nBs1agewBRZYzi-YaxHQCy7e-fxhAlQ?e=yzZzh3)]-->


## Azure Networking

### Resource Group
In Azure, all resouces need to belong to a resource group so we'll create this first. We're also going to set the default Region and Resource Group just to ensure that all of our resouses are created in the correct place.

**NOTE**: I'm using **eastus2** as my region but feel free to change this as required.

```
az group create -l eastus2  -n RancherK3sResourceGroup
az configure --defaults location=eastus2 group=RancherK3sResourceGroup
```
### Vnet, Public IPs and Network Security Group
Once these commands have completed, the networking elements are created inside the Resource Group. These include a vNet with a default subnet, two public IPs for the two Virtual Machines (VMs) that we will create later and a Network Secrity Group (NSG).

```
az network vnet create --resource-group RancherK3sResourceGroup --name RancherK3sVnet --subnet-name RancherK3sSubnet

az network public-ip create --resource-group RancherK3sResourceGroup --name RancherK3sPublicIP1 --sku standard

az network public-ip create --resource-group RancherK3sResourceGroup --name RancherK3sPublicIP2 --sku standard

az network nsg create --resource-group RancherK3sResourceGroup --name RancherK3sNSG1

az network nsg rule create -g RancherK3sResourceGroup --nsg-name RancherK3sNSG1 -n NsgRuleSSH --priority 100 \
    --source-address-prefixes '*' --source-port-ranges '*' \
    --destination-address-prefixes '*' --destination-port-ranges 22 --access Allow \
    --protocol Tcp --description "Allow SSH Access to all VMS."

```
### Azure Load Balancer

Since we're installing K3s on two VMs, we need a LoadBalancer to provide resilliance and protect against VM failure.

First, create a Public IP for the Load Balancer.
```
az network public-ip create --resource-group RancherK3sResourceGroup --name RancherLBPublicIP --sku standard
```

Next, create the Load Balancer with a health probe.
```
az network lb create \
    --resource-group RancherK3sResourceGroup \
    --name K3sLoadBalancer \
    --sku standard \
    --public-ip-address RancherLBPublicIP \
    --frontend-ip-name myFrontEnd \
    --backend-pool-name myBackEndPool

az network lb probe create \
    --resource-group RancherK3sResourceGroup \
    --lb-name K3sLoadBalancer \
    --name myHealthProbe \
    --protocol tcp \
    --port 80   
```
Once the Load Balancer is created the NSG is updated. The ports added are **80 and 443** for access to Rancher Server and **6443** for access the the Kubernetes API of K3s.

```
az network nsg rule create \
    --resource-group RancherK3sResourceGroup \
    --nsg-name RancherK3sNSG1 \
    --name myNetworkSecurityGroupRuleHTTP \
    --protocol tcp \
    --direction inbound \
    --source-address-prefix '*' \
    --source-port-range '*' \
    --destination-address-prefix '*' \
    --destination-port-range 80 443 6443 \
    --access allow \
    --priority 200
```

Now add the Load Balancer configuration in the form of three rules. You need a rule for port 80 and a rule for port 443 to distrubute the load for Rancher server across our two VMs. The third rule is for port 6443, which gives access to the Kubernetes API running on each of the VMs.

```
az network lb rule create \
    --resource-group RancherK3sResourceGroup \
    --lb-name K3sLoadBalancer \
    --name myHTTPRule \
    --protocol tcp \
    --frontend-port 80 \
    --backend-port 80 \
    --frontend-ip-name myFrontEnd \
    --backend-pool-name myBackEndPool \
    --probe-name myHealthProbe

az network lb rule create \
    --resource-group RancherK3sResourceGroup \
    --lb-name K3sLoadBalancer \
    --name myHTTPSRule \
    --protocol tcp \
    --frontend-port 443 \
    --backend-port 443 \
    --frontend-ip-name myFrontEnd \
    --backend-pool-name myBackEndPool \
    --probe-name myHealthProbe

az network lb rule create \
    --resource-group RancherK3sResourceGroup \
    --lb-name K3sLoadBalancer \
    --name myHTTPS6443Rule \
    --protocol tcp \
    --frontend-port 6443 \
    --backend-port 6443 \
    --frontend-ip-name myFrontEnd \
    --backend-pool-name myBackEndPool \
    --probe-name myHealthProbe
```

## Azure Database as a Service (DaaS)

One of the benefits of using K3s as the Kuberentes distribution is that it supports alteratives to etcd. In the example, we'll use MySQL from Azure's database as a service capability.

To create a MySQL database, run the following CLI commands.

First lets create a variable for the name of the database server. This will make running the subsequent commands easier.
**NOTE** The name of your database server needs to be unique across the whole of Azure. If its not you will get and error when creating.

```
K3smysqlserver=<unique-myslq-server-name>
```
Create your MySQL Server. If the name was not unique then an error will be displayed. If it is then update the variable with a new name and run this command again.
```
az mysql server create --resource-group RancherK3sResourceGroup --name $K3smysqlserver --admin-user myadmin --admin-password Password1 --sku-name GP_Gen5_2 --version 5.7
```
Create a firewall rule to allow all the Azure IP access to your database server.

```
az mysql server firewall-rule create --resource-group RancherK3sResourceGroup --server $K3smysqlserver --name "AllowAllWindowsAzureIps" --start-ip-address 0.0.0.0 --end-ip-address 0.0.0.0
```

Add service endpoint for existing subnet.
```
az network vnet subnet update --vnet-name RancherK3sVnet --name RancherK3sSubnet --service-endpoints "Microsoft.Sql"
```

Add vnet rule to database access.
```
az mysql server vnet-rule create --server $K3smysqlserver --name MyK3sVNetRule \
    -g RancherK3sResourceGroup --subnet RancherK3sSubnet --vnet-name RancherK3sVnet
```

Disable TLS for database communication.
```
az mysql server update --resource-group RancherK3sResourceGroup --name $K3smysqlserver --ssl-enforcement Disabled
```
MySQL CLI tool is already installed in the Azure Cloud Shell. The next step is to connect to the MySQL Server and and create a database.

Connect to the new MySQL server.
```
mysql --host $K3smysqlserver.mysql.database.azure.com --user myadmin@$K3smysqlserver -p

```

Check status to make sure MySQL is running.
```
status
```
Create an empty database.
```
CREATE DATABASE kubernetes;

SHOW DATABASES;

exit
```

## Azure Virtual Machines

Next, we'll create the two VMs and install K3s on both of them.

### Network Interfaces

With all of the network elements created, we can create the Network Interface Cards (NIC) for the VMs.
```
az network nic create --resource-group RancherK3sResourceGroup --name nic1 --vnet-name RancherK3sVnet --subnet RancherK3sSubnet --network-security-group RancherK3sNSG1 --public-ip-address RancherK3sPublicIP1 --lb-name K3sLoadBalancer --lb-address-pools myBackEndPool

az network nic create --resource-group RancherK3sResourceGroup --name nic2 --vnet-name RancherK3sVnet --subnet RancherK3sSubnet --network-security-group RancherK3sNSG1 --public-ip-address RancherK3sPublicIP2 --lb-name K3sLoadBalancer --lb-address-pools myBackEndPool
```
### Create the Virtual Machines

To create two virtual machines, first create a text file with our cloud-init configuration. This will deploy Docker, add the ubuntu user to the docker group and install k3s.

```
cat << EOF > cloud-init.txt
#cloud-config
package_upgrade: true
packages:
  - curl
output: {all: '| tee -a /var/log/cloud-init-output.log'}
runcmd:
  - curl https://releases.rancher.com/install-docker/18.09.sh | sh
  - sudo usermod -aG docker ubuntu
  - curl -sfL https://get.k3s.io | sh -s - server --datastore-endpoint="mysql://myadmin@$K3smysqlserver:Password1@tcp($K3smysqlserver.mysql.database.azure.com:3306)/kubernetes"
EOF
```
Deploy the virtual machines.
```
az vm create \
  --resource-group RancherK3sResourceGroup \
  --name K3sNode1 \
  --image UbuntuLTS \
  --nics nic1 \
  --admin-username ubuntu \
  --generate-ssh-keys \
  --custom-data cloud-init.txt


az vm create \
  --resource-group RancherK3sResourceGroup \
  --name K3sNode2 \
  --image UbuntuLTS \
  --nics nic2 \
  --admin-username ubuntu \
  --generate-ssh-keys \
  --custom-data cloud-init.txt

```

## Check Kubernetes is Running

As part of the VM provisioning, K3s should have been installed. Let's connect to the first VM and confirm K3s is running.

```
ssh ubuntu@<publicIPofNode1>
```
Both VMs should be on the list of nodes. If this dosen't work the first time, give it a couple of minutes to run the cloud-init script. It may take a short time to deploy Docker and K3s.

```
sudo k3s kubectl get nodes
```
Example output:
```
ubuntu@ip-172-31-60-194:~$ sudo k3s kubectl get nodes
NAME               STATUS   ROLES    AGE    VERSION
ip-172-31-60-194   Ready    master   44m    v1.17.2+k3s1
ip-172-31-63-88    Ready    master   6m8s   v1.17.2+k3s1 
```

Then test the health of the cluster pods:
```
sudo k3s kubectl get pods --all-namespaces
```

### Save and Start Using the kubeconfig File

While still connected to one of our nodes, we need to grab the contents of the kubeconfig for the cluster. Output the contents to the screen with the command below and copy this to your clipboard.

```
sudo cat /etc/rancher/k3s/k3s.yaml
```
Paste this into a text editor so we can make a change before adding it to the Azure Cloud Shell session we are working in.

Update the `server:` with the external URL of the Load Balancer. You can use a xip.io service to give you a resolvable fully qualified domain name. See the screenshot below.

For example: https://rancher.<LoadBalancerPublicIP>.xip.io:6443>

![[TO DO Insert Image of Kubeconfig](https://granitecomputing.sharepoint.com/:i:/s/MarketingGlobal/EbYcUIN_mJZFiW4WCgNSTbEBmK6V720sJwk2NsJHfSivJQ?e=EQoyS3)(./images/kubeconfig.png)


**NOTE**: Replace the example in the screenshot with the Public IP of your loadbalancer.


Now create a file called `config` in the `/.kube` folder and paste the updated content into that file.

First, disconnet from node1.
```
exit
```
Now create the new directory and edit the file. Paste in the updated contents.
```
mkdir ~/.kube
vi ~/.kube/config
```

Check that kubectl is working and can talk to the cluster. Kubectl and Helm are installed in the Azure Cloud Shell.
```
kubectl get pods --all-namespaces
```

## Install Rancher

Add the Rancher Helm Repo.

```
helm repo add rancher-latest https://releases.rancher.com/server-charts/latest
```
Create the `cattle-system` namespace.
```
kubectl create namespace cattle-system
```

Install the CustomResourceDefinition resources separately.
```
kubectl apply -f https://raw.githubusercontent.com/jetstack/cert-manager/release-0.12/deploy/manifests/00-crds.yaml
```

Create the namespace for `cert-manager`.
```
kubectl create namespace cert-manager
```

Add the Jetstack Helm repository.
```
helm repo add jetstack https://charts.jetstack.io
```

Update your local Helm chart repository cache.
```
helm repo update
```

Install the cert-manager Helm chart.
```
helm install \
  cert-manager jetstack/cert-manager \
  --namespace cert-manager \
  --version v0.12.0
```

Check that Cert-Manager is running, make sure you wait until all of the pods are running.
```
kubectl get pods --namespace cert-manager
```

Install Rancher with Self-Signed Certificates. Ensure you set the host name with the URL of you Rancher server. In this article, we are taking advantage of the xip.io service. Use the Public IP address of the Azure load balancer in the Rancher URL.
```
helm install rancher rancher-latest/rancher \
  --namespace cattle-system \
  --set hostname=rancher.<LoadBalancerPublicIP>.xip.io
```

Wait for Rancher to be deployed...
```
kubectl -n cattle-system rollout status deploy/rancher
```
Once all three replicas are rolled out, click on the url of the Rancher server deployment, shown here.

![[TO DO Insert Image of Rancher URL](https://granitecomputing.sharepoint.com/:i:/s/MarketingGlobal/Ecch1BF_qhJNqkA3s61gbScB4LQqZJ58Akmkt95-gmr2hw?e=hOszY2)(./images/rancherurl.png)



## Clean Up 
Creating resources in Azure costs money, so make sure you delete the Resource Group once you're finished. 
```
az group delete --name RancherK3sResourceGroup
```

## Conclusion




In this article, we provided a quick and easy way to get started using Rancher to provide multi-cluster management of your containerized workloads in Azure. By using K3s, not only have we been able to get up and running extremely quickly, we also have been able to remove etcd and some of the headaches associated with running it in production. By using the Azure Cloud Shell, authentication was easy and all of the tools we needed were available out of the box.




