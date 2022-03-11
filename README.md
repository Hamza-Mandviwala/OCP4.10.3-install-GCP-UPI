# OCP 4.10.3 installation on GCP using UPI method

This guide is intended to walk you through a step-by-step procedure of deploying a 5 node (3 Masters & 2 Workers) OCP 4.10.3 cluster on GCP using the User-Provisioned-Infrastructure (UPI) method.

We have selected GCP as our infrastructure platform, mainly because it offers some free credits upon account creation to get yourself familiarized with the platform. This is more than enough to get our RedHat OpenShift Container Platform 4.10.3 up and running seamlessly using the UPI method at no cost at all.

This guide also covers a lot of basics including the creation of basic GCP components like projects, networks etc. So, if you are comfortable with provisioning your infrastructure components yourself, feel free to skip over to the installation process directly.

The procedure and steps described in this document have been taken mainly from the official RedHat documentation for OpenShift (https://docs.openshift.com/container-platform/4.10/installing/installing_gcp/installing-restricted-networks-gcp.html) .

Some changes have been made to the original steps described in the official docs. These are mainly the creation of networks, subnetworks, DNS zones, Cloud NATs – for which we will walk you through the steps to be performed on the UI, instead of the CLI. The idea is to make you understand the architecture of the cluster clearly.

We will also be skipping the steps of the intermediate service accounts required for worker & master nodes, instead we will be using the same service account that we will be creating initially for authentication and authorization.

## Contents ##
* Basic Flow and Summary of deployment steps
* Final Architecture and Cluster Layout (Post Installation)
* Let’s Get Started!
  * Cluster Specs
  * Prerequisites
  * Steps:
    * Setting up GCP infra components

## Basic Flow and Summary of deployment steps
![My Image](/Users/hamzawork/Desktop/flowdiagram.png)


The above diagram illustrates the basic flow in which we will be deploying our OCP 4.10.3 cluster on GCP. We first create a bastion host, and from that host, we will be running all the necessary commands to setup the bootstrap node, followed by the Master nodes, and finally the worker nodes in a different subnet.

Before we begin the bootstrap process, we will be creating some prerequisite infrastructure components like network, subnetworks, IAM service account, IAM Project, Private DNS zone, Load balancers, Cloud NATs & a Cloud Router.

After the bootstrap process is complete, we will be removing all the bootstrap components from the cluster. And then we will proceed with the creation of the worker nodes.

Once we have our worker nodes up and running, we will verify if all our cluster controllers are showing as ‘available’. Finally, we will configure a reverse proxy on the bastion host so that we can access the OCP Console UI locally on our browser.

## Final Architecture and Cluster Layout (Post Installation)

![My Image](/Users/hamzawork/Desktop/finaldiagram.png)

## Let’s Get Started

### Cluster Specs:
Node	CPUs	Memory/GB	Disk Size	OS	Deployment type	Subnet
Bastion Host	2	8	100Gib	Ubuntu 18.04	Manual	Master-subnet
Bootstrap Node	4	16	128 Gib	RedHat CoreOS	Deployment Manager Template	Master-subnet
Master 1	4	16	128 Gib	RedHat CoreOS	Deployment Manager Template	Master-subnet
Master 2	4	16	128 Gib	RedHat CoreOS	Deployment Manager Template	Master-subnet
Master 3	4	16	128 Gib	RedHat CoreOS	Deployment Manager Template	Master-subnet
Worker 1	4	16	128 Gib	RedHat CoreOS	Deployment Manager Template	Worker-Subnet
Worker 2	4	16	128 Gib	RedHat CoreOS	Deployment Manager Template	Worker Subnet

### Prerequisites:
1.	A GCP account.
2.	The necessary APIs to be enabled as per the documentation here.
3.	Ensure you have necessary limits/quotas available in the region you wish to deploy your cluster into. 
4.	Basic Linux administration skills.
5.	Understanding of IP, routing, reverse proxy (recommended).
6.	A service account configured with the appropriate privileges to perform the below steps. Note that majority of the commands used in this deployment process were using the gcloud & gsutil CLI binaries. These can be downloaded from here.
7.	A RedHat account to download the necessary binaries (oc, kubectl & openshift-installer binaries), RedHat coreos and the pull secret from. If you do not have a RedHat account, you can create one here.

### Steps:

1. Create a GCP project inside your GCP console. I have a project named ‘ocp-project’ created already, and this will be used to host the OCP 4.10.3 cluster.
   1. Go to **IAM & Admin** > **Create a Project**
   2. Enter the project name & organization and click on *Create*.

2. Create a service account under IAM:
   1. Go to **IAM & admin** > **Service Accounts**
   2.	Click on **Create Service Account** and enter the relevant details:
         1.	Service Account name: *ocp-serviceaccount*
         2.	Grant access to the necessary roles per the documentation. I have assigned it the **owner** role since I will be using this service account myself. It is however not recommended to grant the ‘owner’ role to the service account for security reasons. Do refer the necessary roles that the service account requires access to.
         3.	Click **Done**.
         4.	Once your service account is created, we need its json key to be used for authentication & authorization of gcp objects creation:
             1. Click on your service account name from the service accounts list.
             2. Go to the **keys** tab.
             3. Click on **Add key** > **Create New Key** and follow the instructions to create and download the json key.

3. Create a Network from the GCP UI Console:
   1.	Go to **VPC Networks** > **Create VPC Networks**
   2. Put in the appropriate network name. *ocp-network* is used in this demo.
   3. Add 2 new subnets within this network:
         1. For the Master Nodes & Bootstrap node:
             1. Name: master-subnet
             2. Subnet range: 10.1.10.0/24
             3.	Private Google Access: On
             4.	Flow Logs: Off
         2.	For the worker nodes:
             1.	Name: worker-subnet
             2.	Subnet Range: 10.1.20.0/24
             3.	Private Google Access: On
             4.	Flow Logs: Off
   4. Click **Create**.

4.	Create a Firewall Rule to allow all traffic communication between all the instances of the network ‘ocp-network’.

5. Create a ‘Cloud Router’ from the GCP console.
   1. Go to **‘Hybrid Connectivity’** > **‘Cloud Routers’**.
   2. Click on **‘Create Router’**,
   3.	Enter the relevant details:
         1. Network: ocp-network
         2. Region: asia-northeast1
         3. Select *“Advertise all subnets visible to the Cloud Router (Default)”*
         4. Click **Create**.

6. Create 2 Cloud NAT components connected to the router we created above, for both of our subnets created earlier.
   1. Go to **‘Network Services’** > **‘Cloud NAT’**.
   2. Enter the relevant details:
         1. Gateway Name: ocp-nat-master-gw
         2. Network: ocp-network
         3. Region: asia-northeast1
         4. Cloud Router: ocp-router (this is the name of the cloud router we created in the previous step)
         5. NAT Mapping  
             1. Source: Custom
             2. Subnets: master-subnet
             3. NAT IP Addresses: Automatic
         6. Click **Create** .
         7. Repeat steps a, b & c again for the worker-subnet, and just ensure to change the Gateway Name to ocp-nat-worker-gw for example. And ensure both Cloud Networks are connected to the same Cloud Router.

7. Create a private DNS zone from the GCP UI:
   1. Go to **‘Network Services’** > **‘Cloud DNS’**
   2. Click on **‘Create Zone’** and enter the relevant details:
         1. Zone type: Private
         2. Zone name: ocp-private-zone (you can use any name of your choice)
         3. DNS name: hamzacluster.openshift.com (The DNS name consists of a combination of the cluster name & base domain i.e <cluster_name>.<base_domain> . In this example ‘hamzacluster’ is the name of my cluster and ‘openshift.com’ is the base domain that I will be using. Ensure you enter the values correctly as the ‘cluster name’ & ‘base domain’ you specify here will be the same that will have to be used in the install-config.yaml file later.
         4. Network: ocp-network (this needs to be the network we created earlier within which our master-subnet & worker-subnet reside)
         5.	Click **‘Create’**.

8. Create a bastion host in the master-subnet of the ocp-network we created earlier. Bastion host is basically a normal VM instance that can be used log into and run the necessary commands from. Ensure it has an external IP assigned to it as we will be ssh’ing into it and run all the necessary commands from there.
   1. Go to **Compute Engine** > **‘VM Instances’** > **‘Create Instance’**
   2. Put in the relevant details: Instance name, type (e2-standard-2 is used for my demo), region, zone.
   3.	For the boot disk, I have used Ubuntu 18.04 OS with 100 GB disk size.
   4.	For the Networking:
         1. Assign a hostname of your choice. 
         2. Connect the network interface to the ocp-network, and subnet of master-subnet. You can set both Primary Internal IP & External IP to ‘Ephemeral’.
         3.	Keep IP forwarding enabled.
         4. Network Interface Card: VirtIO
   5.	Under **‘Security’** > **‘Manage Access’** -  ensure you add a ssh public key that allows you to ssh into the bastion host from your local machine.
   6.	Click **‘Create’**.

9. SSH into the bastion host from your local machine, and switch to the root user.
        
       ssh -i ~/.ssh/id_rsa username@<Public IP of bastion host>
       sudo su -
10. Copy over the downloaded json key (from step 2 above) to your bastion host.
11. Download some important packages:

        sudo apt install wget git -y
    
12. Download the necessary files & binaries onto the bastion host from your RedHat Cluster Manager login page (you can use the wget tool for this):
    1. openshift-installer binary
    2. oc binary
    3.	kubectl binary
    4.	RedHat Coreos image
    5.	Pull Secret

13. Once downloaded and extracted, copy the openshift-installer, oc & kubectl binary into /usr/local/bin/ directory (Or whatever the $PATH you have configured on your bastion host).
14. Generate a new ssh key pair on your bastion host keeping all default options. The public key from this key pair will be used inserted in your install-config.yaml file. Your cluster nodes will be injected with this ssh key and you will be able to ssh into them later for any kind of monitoring & troubleshooting!
             
        ssh-keygen
      
15. Clone my repository. This contains the necessary .py files required to build your cluster components like load balancers & VM instances.

        git clone https://github.com/Hamza-Mandviwala/OCP4.10.3-install-GCP-UPI.git
        
16. Create an installation directory which will be used to generate the manifests & ignition config files.

        mkdir install_dir

17. Copy the sample-install-config.yaml file into the installation directory.

        cp example-install-config.yaml install_dir/install-config.yaml
        
18. Edit the copied install-config.yaml as per your needs. The important changes are:
    1. Base Domain value (Needs to be the same as specified in your private DNS zone)
    2. Cluster name (Needs to be the same as specified in your private DNS zone)
    3. Pull Secret (To be taken from your RedHat account portal)
    4. SSH key (The one you created in step 14 above. The public key is present at ~/.ssh/id_rsa.pub on most Linux machines)
    5. GCP platform specific parameters like project ID, region will have to specified per your choice. In this demo, we have chosen asia-northeast1 as it is the closest to my geographical location and has the necessary limits to host the cluster.

19. Export the GCP application credentials. This is the service account key that will be used for creating the necessary infra components.

        export GOOGLE_APPLICATION_CREDENTIALS=<path to your service account json key file>
        
20. Create the manifest files for your openshift cluster.

        openshift-install create manifests --dir install_dir/
    
21. The manifests need some changes to be done as follows:
    1. Open the file install_dir/manifests/cluster-ingress-default-ingresscontroller.yaml 

           vi install_dir/manifests/cluster-ingress-default-ingresscontroller.yaml
    2. Under spec.endpointPublishingStrategy :
       1. Remove the ‘loadbalancer’ section completely so that only the ‘type’ section remains.
       2. For the ‘type’ section, change the value to ‘HostNetwork’.
       3. Add the parameter of ‘replicas: 2’ below the ‘type’.
       4. Your resulting file should have the section look something like:
                  
              spec:
                endpointPublishingStrategy:
                  type: HostNetwork
                  replicas: 2
       5. Save the file.
       6. Refer the example-cluster-ingress-default-ingresscontroller.yaml file to compare and see how the resulting file should look like.
    3. Remove the manifest files for the worker & master machines, as we will be creating the master & worker nodes using the Deployment Manager templates.
              
           rm -f install_dir/openshift/99_openshift-cluster-api_master-machines-*
           rm -f install_dir/openshift/99_openshift-cluster-api_worker-machineset-*

22. Now let’s create the ignition config files.

          openshift-install create ignition-configs --dir install_dir/
          
23. 


































