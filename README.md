# Basic 3-Tier Web App (with LB)

## Purpose

Build a load balanced [3-tier](https://en.wikipedia.org/wiki/Multitier_architecture) web application that seperates the web / applicaton and data tiers by placing them in separate sub-networks.

Based on [Solution Tutorials - Highly Available & Scalable Web App](https://console.bluemix.net/docs/tutorials/highly-available-and-scalable-web-application.html#use-virtual-servers-to-build-highly-available-and-scalable-web-app)

This document illustrates the deployment of [WordPress](https://wordpress.com) on top of a [LAMP stack](https://en.wikipedia.org/wiki/LAMP_(software_bundle)) hosted on [IBM Virtual Private Cloud Infrastructure](https://cloud.ibm.com/docs/infrastructure/vpc?topic=vpc-getting-started-with-ibm-cloud-virtual-private-cloud-infrastructure)(VPC). The main objective is to highlight the features of IBM VPC but at the end of this tutorial, a tested and working application environment will be deployed.

Features: 
1. Application 
- A load balanced application - WordPress
- using multiple databases - HyperDB
- with master/slave data replication - MySQL
2. Infrastructure 
- Public isolation - VPC
- where application and data layers are deployed on separate subnets
- with separate network security groups.
- using bring-your-own-IP

Below is the IBM Virtual Private Cloud (VPC) architecture of the solution showing public isolation for both Application (through a Load Balancer) and Data.

## VPC Architecture

![3tier Web App](3TWebAppDrawio.png)

## Assumptions and Limitations

- This document expects the reader to have a basic level of undertanding of network infrastructure and application deployment on a Linux environment.
- The solution will use HTTP.
- The LAMP stack will use (Nginx)[https://www.nginx.com/] Web Appliction Server and [MySQL](https://www.mysql.com/) will be deployed on a separate server.
- Fixes to issues found during the deployment of the environment have been provided. However, these fixes are as of the time of this writting and other issues may occur with new deplyments or versions of the stack.
- Not shown in the architetcure diagram is the use a [public IP](https://en.wikipedia.org/wiki/IP_address) addresses in order to deploy the application. IBM VPC uses a [floating IP and a Public Gateway](https://cloud.ibm.com/docs/infrastructure/vpc-network?topic=vpc-network-about-networking-for-vpc#about-networking-for-vpc) to allow internet traffic. We will use these to access the VSIs and pull the software from public repositories. Once the images are deployed, floating IPs will be removed for improved system isolation.
- Bring-Your-Own-Image (BYOI) not supported at the time of this writting.
- Network storage not supported at the time of this writting.
  
## VPC Functional Coverage
| Function | Result | Notes |
| -------- | ------ | ----- |
| VPC | :white_check_mark: | |
| Subnets | :white_check_mark: | |
| Private IP (BYOIP) | :white_check_mark: | |
| Virtual Server Instance (VSI) | :white_check_mark: | |
| Multiple Network Interfaces in VSI | :white_check_mark: | |
| Load Balancer | :white_check_mark: | |
| Floating IPv4 | :white_check_mark: | :warning: Temporary use to deploy application images |
| Public Gateway | :white_check_mark: | :warning: Temporary use to deploy application images |
| Storage BYOI support (both boot and secondary) | :warning: | Not available - Waiting for support.|

### System Requirements

#### Operating system

| Tier  | Operating system |
| ------------- | ------------- |
| Web Server & Application | Ubuntu 18.04  |
| Data  | Ubuntu 18.04  |

#### Hardware

| Tier | Type | Profile |
| ------------- | ------------- | ------- |
| Web Server and Application  |  VSI | b-4x16 |
| Data| VSI  | b-4x16 |

#### Runtime Services

| Service Name |
| ------- |
| None at this time. |

## Documented Steps
To build this scenario we will first deploy the VPC infrastructure followed by the deployment and configuration of the application. Then, we will build configure an HA application cluster to enable scalability of the application when higher traffic reqires new nodes added to the load ablancer.

## Prerequisites

The following needs to be executed before starting with the deployment:
1. Have access to a public SSH key as described in [SSH Keys](https://cloud.ibm.com/docs/vsi-is?topic=virtual-servers-is-ssh-keys#ssh-keys).
2. Create a new resource group called `VPC1` as described in [Managing resource groups](https://cloud.ibm.com/docs/resources?topic=resources-rgs#rgs)
3. Once the `VPC1` resource group has been created, update User permissions and provide the reqquired access as described in [Managing user permissions for VPC resources](https://cloud.ibm.com/docs/infrastructure/vpc?topic=vpc-managing-user-permissions-for-vpc-resources#managing-user-permissions-for-vpc-resources)

### Deploy VPC Infrastructure

IBM Cloud provides four methods to deploy the VPC infrastructure and three of them are documented here. The reader may follow the instructions using one of these to setup the environment for this scenario.

[Deploy using CLI](CLI.md)

[Deploy using API](API.md)

[Deploy using UI](UI.md)

### Install Web Application

Deploy the application once the VPC infrastructure has been deployed.

[Install Application Layer](WebApp.md)

## Error Scenarios

Application layer failures are included during the deployment and test of the software stack. No infrastructure failures were introduced.

## Documentation Provided

Useful links for VPC documentation.

[Getting started with IBM Cloud Virtual Private Cloud](https://cloud.ibm.com/docs/infrastructure/vpc/getting-started.html#getting-started-with-ibm-cloud-virtual-private-cloud)

[Assigning role-based access to VPC resources](https://cloud.ibm.com/docs/infrastructure/vpc/assigning-role-based-access-to-vpc-resources)

[IBM Cloud CLI for VPC Reference](https://cloud.ibm.com/docs/infrastructure-service-cli-plugin)

[Regional API for VPC](https://cloud.ibm.com/apidocs/rias)

[IBM Cloud Virtual Private Cloud API error messages](https://cloud.ibm.com/docs/infrastructure/vpc?topic=vpc-rias-error-messages#rias-error-messages)
## Observations
None.
