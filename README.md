# OCP 4.10.3 installation on GCP using UPI method

This guide is intended to walk you through a step-by-step procedure of deploying a 5 node (3 Masters & 2 Workers) OCP 4.10.3 cluster on GCP using the User-Provisioned-Infrastructure (UPI) method.

We have selected GCP as our infrastructure platform, mainly because it offers some free credits upon account creation to get yourself familiarized with the platform. This is more than enough to get our RedHat OpenShift Container Platform 4.10.3 up and running seamlessly using the UPI method at no cost at all.

This guide also covers a lot of basics including the creation of basic GCP components like projects, networks etc. So, if you are comfortable with provisioning your infrastructure components yourself, feel free to skip over to the [openshift installation](https://github.com/Hamza-Mandviwala/OCP4.10.3-install-GCP-UPI/edit/main/README.md#openshift-installation) process directly. It would still be recommended to follow through this document entirely to avoid any unexpected issues.

The procedure and steps described in this document have been taken mainly from the official RedHat documentation for OpenShift (https://docs.openshift.com/container-platform/4.10/installing/installing_gcp/installing-restricted-networks-gcp.html) .

Some changes have been made to the original steps described in the official docs. These are mainly the creation of networks, subnetworks, DNS zones, Cloud NATs – for which we will walk you through the steps to be performed on the UI, instead of the CLI. The idea is to understand how the cluster looks from the ground up.

We will also be skipping the steps of the intermediate service accounts required for worker & master nodes, instead we will be using the same service account that we will be creating initially for authentication and authorization.

## Contents ##
* [Basic Flow and Summary of deployment steps](https://github.com/Hamza-Mandviwala/OCP4.10.3-install-GCP-UPI/edit/main/README.md#basic-flow-and-summary-of-deployment-steps)
* [Final Architecture and Cluster Layout (Post Installation)](https://github.com/Hamza-Mandviwala/OCP4.10.3-install-GCP-UPI/edit/main/README.md#final-architecture-and-cluster-layout-post-installation)
* [Let’s Get Started!](https://github.com/Hamza-Mandviwala/OCP4.10.3-install-GCP-UPI/edit/main/README.md#lets-get-started)
  * [Cluster Specs](https://github.com/Hamza-Mandviwala/OCP4.10.3-install-GCP-UPI/edit/main/README.md#cluster-specs)
  * [Prerequisites](https://github.com/Hamza-Mandviwala/OCP4.10.3-install-GCP-UPI/edit/main/README.md#prerequisites)
  * [Steps:](https://github.com/Hamza-Mandviwala/OCP4.10.3-install-GCP-UPI/edit/main/README.md#steps)
    * [Creating GCP Infra components](https://github.com/Hamza-Mandviwala/OCP4.10.3-install-GCP-UPI/edit/main/README.md#creating-gcp-infra-components)
    * [OpenShift Installation](https://github.com/Hamza-Mandviwala/OCP4.10.3-install-GCP-UPI/edit/main/README.md#openshift-installation)
    * [Creating a second Internal Loadbalancer for worker plane traffic](https://github.com/Hamza-Mandviwala/OCP4.10.3-install-GCP-UPI/edit/main/README.md#creating-a-second-internal-loadbalancer-for-worker-plane-traffic) (Mandatory step for cluster install to reach completion)
    * [Configuring a Reverse Proxy for external UI access](https://github.com/Hamza-Mandviwala/OCP4.10.3-install-GCP-UPI/edit/main/README.md#configuring-a-reverse-proxy-for-external-ui-access)


## Basic Flow and Summary of deployment steps

<img width="1792" alt="flowdiagram" src="https://user-images.githubusercontent.com/53118271/157832070-351606fa-92fd-45b7-96a2-e3fa7d83ddb0.png">

The above diagram illustrates the basic flow in which we will be deploying our OCP 4.10.3 cluster on GCP. We first create a bastion host, and from that host, we will be running all the necessary commands to setup the bootstrap node, followed by the Master nodes, and finally the worker nodes in a different subnet.

Before we begin the bootstrap process, we will be creating some prerequisite infrastructure components like network, subnetworks, IAM service account, IAM Project, Private DNS zone, Load balancers, Cloud NATs & a Cloud Router.

After the bootstrap process is complete, we will be removing all the bootstrap components from the cluster. And then we will proceed with the creation of the worker nodes.

Once we have our worker nodes up and running, we will verify if all our cluster controllers are showing as ‘available’. Finally, we will configure a reverse proxy on the bastion host so that we can access the OCP Console UI locally on our browser.

## Final Architecture and Cluster Layout (Post Installation)

<img width="1012" alt="finaldiagram" src="https://user-images.githubusercontent.com/53118271/157832353-7b5ffecd-9738-4be7-b424-0c706a928beb.png">


# Let’s Get Started

## Cluster Specs:

| Node	| CPUs	| Memory/GB	| Disk Size	| OS	| Deployment type	|Subnet| Instance Type |
|------|------|-----------|-----------|----|-----------------|------|---------------|
| Bastion Host |	2 |	8	| 100Gib |	Ubuntu 18.04 |	Manual |	Master-subnet | e2-standard-2 |
| Bootstrap Node	| 4	|16	|128Gib	| RedHat CoreOS	| Deployment Manager Template	| Master-subnet | e2-standard-4 |
| Master 1	| 4	| 16 | 128Gib	| RedHat CoreOS	| Deployment Manager Template	| Master-subnet | e2-standard-4 |
| Master 2 |	4 |	16	| 128Gib	| RedHat CoreOS	| Deployment Manager Template	| Master-subnet | e2-standard-4 |
| Master 3	| 4	| 16	| 128Gib	| RedHat CoreOS	| Deployment Manager Template	| Master-subnet | e2-standard-4 |
| Worker 1	| 4	| 16	| 128Gib	| RedHat CoreOS	| Deployment Manager Template |	Worker-Subnet | e2-standard-4 |
| Worker 2	| 4	| 16	| 128Gib	| RedHat CoreOS	| Deployment Manager Template	| Worker Subnet | e2-standard-4 |

## Prerequisites:
1.	A GCP account.
2.	The necessary APIs to be enabled as per the documentation here.
3.	Ensure you have necessary limits/quotas available in the region you wish to deploy your cluster into. 
4.	Basic Linux administration skills.
5.	Basic understanding of public cloud platforms like AWS, GCP, Azure.
6.	Understanding of IP, routing, reverse proxy (recommended).
7.	A service account configured with the appropriate privileges to perform the below steps. Note that majority of the commands used in this deployment process were using the gcloud & gsutil CLI binaries. These can be downloaded from [here](https://cloud.google.com/sdk/docs/install).
8.	A RedHat account to download the necessary binaries (oc, kubectl & openshift-installer binaries), RedHat coreos and the pull secret from. If you do not have a RedHat account, you can create one [here](https://www.redhat.com/wapps/ugc/register.html?_flowId=register-flow&_flowExecutionKey=e1s1).

## Steps:

### Creating GCP Infra components

1. Create a GCP project inside your GCP console. I have a project named ‘ocp-project’ created already, and this will be used to host the OCP 4.10.3 cluster.
   1. Go to **IAM & Admin** > **Create a Project**
   2. Enter the project name & organization and click on *Create*.

2. Create a service account under IAM:
   1. Go to **IAM & admin** > **Service Accounts**
   2.	Click on **Create Service Account** and enter the relevant details:
         1.	Service Account name: *ocp-serviceaccount* (you can use any service account name of your choice)
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
   3. Click **Create** .
   4. Repeat steps i, ii & iii again for the worker-subnet, and just ensure to change the Gateway Name to ocp-nat-worker-gw for example. And ensure both Cloud Networks are connected to the same Cloud Router.

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

### OpenShift Installation 

9. SSH into the bastion host from your local machine, and switch to the root user.
        
       ssh -i ~/.ssh/id_rsa username@<Public IP of bastion host>
       sudo su -
10. Copy over the downloaded json key (from step 2 above) to your bastion host.
11. Download some important packages:

        sudo apt install wget git -y
    
12. Download the necessary files & binaries onto the bastion host from your [RedHat Cluster Manager](https://sso.redhat.com/auth/realms/redhat-external/protocol/openid-connect/auth?client_id=cloud-services&redirect_uri=https%3A%2F%2Fconsole.redhat.com%2F&state=1b3b6ba7-b426-4319-9197-8d4be1f14e5f&response_mode=fragment&response_type=code&scope=openid&nonce=72beef5c-277a-4dd7-a840-721e8eddbcac) login page (you can use the wget tool to directly download onto your bastion host, or simply download them and copy over to your bastion host using scp):
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

        cp sample-install-config.yaml install_dir/install-config.yaml
        
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
          
23. Set the environment variables for your environment. Please set the values as per your needs.

        export BASE_DOMAIN=openshift.com
        export BASE_DOMAIN_ZONE_NAME=ocp-private-zone
        export NETWORK=ocp-network
        export MASTER_SUBNET=master-subnet
        export WORKER_SUBNET=worker-subnet
        export NETWORK_CIDR='10.1.0.0/16'
        export MASTER_SUBNET_CIDR='10.1.10.0/24'
        export WORKER_SUBNET_CIDR='10.1.20.0/24'
        export KUBECONFIG=/root/install_dir/auth/kubeconfig 
        export CLUSTER_NAME=`jq -r .clusterName /root/install_dir/metadata.json`
        export INFRA_ID=`jq -r .infraID /root/install_dir/metadata.json`
        export PROJECT_NAME=`jq -r .gcp.projectID /root/install_dir/metadata.json`
        export REGION=`jq -r .gcp.region /root/install_dir/metadata.json`
        export CLUSTER_NETWORK=(`gcloud compute networks describe ${NETWORK} --format json | jq -r .selfLink`)
        export CONTROL_SUBNET=(`gcloud compute networks subnets describe ${MASTER_SUBNET} --region=${REGION} --format json | jq -r .selfLink`)
        export COMPUTE_SUBNET=(`gcloud compute networks subnets describe ${WORKER_SUBNET} --region=${REGION} --format json | jq -r .selfLink`)
        export ZONE_0=(`gcloud compute regions describe ${REGION} --format=json | jq -r .zones[0] | cut -d "/" -f9`)
        export ZONE_1=(`gcloud compute regions describe ${REGION} --format=json | jq -r .zones[1] | cut -d "/" -f9`)
        export ZONE_2=(`gcloud compute regions describe ${REGION} --format=json | jq -r .zones[2] | cut -d "/" -f9`)
      
    These environment variables will be required for the upcoming commands as they call these variables in them, so be sure to set these.

    Ensure you copy the .py files to your current working directory from the git repository, as these will be referenced in the upcoming commands.

24. Now we create the internal loadbalancer component that will be used for api-server related communication by the cluster nodes. The following commands will create the load balancer, its corresponding health check component, as well as the Backend empty instance groups into which the master nodes will be put into at the time of Master nodes creation in a later step.
    1. Create the .yaml file
       
           cat <<EOF >02_infra.yaml
           imports:
           - path: 02_lb_int.py 
           resources: 
           - name: cluster-lb-int
             type: 02_lb_int.py
             properties:
               cluster_network: '${CLUSTER_NETWORK}'
               control_subnet: '${CONTROL_SUBNET}' 
               infra_id: '${INFRA_ID}'
               region: '${REGION}'
               zones: 
               - '${ZONE_0}'
               - '${ZONE_1}'
               - '${ZONE_2}'
           EOF
    2. Create the corresponding GCP object:
    
           gcloud deployment-manager deployments create ${INFRA_ID}-infra --config 02_infra.yaml
           
           
25. We now need to get the Cluster IP. This is basically the loadbalancer IP that we created in the previous step. This IP is used as the Host IP addresses for the record sets that will be put into our private DNS zone. 

        export CLUSTER_IP=(`gcloud compute addresses describe ${INFRA_ID}-cluster-ip --region=${REGION} --format json | jq -r .address`)
        
26. We now create the required record sets for apiserver communication among the cluster nodes.

        if [ -f transaction.yaml ]; then rm transaction.yaml; fi
        gcloud dns record-sets transaction start --zone ocp-private-zone
        gcloud dns record-sets transaction add ${CLUSTER_IP} --name api.${CLUSTER_NAME}.${BASE_DOMAIN}. --ttl 60 --type A --zone ocp-private-zone
        gcloud dns record-sets transaction add ${CLUSTER_IP} --name api-int.${CLUSTER_NAME}.${BASE_DOMAIN}. --ttl 60 --type A --zone ocp-private-zone
        gcloud dns record-sets transaction execute --zone ocp-private-zone
       
27. Let’s export the service account emails as these will be called inside the deployment manager templates of our master & worker nodes. In our demo, we will be using the same service account we created earlier:

        export MASTER_SERVICE_ACCOUNT=(`gcloud iam service-accounts list --filter "email~^ocp-serviceaccount@${PROJECT_NAME}." --format json | jq -r '.[0].email'`)
        export WORKER_SERVICE_ACCOUNT=(`gcloud iam service-accounts list --filter "email~^ocp-serviceaccount@${PROJECT_NAME}." --format json | jq -r '.[0].email'`)
     
     If the above commands do not set the correct service account email ENV variable, you can try running **gcloud iam service-accounts list** , copy the email address for your service account and set it as the environment variable for both MASTER_SERVICE_ACCOUNT & WORKER_SERVICE_ACCOUNT.
     
28. Now let’s create 2 google cloud buckets. One will be for storing the bootstrap.ign file for the bootstrap node, and the other will be the one to store the RedHat Coreos image that the cluster nodes will pull to boot up from.
    1. Bucket to store bootstrap.ign file.
    
           gsutil mb gs://${INFRA_ID}-bootstrap-ignition
           gsutil cp install_dir/bootstrap.ign gs://${INFRA_ID}-bootstrap-ignition/
           
    2. Bucket to store RedHat Coreos file.

           gsutil mb gs://rhcosbucket/
           gsutil cp rhcos-4.10.3-x86_64-gcp.x86_64.tar.gz gs://rhcosbucket/
           export IMAGE_SOURCE=gs://rhcosbucket/rhcos-4.10.3-x86_64-gcp.x86_64.tar.gz
           gcloud compute images create "${INFRA_ID}-rhcos-image" --source-uri="${IMAGE_SOURCE}"
           
29. Let’s set the CLUSTER_IMAGE env to be called later by the node creation commands.

        export CLUSTER_IMAGE=(`gcloud compute images describe ${INFRA_ID}-rhcos-image --format json | jq -r .selfLink`)

30. Let’s set the BOOTSTRAP_IGN env to be called in the next step of bootstrap node creation.
       
        export BOOTSTRAP_IGN=`gsutil signurl -d 1h service-account-key.json gs://${INFRA_ID}-bootstrap-ignition/bootstrap.ign | grep "^gs:" | awk '{print $5}'`

31. Now we create the bootstrap node itself using the relevant Deployment Manager template The below commands will create the bootstrap node, a public IP for it, and an empty instance group for the bastion host:
    1. Create the .yaml file that sets the metadata for the bootstrap node:
    
           cat <<EOF >04_bootstrap.yaml
           imports:
           - path: 04_bootstrap.py

           resources:
           - name: cluster-bootstrap
             type: 04_bootstrap.py
             properties:
               infra_id: '${INFRA_ID}' 
               region: '${REGION}' 
               zone: '${ZONE_0}' 

               cluster_network: '${CLUSTER_NETWORK}' 
               control_subnet: '${CONTROL_SUBNET}' 
               image: '${CLUSTER_IMAGE}' 
               machine_type: 'e2-standard-4' 
               root_volume_size: '128' 

               bootstrap_ign: '${BOOTSTRAP_IGN}' 
           EOF
    2. Run the command to create the google cloud object for the bootstrap node.

           gcloud deployment-manager deployments create ${INFRA_ID}-bootstrap --config 04_bootstrap.yaml
           
        
32. Now we need to manually add the bootstrap node to the new empty instance group and add it as part of the internal load balancer we created earlier. This is mandatory as the initial temporary bootstrap cluster is hosted on the bootstrap node.

        gcloud compute instance-groups unmanaged add-instances ${INFRA_ID}-bootstrap-instance-group --zone=${ZONE_0} --instances=${INFRA_ID}-bootstrap
        gcloud compute backend-services add-backend ${INFRA_ID}-api-internal-backend-service --region=${REGION} --instance-group=${INFRA_ID}-bootstrap-instance-group --instance-group-zone=${ZONE_0}

33. We now proceed with the master nodes creation. Note that we are using instance types e2-standard-4, but you can use any other type which suits you best. However please ensure the type you choose accommodates at least 16GB Memory and 4 vCPUs, as OCP will not be able to run the required containers on lower spec instance types. I have tried many times and it has usually failed or has been unstable.
    1. Set the env variable for the master ignition config file

           export MASTER_IGNITION=`cat install_dir/master.ign`

    2. Create the .yaml file that sets the metadata for the master nodes:

           cat <<EOF >05_control_plane.yaml
           imports:
           - path: 05_control_plane.py

           resources:
           - name: cluster-control-plane
             type: 05_control_plane.py
             properties:
               infra_id: '${INFRA_ID}' 
               zones: 
               - '${ZONE_0}'
               - '${ZONE_1}'
               - '${ZONE_2}'

               control_subnet: '${CONTROL_SUBNET}' 
               image: '${CLUSTER_IMAGE}' 
               machine_type: 'e2-standard-4' 
               root_volume_size: '128'
               service_account_email: '${MASTER_SERVICE_ACCOUNT}' 

               ignition: '${MASTER_IGNITION}' 
           EOF
      
      3. Run the command to create the google cloud object for the master nodes:
    
        gcloud deployment-manager deployments create ${INFRA_ID}-control-plane --config 05_control_plane.yaml
        
34. Once your master nodes have been deployed, we also need to add these to their respective instance groups that were created earlier in the loadbalancer creation step:

        gcloud compute instance-groups unmanaged add-instances ${INFRA_ID}-master-${ZONE_0}-instance-group --zone=${ZONE_0} --instances=${INFRA_ID}-master-0
        gcloud compute instance-groups unmanaged add-instances ${INFRA_ID}-master-${ZONE_1}-instance-group --zone=${ZONE_1} --instances=${INFRA_ID}-master-1
        gcloud compute instance-groups unmanaged add-instances ${INFRA_ID}-master-${ZONE_2}-instance-group --zone=${ZONE_2} --instances=${INFRA_ID}-master-2
        
35. At this point we must now wait for the bootstrap process to complete. You can now monitor the bootstrap process:
    1. ssh into your bootstrap node and run `journalctl -b -f -u release-image.service -u bootkube.service` . Upon bootstrap process completion, the output of this command should stop at a message that looks something like `systemd[1]: bootkube.service: Succeeded`. 
    2. You can also ssh into the master nodes and run a sudo crictl ps to monitor the container creation of the various OCP components. Sometimes the kube-apiserver & etcd related components fluctuate and keep flapping. Do not panic and allow some time for these to stabilize. You can also perform a rolling reboot of your master nodes if you wish to.

36. Once your bootstrap process completes, you can remove your bootstrap components:

        gcloud compute backend-services remove-backend ${INFRA_ID}-api-internal-backend-service --region=${REGION} --instance-group=${INFRA_ID}-bootstrap-instance-group --instance-group-zone=${ZONE_0}
        gsutil rm gs://${INFRA_ID}-bootstrap-ignition/bootstrap.ign
        gsutil rb gs://${INFRA_ID}-bootstrap-ignition
        gcloud deployment-manager deployments delete ${INFRA_ID}-bootstrap

37. Now we are good to create our worker nodes. Note that if you run an oc get co command from your bastion host at this time, you will still see a few operators unavailable, typically the ingress, console, authentication, and a few other cluster operators. This is because they depend on some components which need to come up on the worker nodes. For example, the ingress cluster operator will deploy the router pods on the worker nodes, and only then will a route to your console will be created, and eventually your authentication operator would also reach completion.
    1. Set the env variable for the worker ignition config file.

           export WORKER_IGNITION=`cat install_dir/worker.ign`

    2. Create the .yaml file that sets the metadata for the worker nodes

           cat <<EOF >06_worker.yaml
           imports:
           - path: 06_worker.py

           resources:
           - name: 'worker-0' 
             type: 06_worker.py
             properties:
               infra_id: '${INFRA_ID}' 
               zone: '${ZONE_0}' 
               compute_subnet: '${COMPUTE_SUBNET}' 
               image: '${CLUSTER_IMAGE}' 
               machine_type: 'e2-standard-4' 
               root_volume_size: '128'
               service_account_email: '${WORKER_SERVICE_ACCOUNT}' 
               ignition: '${WORKER_IGNITION}' 
           - name: 'worker-1'
             type: 06_worker.py
             properties:
               infra_id: '${INFRA_ID}' 
               zone: '${ZONE_1}' 
               compute_subnet: '${COMPUTE_SUBNET}' 
               image: '${CLUSTER_IMAGE}' 
               machine_type: 'e2-standard-4' 
               root_volume_size: '128'
               service_account_email: '${WORKER_SERVICE_ACCOUNT}' 
               ignition: '${WORKER_IGNITION}' 
           EOF

     3. Run the command to create the google cloud object for the worker node

            gcloud deployment-manager deployments create ${INFRA_ID}-worker --config 06_worker.yaml

38. With the node deployments created, there should be 2 CSRs in a pending state, we need to approve these. Once you approve the first 2, there will be an additional 2 CSRs generated by those nodes, therefore, 4 CSRs (in sequence of 2 CSRs each) in total that we must approve.

        oc get csr
        oc adm certificate approve <csr name>

39. Now if you run oc get nodes, you should be able to see your worker nodes too. 

### Creating a second Internal Loadbalancer for worker plane traffic

40. We will now create 2 new unmanaged instance groups for each of our worker nodes. This is because later we will be creating another internal load balancer that will forward the worker node specific traffic.
    1. Go to **‘Compute Engine’** > **‘Instance Groups’**
    2. Click on **‘Create Instance Group’** and select **‘New unmanaged instance group’**
    3. Enter the relevant details like name, region & zone. Ensure region & zone is the same as where your worker nodes reside.
    4. For networking options:
       1.	Network: ocp-network (Name of your network that hosts the cluster)
       2. Subnetwork: worker-subnet (needs to be the name of your worker nodes subnet)
       3. VM Instances: Select one of the worker nodes from the drop-down list.
    5. Click **‘Create’**
    6.	Repeat steps i., ii., iii.,iv. & v. for creating a second instance group that will have the second worker node in it.

41. Now we create the other internal load balancer that will have the 2 instance groups we created in the previous step as backends:
    1. Go to **‘Network Services’** > **‘Loadbalancing’**
    2. Click on **‘Create Load Balancer’** > Select **‘TCP load balancing’**
    3. Select the following options:
       1. Internet facing or Internal only: Only Between my VMs
       2. Multiple regions or Single region: Single Region 
       3. Click ‘Continue’
    4. Enter the name & region.
    5. For the network, select your network name to which your VM Instances are connected. (‘ocp-network ‘in our exercise)
    6. For **Backend Configuration**:
       1. Select one of the instance groups we created in the previous step, click Done.
       2. Add another backend by selecting the second instance group we created in the previous step, click Done.
       3.	For the HealthCheck, click on **‘Create a Health Check’** and:
              1. Enter the name e.g  wildcard-apps-healthcheck 
              2. Scope: Regional
              3. Protocol: TCP
              4. Port: 443
              5. Proxy Protocol: None
              6. Logs: Off
              7. Leave the Health Criteria parameters to the default.
              8. Click Save.
    7. For the **Frontend Configuration**:
       1. You can give it a name if you want to, we have left it blank for this exercise.
       2. Subnetwork: master-subnet
       3. Internal IP – Purpose: Shared
              1. IP Address: Ephemeral (Automatic)
       4. Ports: All
       5. Global Access: Disable
       6. Click **Done**.
    8. Click **‘Create’**.
    9. From the loadbalancers list, select the name of the newly created loadbalancer and take note of the Frontend IP address. This will be the IP address that will be used for our new DNS record set in the next step.

42. Now let’s add a new DNS record set for the wildcard of `*.apps.<cluster name>.<base domain>` to our private DNS zone. This DNS record is very important and is required for your ingress cluster operator to be able to listen and receive ingress traffic from the Master nodes. At this time, the console pods & authentication pods on the master nodes try to repeatedly connect with the ingress (router) pods to establish a legit route for their respective path endpoints. Because of the absence of this record set in step 37 above, some of the cluster operators remain unavailable.
    1. Go to **‘Network Services’** > **‘Cloud DNS’**
    2. Select the private DNS zone we created earlier.
    3. Click on **‘Add Record Set’**.
    4. Enter the DNS name as `*.apps` . The clustername & basedomain should get auto-filled.
    5. Set TTL value to 1 and TTL Unit to minutes.
    6. Resource Record Type: A
    7. Routing Policy: Default Record Type
    8. IP Address: 10.1.20.11 (This needs to be the IP of the loadbalancer you created in the previous step)
    9. Click **‘Create’**.
 
 
Once this is done, run a `watch oc get co`, and you should start seeing all your Cluster Operators becoming Available.
 
### Configuring a Reverse Proxy for external UI access
 
43. Now that our cluster is entirely setup, we still cannot access the UI externally since it is a private cluster and only configured to expose it internally within the GCP network. For this, we will configure a reverse proxy on our bastion host:
    1. Install the haproxy package
                
           sudo apt install haproxy -y
  
    2. Open the haproxy.cfg file to make some configuration changes

           vi /etc/haproxy/haproxy.cfg
 
    3. Add the following section in the end:

           frontend localhost
               bind *:80
               bind *:443
               option tcplog
               mode tcp
               default_backend servers
 
           backend servers
               mode tcp
               server hamzacluster.openshift.com 10.1.10.8 # Note that in this last line, the IP address 10.1.10.8 should be replaced by your loadbalancer IP address.
    
    4. Once the haproxy is configured, restart it: systemctl restart haproxy.
    5.	Run an `oc get routes -A` command to get the Console URL through which you can access.
    6. On your local machine, add an entry in the /etc/hosts file (Linux & Mac Users) that points to the Public IP of your bastion host for the name address of console-openshift-console.apps.(clustername).(basedomain) & oauth-openshift.apps.(clustername).(basedomain). For Windows Users, you might have to add the entry into `c:\Windows\System32\Drivers\etc\hosts`.

       Something like :

           <Bastion host Public IP> console-openshift-console.apps.hamzacluster.openshift.com oauth-openshift.apps.hamzacluster.openshift.com

    7. You should now be able to access the OCP console UI through your browser using https://console-openshift-console.apps.(clustername).(basedomain) .

















































