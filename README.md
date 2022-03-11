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
![picture alt] (/Users/hamzawork/Desktop/flowdiagram.png)






