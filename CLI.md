## Purpose

This section provides the steps to create a VPC and the required resources for the scenario described in [*Basic 3-Tier Web App (with LB)*](README.md).

For this section, the IBM Cloud CLI (Command-Line Interface) will be used.

## Prerequisites

0. Any prerequisites mentioned in the *Basic 3-Tier Web App (with LB)* main page like providing required user access.
1. Install the [IBM Cloud CLI](https://cloud.ibm.com/docs/cli?topic=cli-getting-started)
2. Install the vpc-infrastructure plugin using the following CLI command:
   ```
   ibmcloud plugin install vpc-infrastructure
   ```
   Note: CLI commands noted in this document were executed using `version 0.3.2` of the __*vpc-infrastructure*__ plugin. You may list the installed plugins using the following command:
   ```
   ibmcloud plugin list
   ```
   Result
   ```
   Listing installed plug-ins...

   Plugin Name                            Version   Status
   container-registry                     0.1.368   Update Available
   container-service/kubernetes-service   0.1.668   Update Available
   dev                                    2.1.12    Update Available
   vpc-infrastructure                     0.3.2
   sdk-gen                                0.1.12
   cloud-functions/wsk/functions/fn       1.0.27    Update Available
   ```
   After plugin installation, vpc-infrastructure commands can be executed with `ibmcloud is`. The syntax for each of these commands is defined in [IBM Cloud CLI for VPC Reference](https://cloud.ibm.com/docs/vpc?topic=vpc-creating-a-vpc-using-cli).

   Note that the syntax of a command may be updated on a new release of the vpc-infrastructure. However, there may be some delays to update the online documentation. The following command can be executed from the command line to obtain its syntax:

      `ibmcloud is help <command>`. Where `<command>` is the specific vpc-infrastructure command.

## Activities executed to setup the VPC environment

For an overview of IBM Virtual Private Cloud (VPC), please refer to [About VPC](https://cloud.ibm.com/docs/vpc?topic=vpc).

1. Create an SSH key to be used when a virtual instance (VSI) resource is created.
2. Create a VPC.
3. Create Address Prefixes (CIDR) for the VPC.
4. Create Subnets.
5. Choose a profile and an image to create a Virtual Server Instance (VSI)
6. Create Security Groups
7. Create VPC VSIs.
8. Create and configure a Load Balancer.
9. Create Floating IPs and assign them to the VSIs.
10. Create a Public Gateway.
11. Add rules to Security Groups.

Once the above steps are completed, the VPC infrastructure will be ready for the next activities.

## Login to IBM Cloud

For a federated account use single sign on:  
   `$ ibmcloud login -sso`  
Otherwise use the default login:  
   `$ ibmcloud login`  
If you have an API Key, use --apikey:  
   `$ ibmcloud login --apikey [your API Key]`  

# Deploy VPC Infrastructure   

## Set Resource Group, Region and Zone

Resources in IBM Cloud are assigned to a Resource Group. In our case, we want to use resource group **VPC1** that was created previously. In addition, we will allocate the resources in the **us-south** region.

For more information on Regions and Zones please refer to [Creating a VPC in a different region](https://cloud.ibm.com/docs/vpc?topic=vpc-creating-a-vpc-in-a-different-region).

### Set Resource Group and Zone

After login, the target environment will set to the account's defaults. This means that any new VPC resources will be assigned the account's __*default*__ Resource Group.  

In our case we want to use resource group **VPC1**, created previously, and locate the VPC resources in the **us-south** region.

Use the `ibmcloud target` command to select the desired resource group and region for the VPC.  
```
ibmcloud target  -g VPC1 -r us-south
```
Result
```
Targeted resource group VPC1

Switched to region us-south


API endpoint:      https://api.ng.bluemix.net
Region:            us-south
User:              msalas@us.ibm.com
Account:           Phillip Trent's Account (843f59bad5553123f46652e9c43f9e89) <-> 1691265
Resource group:    VPC1
CF API endpoint:
Org:
Space:
```
Now that we are in `us-south` region, let's find out what zones are available using the zones command.
```
ibmcloud is zones us-south
```
Result
```
Listing zones in region us-south under account Phillip Trent's Account as user pltrent@us.ibm.com...
Name         Region     Status   
us-south-3   us-south   available   
us-south-1   us-south   available   
us-south-2   us-south   available
```
**NOTE**: All resources will be created in zone `us-south-1` throughout this use case.

We will use [environment variables](https://stackoverflow.com/questions/3035427/how-to-create-a-new-environment-variable-in-unix) throughout this use case to reference values and facilitate copy/paste activities.

Since we do not need to keep these permanently, we will store them in file `.vpc_ids` and execute `source .vpc_ids` as new entries are added to the file. This will allow you to restore these if you close your session and/or wish to continue at a later time. Only those IDs needed for this use case will be saved and highlighted in the documentation as follows:

- Environment variable: `ZONE=us-south-1`

To verify that this variable was saved, execute `echo $ZONE` and make sure the response is not empty.

### Identify Resource Group ID

A Resource Group ID is required by most CLI calls. The Resource Group used by most CLIs will be `VPC1` (previously set with `target`). However, some CLI commands require the object ID of the Resource Group. Issue the following cURL command to get the list of Resource Groups and identify the ID for `VPC1`:
```
ibmcloud resource groups
```
Result
```
Retrieving all resource groups under account Phillip Trent's Account as msalas@us.ibm.com...
OK
Name           ID                                 Default Group   State
default        00d24065a2ec44efb9de172e6d19b919   true            ACTIVE
VPC1           594a009f2d4b4128ad1f25b55c991de0   false           ACTIVE
```
- Environment variable: `RESOURCE_GROUP=594a009f2d4b4128ad1f25b55c991de0`

### About Resource IDs

Objects in IBM Cloud are assigned a unique object ID. This is important because several *CLI* commands require an object ID representing a resource.

In the above resource groups, `VPC1` has been assigned ID `594a009f2d4b4128ad1f25b55c991de0`.

After creating each resource, we will keep the ID using __*environment variables*__ for later use. For example, you will need a Subnet ID to create a resource on that subnet, an SSH Key ID to create a Virtual Server Instance (VSI), and so on.

**Note**: If at any point you encounter an error calling an API, first verify the environment variable is correct for the resource you are attempting to update. If an ID is misplaced or was saved with an incorrect value, You can always use a CLI command to list the details of a resource. For example,
```
ibmcloud is instances
```
will give the list of all the VSIs and their IDs. Then you can use `<object_id>`
```
ibmcloud is instance <object_id>
```
to get the details of the specific VSI.

## Create an SSH Key

An SSH key is required when creating a VPC instance. We will use a public key previously created (see prerequisites above).

Copy the SSH public key you wish to use to file `vpc-key.pub` and call the key-create command to load it to the VPC environment. Optionally, you can just use the public key directly (.ssh/id_rsa.pub).

Syntax: [Import an RSA public key](https://cloud.ibm.com/docs/vpc?topic=vpc-creating-a-vpc-using-cli#add-ssh-key-cli)

Create an SSH key named `vpc-key`
```
ibmcloud is key-create vpc-key @vpc-key.pub
```
Result
```
Creating key vpc-key under account Phillip Trent's Account as user msalas@us.ibm.com...

ID            636f6d70-0000-0001-0000-000000156431
Name          vpc-key
Type          rsa
Length        2048
FingerPrint   SHA256:ZAmojbQXSEbPnqkUY2Hp4r8d/vwlrEWsJXtB5sKBYs0
Key           ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABAQCwawN5NFHzyEHxS2NOOYUR2YkiKGpL6+axsQm2sTjlhyqE1NF2k+NsY2QgeMb1YbNqwrclLYy6yitDWqVebJCPKHntpm/J85S4Oup8C3kz+elu3dpdJM8RR2VSoA6qUkFfS9bmL3cucPtbOHYHcMhC7m7lVmwOFQ4pTOcfX85yS7l6B9m9sawJBKomLwJpRJsRVOgYh0C3jWApDt21SVGRK5HUBOob3xtcBfPCDvb4I0IfzbsgidUKHy4iRax88oWnmwJm5G9MNpgU4u10ly2a/vUfxzGQhHmDn5O7cPg2sLhIVrEXr1uAYQG3N/Es0GKF4AvEEw4sQpNlVp2ZLmkl msalas@MSMBA2013.local

Created       now
```
- Environment variable: `SSH_KEY=636f6d70-0000-0001-0000-000000156431`

## Create VPC

Syntax: [Create a VPC](https://cloud.ibm.com/docs/vpc?topic=vpc-creating-a-vpc-using-cli)

Create a VPC named `wp_vpc`.

```
ibmcloud is vpc-create wp_vpc
```
Result
```
Creating vpc wp_vpc in resource group VPC1 under account Phillip Trent's Account as user msalas@us.ibm.com...

ID                       4c69d7fe-9407-48e3-9855-dc27f595d321
Name                     wp_vpc
Classic Access           no
Default Network ACL      allow-all-network-acl-4c69d7fe-9407-48e3-9855-dc27f595d321(231ed3f3-d4fc-4613-9d8c-851aeb627f46)
Default Security Group   football-staring-satiable-goldfish(2d364f0a-a870-42c3-a554-000001537989)
Resource Group           (594a009f2d4b4128ad1f25b55c991de0)
Created                  3 seconds ago
Status                   available
```

- Environment variable: `VPC=4c69d7fe-9407-48e3-9855-dc27f595d321`

## Create Address Prefixes

For more information on address prefixes, please refer to [Understanding IP address ranges, address prefixes, regions, and subnets](https://cloud.ibm.com/docs/vpc?topic=vpc-vpc-addressing-plan-design).

Syntax: [Create an address prefix](https://cloud.ibm.com/docs/vpc?topic=vpc-infrastructure-cli-plugin-vpc-reference#vpc-cli-ref)

Create address prefixes for `10.10.11.0/24` and `10.10.12.0/24`.

**Prefix = cidr1**
```
ibmcloud is vpc-address-prefix-create cidr1 $VPC $ZONE 10.10.11.0/24
```
Result
```
Creating address prefix cidr1 of vpc 4c69d7fe-9407-48e3-9855-dc27f595d321 under account Phillip Trent's Account as user msalas@us.ibm.com...

ID            71a7cb69-7369-4136-8cdf-edeb9e70475a
Name          cidr1
CIDR Block    10.10.11.0/24
Zone          us-south-1
Has Subnets   no
Is Default    no
Created       now
```
**Prefix = cidr2**
```
ibmcloud is vpc-address-prefix-create cidr2 $VPC $ZONE 10.10.12.0/24
```
Result
```
Creating address prefix cidr2 of vpc 4c69d7fe-9407-48e3-9855-dc27f595d321 under account Phillip Trent's Account as user msalas@us.ibm.com...

ID            a7a9dfd0-9301-4441-8c65-4d76f2937bf0
Name          cidr2
CIDR Block    10.10.12.0/24
Zone          us-south-1
Has Subnets   no
Is Default    no
Created       now
```

## Create Two VPC Subnets

Create two VPC Subnets for ipv4-cidr-blocks `10.10.11.0/24` and `10.10.12.0/24`.  

The __*application*__ tier will be `subnet1` and the __*data*__ tier will be `subnet2`.

Syntax: [Create a subnet](https://cloud.ibm.com/docs/vpc?topic=vpc-infrastructure-cli-plugin-vpc-reference#subnet-create)

**Subnet1**
```
ibmcloud is subnet-create subnet1 $VPC $ZONE --ipv4-cidr-block 10.10.11.0/24
```
Result
```
Creating Subnet subnet1 under account Phillip Trent's Account as user msalas@us.ibm.com...

ID                  6e422da5-25d7-49aa-aaad-5a6e12c47a6a
Name                subnet1
IPv*                ipv4
IPv4 CIDR           10.10.11.0/24
IPv6 CIDR           -
Address Available   251
Address Total       256
ACL                 allow-all-network-acl-4c69d7fe-9407-48e3-9855-dc27f595d321(231ed3f3-d4fc-4613-9d8c-851aeb627f46)
Gateway             -
Created             1 second ago
Status              pending
Zone                us-south-1
VPC                 wp_vpc(4c69d7fe-9407-48e3-9855-dc27f595d321)
```

- Environment variable: `SUBNET1=6e422da5-25d7-49aa-aaad-5a6e12c47a6a`

**Subnet2**
```
ibmcloud is subnet-create subnet2 $VPC $ZONE --ipv4-cidr-block 10.10.12.0/24
```
Result
```
ID                  143f4dd6-fc2f-4404-b913-392fe4e11ffa
Name                subnet2
IPv*                ipv4
IPv4 CIDR           10.10.12.0/24
IPv6 CIDR           -
Address Available   251
Address Total       256
ACL                 allow-all-network-acl-4c69d7fe-9407-48e3-9855-dc27f595d321(231ed3f3-d4fc-4613-9d8c-851aeb627f46)
Gateway             -
Created             1 second ago
Status              pending
Zone                us-south-1
VPC                 wp_vpc(4c69d7fe-9407-48e3-9855-dc27f595d321)
```

- Environment variable: `SUBNET2=143f4dd6-fc2f-4404-b913-392fe4e11ffa`

The initial status of a newly created subnet is set to __*pending*__.  You must wait until the subnet status is available before assigning any resources to it.

To check the subnet status, display the subnet details.  Keep checking until the status is set to available.

```
ibmcloud is subnet $SUBNET1
```
Result
```
Getting Subnet 6e422da5-25d7-49aa-aaad-5a6e12c47a6a under account Phillip Trent's Account as user msalas@us.ibm.com...

ID                  6e422da5-25d7-49aa-aaad-5a6e12c47a6a
Name                subnet1
IPv*                ipv4
IPv4 CIDR           10.10.11.0/24
IPv6 CIDR           -
Address Available   251
Address Total       256
ACL                 allow-all-network-acl-4c69d7fe-9407-48e3-9855-dc27f595d321(231ed3f3-d4fc-4613-9d8c-851aeb627f46)
Gateway             -
Created             1 minute ago
Status              available
Zone                us-south-1
VPC                 wp_vpc(4c69d7fe-9407-48e3-9855-dc27f595d321)
```

## VPC Instance Profiles and Images

Before continuing we must select an instance profile and image for our VPC instances.  
- The profile describes the instance size in terms of CPUs and memory.  To see a list of supported profiles use the `instance-profiles` command.
- The image is the operating system that will be loaded into the instance. To see a list of available images use the `images` command.

We will use the `b-4x16` balanced profile for all our instances, which is 4 CPUs and 16G of memory.  For OS image, the `ubuntu-18.04-amd64` which is Ubuntu Linux (18.04 LTS Bionic Beaver Minimal Install).

**List instance profiles**
```
ibmcloud is instance-profiles
```
Result
```
Listing server profiles under account Phillip Trent's Account as user pltrent@us.ibm.com...
Name       Family   
m-62x496   memory   
b-62x248   balanced   
c-2x4      cpu   
b-4x16     balanced   
b-16x64    balanced   
c-16x32    cpu   
b-32x128   balanced   
m-4x32     memory   
m-2x16     memory   
b-2x8      balanced   
m-16x128   memory   
c-8x16     cpu   
c-4x8      cpu   
m-8x64     memory   
b-48x192   balanced   
c-32x64    cpu   
b-8x32     balanced   
m-32x256   memory
```
**List Images**
```
ibmcloud is images
```
Result
```
Listing images under account Phillip Trent's Account as user pltrent@us.ibm.com...
ID                                     Name                    Format   OS                                                        Arch    Created        Status   Visibility   
cc8debe0-1b30-6e37-2e13-744bfb2a0c11   centos-7.x-amd64        -        CentOS (7.x - Minimal Install)                            amd64   2 months ago   READY    public   
660198a6-52c6-21cd-7b57-e37917cef586   debian-8.x-amd64        -        Debian GNU/Linux (8.x jessie/Stable - Minimal Install)    amd64   2 months ago   READY    public   
e15b69f1-c701-f621-e752-70eda3df5695   debian-9.x-amd64        -        Debian GNU/Linux (9.x Stretch/Stable - Minimal Install)   amd64   2 months ago   READY    public   
7eb4e35b-4257-56f8-d7da-326d85452591   ubuntu-16.04-amd64      -        Ubuntu Linux (16.04 LTS Xenial Xerus Minimal Install)     amd64   2 months ago   READY    public   
cfdaf1a0-5350-4350-fcbc-97173b510843   ubuntu-18.04-amd64      -        Ubuntu Linux (18.04 LTS Bionic Beaver Minimal Install)    amd64   2 months ago   READY    public   
b45450d3-1a17-2226-c518-a8ad0a75f5f8   windows-2012-amd64      -        Windows Server (2012 Standard Edition)                    amd64   2 months ago   READY    public   
81485856-df27-93b8-a838-fa28a29b3b04   windows-2012-r2-amd64   -        Windows Server (2012 R2 Standard Edition)                 amd64   2 months ago   READY    public
```

- Environment variable: `UBUNTU=cfdaf1a0-5350-4350-fcbc-97173b510843`

## Security Groups and Access Control Lists

For purposes of this use case, we will create two security groups for application and data servers. For more information on security groups, please refer to [Security in your IBM Cloud VPC](https://cloud.ibm.com/docs/vpc?topic=vpc-using-security-groups).

Syntax: [Create a security group](https://cloud.ibm.com/docs/vpc?topic=vpc-infrastructure-cli-plugin-vpc-reference#security-group-create)

**Application Security Group - app_sg**
```
ibmcloud is security-group-create app_sg $VPC
```
Result
```
Creating security group app_sg in resource group VPC1 under account Phillip Trent's Account as user msalas@us.ibm.com...

ID               2d364f0a-a870-42c3-a554-000001538227
Name             app_sg
Created          now
VPC              wp_vpc(4c69d7fe-9407-48e3-9855-dc27f595d321)
Resource Group   -
```
- Environment variable: `APP_SG=2d364f0a-a870-42c3-a554-000001538227`

**Data Security Group - data_sg**
```
ibmcloud is security-group-create data_sg $VPC
```
Result
```
Creating security group data_sg in resource group VPC1 under account Phillip Trent's Account as user msalas@us.ibm.com...

ID               2d364f0a-a870-42c3-a554-000001538067
Name             data_sg
Created          now
VPC              wp_vpc(4c69d7fe-9407-48e3-9855-dc27f595d321)
Resource Group   -
```
- Environment variable: `DATA_SG=2d364f0a-a870-42c3-a554-000001538067`

## Create Data Tier VPC Instances - Subnet2

Now we have all the required information, let's create two Ubuntu 18.04 VSIs in `subnet2` for the MySQL backend.

Syntax: [Create a server instance](https://cloud.ibm.com/docs/vpc?topic=vpc-infrastructure-cli-plugin-vpc-reference#instance-create)

**Instance = MySQL1**
```
ibmcloud is instance-create mysql1 $VPC $ZONE b-4x16 $SUBNET2 1000 --image-id $UBUNTU --key-ids $SSH_KEY --security-group-ids $DATA_SG
```
Result
```
Creating instance mysql1 under account Phillip Trent's Account as user msalas@us.ibm.com...

ID                       ee4c3eb6-ad47-4449-b8cc-39569d109de9
Name                     mysql1
Profile                  b-4x16
CPU Arch                 amd64
CPU Cores                4
CPU Frequency            2000
Memory                   16

Primary Interface        primary(1fb48864-b95b-4bc3-ac02-a05def60ba3e)
Primary Address          10.10.12.12
Attached Floating IP:    No Floating IP attached

Image                    ubuntu-18.04-amd64(cfdaf1a0-5350-4350-fcbc-97173b510843)
Status                   pending
Created                  11 seconds ago
VPC                      wp_vpc(4c69d7fe-9407-48e3-9855-dc27f595d321)
Zone                     us-south-1
```

- Environment variable: `MYSQL1=ee4c3eb6-ad47-4449-b8cc-39569d109de9`
- Environment variable: `MYSQL1_NIC=1fb48864-b95b-4bc3-ac02-a05def60ba3e`


**Instance = MySQL2**
```
ibmcloud is instance-create mysql2 $VPC $ZONE b-4x16 $SUBNET2 1000 --image-id $UBUNTU --key-ids $SSH_KEY --security-group-ids $DATA_SG
```
Result
```
Creating instance mysql2 under account Phillip Trent's Account as user msalas@us.ibm.com...

ID                       7a4056d6-7527-43a7-9424-3e6088842eee
Name                     mysql2
Profile                  b-4x16
CPU Arch                 amd64
CPU Cores                4
CPU Frequency            2000
Memory                   16

Primary Interface        primary(c5c462b8-8aad-4f31-b6e7-45a6333ac489)
Primary Address          10.10.12.14
Attached Floating IP:    No Floating IP attached

Image                    ubuntu-18.04-amd64(cfdaf1a0-5350-4350-fcbc-97173b510843)
Status                   pending
Created                  6 seconds ago
VPC                      wp_vpc(4c69d7fe-9407-48e3-9855-dc27f595d321)
Zone                     us-south-1
```

- Environment variable: `MYSQL2=7a4056d6-7527-43a7-9424-3e6088842eee`
- Environment variable: `MYSQL2_NIC=c5c462b8-8aad-4f31-b6e7-45a6333ac489`

## Create Web and Application tier VPC instances - Subnet1

Next, create two Ubuntu VSIs in `subnet1` for the application tier.

In this case we will use a json file to pick-up values for the creation of a second ethernet interface to connect to resources in `subnet2` where MySQL servers will be located.

The CLI syntax to be used in this case will include `--network-interface @jsonfilename.json` to refer to the values stored in the json file. For example,

`ibmcloud is instance-create [...] --network-interface @jsonfilename.json`

(Note that some entries were omitted with [`...`] for illustration purposes).

 In our scenario we will use file `appeth1.json` which contains the required data to create the secondary network interface (since this is a json, we will use the object IDs in the file).
 ```
 [
    {
        "port_speed": 1000,
        "name": "eth1",
        "subnet": {
            "id": "143f4dd6-fc2f-4404-b913-392fe4e11ffa"
        },
        "security_groups": [
            {
                "id": "2d364f0a-a870-42c3-a554-000001538067"
            }
        ]
    }
]
```
 The file contains variables for port speed (1000), subnet ID (subnet2) and security group ID (data_sg). The JSON file requires the actual object ID instead of environment variables $SUBNET2 and $DATA_SG.

 **A sample of this file is available [here](appeth1.json)**. (Replace the ID values with your own).

**Instance = AppServ1**
```
ibmcloud is instance-create appserv1 $VPC $ZONE b-4x16 $SUBNET1 1000 --image-id $UBUNTU --key-ids $SSH_KEY --security-group-ids $APP_SG --network-interface @appeth1.json
```
Result
```
Creating instance appserv1 under account Phillip Trent's Account as user msalas@us.ibm.com...

ID                       6c3eef2c-039b-4694-8f81-93318fbe46a6
Name                     appserv1
Profile                  b-4x16
CPU Arch                 amd64
CPU Cores                4
CPU Frequency            2000
Memory                   16

Primary Interface        primary(b69b180e-26d8-447f-86f2-503be7165fc9)
Primary Address          10.10.11.13
Attached Floating IP:    No Floating IP attached


Additional Interface     eth1(bb0bcb0a-4bd5-482f-9491-91eda0e582e6)
Address                  10.10.12.8
Attached Floating IP:    No Floating IP attached

Image                    ubuntu-18.04-amd64(cfdaf1a0-5350-4350-fcbc-97173b510843)
Status                   pending
Created                  6 seconds ago
VPC                      wp_vpc(4c69d7fe-9407-48e3-9855-dc27f595d321)
Zone                     us-south-1
```

- Environment variable: `APPSERV1=6c3eef2c-039b-4694-8f81-93318fbe46a6`
- Environment variable: `APPSERV1_NIC0=b69b180e-26d8-447f-86f2-503be7165fc9`
- Environment variable: `APPSERV1_NIC1=bb0bcb0a-4bd5-482f-9491-91eda0e582e6`
- Environment variable: `APPSERV1_IP=10.10.11.13`

**Instance = AppServ2**
```
ibmcloud is instance-create appserv2 $VPC $ZONE b-4x16 $SUBNET1 1000 --image-id $UBUNTU --key-ids $SSH_KEY --security-group-ids $APP_SG --network-interface @appeth1.json
```
Result
```
Creating instance appserv2 under account Phillip Trent's Account as user msalas@us.ibm.com...

ID                       802369d8-6070-407f-af1f-8d42c0254286
Name                     appserv2
Profile                  b-4x16
CPU Arch                 amd64
CPU Cores                4
CPU Frequency            2000
Memory                   16

Primary Interface        primary(39ccf7b5-abca-4239-bc76-ddbc4ed4f1bb)
Primary Address          10.10.11.9
Attached Floating IP:    No Floating IP attached


Additional Interface     eth1(cf348a7e-26ab-457a-a542-dc178a6a6273)
Address                  10.10.12.5
Attached Floating IP:    No Floating IP attached

Image                    ubuntu-18.04-amd64(cfdaf1a0-5350-4350-fcbc-97173b510843)
Status                   pending
Created                  8 seconds ago
VPC                      wp_vpc(4c69d7fe-9407-48e3-9855-dc27f595d321)
Zone                     us-south-1
```

- Environment variable: `APPSERV2=802369d8-6070-407f-af1f-8d42c0254286`
- Environment variable: `APPSERV2_NIC0=39ccf7b5-abca-4239-bc76-ddbc4ed4f1bb`
- Environment variable: `APPSERV2_NIC1=cf348a7e-26ab-457a-a542-dc178a6a6273`
- Environment variable: `APPSERV1_IP=10.10.11.9`

## Create Web Tier VPC Instance

In this section we will create and configure a VPC load balancer for the web application tier. For more information on configuration of load Balancers (listeners, back-end pools, etc.) see [Using Load Balancers for VPC](https://cloud.ibm.com/docs/vpc?topic=vpc-nlb-vs-elb)

### Create the Load Balancer

Create a `public` load balancer `lb1` on `subnet1`.

Syntax: [Create a Load Balancer](https://cloud.ibm.com/docs/vpc?topic=vpc-infrastructure-cli-plugin-vpc-reference#load-balancer-create)

**Load Balancer = LB1**
```
ibmcloud is load-balancer-create lb1 public --subnet $SUBNET1 --resource-group-id $RESOURCE_GROUP
```
Result
```
Creating load balancer lb1 in resource group VPC1 under account Phillip Trent's Account as user msalas@us.ibm.com...

ID                 10b3514b-d17a-40ef-8ca0-28de83552baf
Name               lb1
Created            2 seconds ago
Host Name          10b3514b-us-south.lb.appdomain.cloud
Is Public          yes
Listeners
Operating Status   offline
Pools
Private IPs        -
Provision Status   create_pending
Public IPs         -
Subnets            6e422da5-25d7-49aa-aaad-5a6e12c47a6a
Resource Group     594a009f2d4b4128ad1f25b55c991de0
```

- Environment variable: `LB1=10b3514b-d17a-40ef-8ca0-28de83552baf`

__NOTE: Before proceeding with the configuration step, wait until the operating status of the load balancer is set to online. *This may take a couple of minutes*__

You can verify the load balancer is online with the following command:
```
ibmcloud is load-balancer $LB1
```
Result
```
Getting load balancer 10b3514b-d17a-40ef-8ca0-28de83552baf under account Phillip Trent's Account as user msalas@us.ibm.com...

ID                 10b3514b-d17a-40ef-8ca0-28de83552baf
Name               lb1
Created            10 minutes ago
Host Name          10b3514b-us-south.lb.appdomain.cloud
Is Public          yes
Listeners
Operating Status   online
Pools
Private IPs        10.10.11.7,10.10.11.6
Provision Status   active
Public IPs         169.61.244.26,169.61.244.157
Subnets            6e422da5-25d7-49aa-aaad-5a6e12c47a6a
Resource Group     594a009f2d4b4128ad1f25b55c991de0
```

### Configure the Load Balancer

Configuring the load balancer involves creating a pool, pool members and a listener that points to our application servers.

**Note**: You may need to wait for each activity to complete (status change from `update pending` to `active`) before continuing to the next activity.

**Create Pool**

Create load balancer `Pool1` for `http` protocol using a `round-robin` method and health checks every `20 seconds`.

Syntax: [Create a load balancer pool](https://cloud.ibm.com/docs/vpc?topic=vpc-infrastructure-cli-plugin-vpc-reference#load-balancer-pool-create)
```
ibmcloud is load-balancer-pool-create pool1 $LB1 round_robin http 20 3 5 http
```
Result
```
Creating pool pool1 of load balancer 10b3514b-d17a-40ef-8ca0-28de83552baf under account Phillip Trent's Account as user msalas@us.ibm.com...

ID                                4b0c3da2-f990-4112-a3fd-9263d511ba6a
Name                              pool1
Algorithm                         round_robin
Health Monitor Delay              20
Health Monitor Retries            3
Health Monitor Timeout            5
Health Monitor Type               http
Health Monitor URL                /
Protocol                          http
Session Persistence Type          -
Session Persistence Cookie Name   -
Members
Provision Status                  active
Created                           now
```

- Environment variable: `POOL1=4b0c3da2-f990-4112-a3fd-9263d511ba6a`

**Add Pool Members**

Add a pool member for each application server.  In our case we will have two pool members: `AppServ1` and `AppServ2`. Port `80` will be used to communicate with he servers.

Syntax: [Create a load balancer pool member](https://cloud.ibm.com/docs/vpc?topic=vpc-infrastructure-cli-plugin-vpc-reference#load-balancer-pool-member-create)

**Pool member = 10.10.11.13 (AppServ1)**
```
ibmcloud is load-balancer-pool-member-create $LB1 $POOL1 80 $APPSERV1_IP
```
Result
```
Creating member of pool 4b0c3da2-f990-4112-a3fd-9263d511ba6a under account Phillip Trent's Account as user msalas@us.ibm.com...

ID                 57eefc4f-b8e2-408f-ac1f-cd6f60a08b3e
Port               80
Target Address     10.10.11.13
Weight             50
Health             unknown
Created            1 second ago
Provision Status   create_pending
```
**Pool member = 10.10.11.9 (AppServ2)**
```
ibmcloud is load-balancer-pool-member-create $LB1 $POOL1 80 $APPSERV2_IP
```
Result
```
Creating member of pool 4b0c3da2-f990-4112-a3fd-9263d511ba6a under account Phillip Trent's Account as user msalas@us.ibm.com...

ID                 59bc76b0-7b7b-4ad0-a17a-038a8b31b407
Port               80
Target Address     10.10.11.9
Weight             50
Health             unknown
Created            1 second ago
Provision Status   create_pending
```

**Add Listener**

Add a public front-end `http` listener for our web application using port `80` and assign it to back-end pool `Pool1`

Syntax: [Create a load balancer listener](https://cloud.ibm.com/docs/vpc?topic=vpc-infrastructure-cli-plugin-vpc-reference#load-balancer-listener-create)

```
ibmcloud is load-balancer-listener-create $LB1 80 http --default-pool $POOL1
```
Result
```
Creating listener of load balancer 10b3514b-d17a-40ef-8ca0-28de83552baf under account Phillip Trent's Account as user msalas@us.ibm.com...

ID                     f8a10899-b648-40a4-823d-d7335ea40370
Certificate Instance   -
Connection Limit       0
Port                   80
Protocol               http
Default Pool           4b0c3da2-f990-4112-a3fd-9263d511ba6a
Provision Status       create_pending
Created                1 second ago
```

**Note**: Load Balancer health checks will fail until the application is installed in section [Install and Configure Application Software](WebApp.md).

## Prepare to Load Application Software

Because custom images are not supported (Bring-Your-Own-Image), we will enable access to the internet for each VPC instance so we can download the required application software. Since the VSIs are isolated from the internet, a floating IPs will be used to temporarily gain access. Once the application software has been installed, internet access will be disabled.

**Create Public IPs**

Reserve and associate a floating IP address to enable each instance to be reachable from the internet.

Syntax: [Reserve a floating IP](https://cloud.ibm.com/docs/vpc?topic=vpc-infrastructure-cli-plugin-vpc-reference#floating-ip-reserve)

**FIP = app1fip**
```
ibmcloud is floating-ip-reserve app1fip --zone $ZONE
```
Result
```
Creating floating IP app1fip in resource group VPC1 under account Phillip Trent's Account as user msalas@us.ibm.com...

ID               bfbc369a-8d67-4900-a44c-739e28a17a6d
Address          169.61.244.19
Name             app1fip
Target           -
Target Type      -
Created          1 second ago
Status           pending
Zone             us-south-1
Resource Group   (594a009f2d4b4128ad1f25b55c991de0)
```

- Environment variable: `APP1FIP=bfbc369a-8d67-4900-a44c-739e28a17a6d`

**FIP = app2fip**
```
ibmcloud is floating-ip-reserve app2fip --zone $ZONE
```
Result
```
Creating floating IP app2fip in resource group VPC1 under account Phillip Trent's Account as user msalas@us.ibm.com...

ID               617bc7f7-4d62-4f67-9991-4378594fbf59
Address          169.61.244.186
Name             app2fip
Target           -
Target Type      -
Created          2 seconds ago
Status           pending
Zone             us-south-1
Resource Group   (594a009f2d4b4128ad1f25b55c991de0)
```

- Environment variable: `APP2FIP=617bc7f7-4d62-4f67-9991-4378594fbf59`

**FIP = data1fip**
```
ibmcloud is floating-ip-reserve data1fip --zone $ZONE
```
Result
```
Creating floating IP data1fip in resource group VPC1 under account Phillip Trent's Account as user msalas@us.ibm.com...

ID               57136ac1-ceb1-491f-a175-b37c6bb0d8b4
Address          169.61.244.91
Name             data1fip
Target           -
Target Type      -
Created          2 seconds ago
Status           pending
Zone             us-south-1
Resource Group   (594a009f2d4b4128ad1f25b55c991de0)
```

- Environment variable: `DATA1FIP=57136ac1-ceb1-491f-a175-b37c6bb0d8b4`

**FIP = data2fip**
```
ibmcloud is floating-ip-reserve data2fip --zone $ZONE
```
Result
```
Creating floating IP data2fip in resource group VPC1 under account Phillip Trent's Account as user msalas@us.ibm.com...

ID               ed744792-ecea-4d70-97bf-8e07bae38936
Address          169.61.244.105
Name             data2fip
Target           -
Target Type      -
Created          1 second ago
Status           pending
Zone             us-south-1
Resource Group   (594a009f2d4b4128ad1f25b55c991de0)
```

- Environment variable: `DATA2FIP=ed744792-ecea-4d70-97bf-8e07bae38936`

**Assign Public IPs to VSIs**

Add a reserved IP address to each VPC instance's primary interface (obtained when each server was created).

Syntax: [Associate a floating IP with a network interface](https://cloud.ibm.com/docs/vpc?topic=vpc-infrastructure-cli-plugin-vpc-reference#instance-network-interface-floating-ip-add)

**Associate app1fip to instance AppServ1**
```
ibmcloud is instance-network-interface-floating-ip-add $APPSERV1 $APPSERV1_NIC0 $APP1FIP
```
Result
```
Creating floatingip bfbc369a-8d67-4900-a44c-739e28a17a6d for instance 6c3eef2c-039b-4694-8f81-93318fbe46a6 under account Phillip Trent's Account as user msalas@us.ibm.com...

ID            bfbc369a-8d67-4900-a44c-739e28a17a6d
Address       169.61.244.19
Name          app1fip
Target        primary(b69b180e-.)
Target Type   intf
Target IP     10.10.11.13
Created       4 minutes ago
Status        available
Zone          us-south-1
```
**Associate app2fip to instance AppServ2**
```
ibmcloud is instance-network-interface-floating-ip-add $APPSERV2 $APPSERV2_NIC0 $APP2FIP
```
Result
```
Creating floatingip 617bc7f7-4d62-4f67-9991-4378594fbf59 for instance 802369d8-6070-407f-af1f-8d42c0254286 under account Phillip Trent's Account as user msalas@us.ibm.com...

ID            617bc7f7-4d62-4f67-9991-4378594fbf59
Address       169.61.244.186
Name          app2fip
Target        primary(39ccf7b5-.)
Target Type   intf
Target IP     10.10.11.9
Created       3 minutes ago
Status        available
Zone          us-south-1
```
**Associate data1fip to instance MySQL1**
```
ibmcloud is instance-network-interface-floating-ip-add $MYSQL1 $MYSQL1_NIC $DATA1FIP
```
Result
```
Creating floatingip 57136ac1-ceb1-491f-a175-b37c6bb0d8b4 for instance ee4c3eb6-ad47-4449-b8cc-39569d109de9 under account Phillip Trent's Account as user msalas@us.ibm.com...

ID            57136ac1-ceb1-491f-a175-b37c6bb0d8b4
Address       169.61.244.91
Name          data1fip
Target        primary(1fb48864-.)
Target Type   intf
Target IP     10.10.12.12
Created       3 minutes ago
Status        available
Zone          us-south-1
```
**Associate data2fip to instance MySQL2**
```
ibmcloud is instance-network-interface-floating-ip-add $MYSQL2 $MYSQL2_NIC $DATA2FIP
```
Result
```
Creating floatingip ed744792-ecea-4d70-97bf-8e07bae38936 for instance 7a4056d6-7527-43a7-9424-3e6088842eee under account Phillip Trent's Account as user msalas@us.ibm.com...

ID            ed744792-ecea-4d70-97bf-8e07bae38936
Address       169.61.244.105
Name          data2fip
Target        primary(c5c462b8-.)
Target Type   intf
Target IP     10.10.12.14
Created       2 minutes ago
Status        available
Zone          us-south-1
```

### Create a Public Gateway

Create a Public Gateway to give access to the internet and deploy images to the application and database servers from the public repositories.

Syntax: [Create a public gateway](https://cloud.ibm.com/docs/vpc?topic=vpc-infrastructure-cli-plugin-vpc-reference#public-gateway-create)

**Create Public Gateway - wp_vpc_pub_gw**
```
ibmcloud is public-gateway-create wp_vpc_pub_gw $VPC $ZONE --resource-group-id $RESOURCE_GROUP
```
Result
```
Creating public gateway wp_vpc_pub_gw in resource group 594a009f2d4b4128ad1f25b55c991de0 under account Phillip Trent's Account as user msalas@us.ibm.com...

ID               f810e374-1b36-4b5c-ab9e-dc746fb47cb2
Name             wp_vpc_pub_gw
Floating IP      169.61.244.191(f810e374-1b36-4b5c-ab9e-dc746fb47cb2)
Status           pending
Created          now
Zone             us-south-1
VPC              wp_vpc(4c69d7fe-9407-48e3-9855-dc27f595d321)
Resource Group   -
```
- Environment variable: `PUBGW=f810e374-1b36-4b5c-ab9e-dc746fb47cb2`

**Add Public Gateway to each subnet**

Syntax: [Update a subnet](https://cloud.ibm.com/docs/vpc?topic=vpc-infrastructure-cli-plugin-vpc-reference#subnet-update)

**Subnet1**
```
ibmcloud is subnet-update $SUBNET1 --public-gateway-id $PUBGW
```
Result
```
Updating Subnet 6e422da5-25d7-49aa-aaad-5a6e12c47a6a under account Phillip Trent's Account as user msalas@us.ibm.com...

ID                  6e422da5-25d7-49aa-aaad-5a6e12c47a6a
Name                subnet1
IPv*                ipv4
IPv4 CIDR           10.10.11.0/24
IPv6 CIDR           -
Address Available   245
Address Total       256
ACL                 allow-all-network-acl-4c69d7fe-9407-48e3-9855-dc27f595d321(231ed3f3-d4fc-4613-9d8c-851aeb627f46)
Gateway             wp_vpc_pub_gw(f810e374-1b36-4b5c-ab9e-dc746fb47cb2)
Created             1 hour ago
Status              available
Zone                us-south-1
VPC                 wp_vpc(4c69d7fe-9407-48e3-9855-dc27f595d321)
```
**Subnet2**
```
ibmcloud is subnet-update $SUBNET2 --public-gateway-id $PUBGW
```
Result
```
Updating Subnet 143f4dd6-fc2f-4404-b913-392fe4e11ffa under account Phillip Trent's Account as user msalas@us.ibm.com...

ID                  143f4dd6-fc2f-4404-b913-392fe4e11ffa
Name                subnet2
IPv*                ipv4
IPv4 CIDR           10.10.12.0/24
IPv6 CIDR           -
Address Available   247
Address Total       256
ACL                 allow-all-network-acl-4c69d7fe-9407-48e3-9855-dc27f595d321(231ed3f3-d4fc-4613-9d8c-851aeb627f46)
Gateway             wp_vpc_pub_gw(f810e374-1b36-4b5c-ab9e-dc746fb47cb2)
Created             1 hour ago
Status              available
Zone                us-south-1
VPC                 wp_vpc(4c69d7fe-9407-48e3-9855-dc27f595d321)
```

## Add Rules to Security Groups

In our scenario we will configure the security groups to enable the required ports and protocols.

To allow ssh, MySQL, and HTTP traffic, in each security group do the following:

Syntax: [Add a rule to a security group. The IP version defaults to IPv4](https://cloud.ibm.com/docs/vpc?topic=vpc-infrastructure-cli-plugin-vpc-reference#security-group-rule-add)

**Application Security Group**

**Add an inbound rule to allow all tcp access on port 22 for SSH access to the VSIs.**
```
ibmcloud is security-group-rule-add $APP_SG inbound tcp --remote 0.0.0.0/0 --port-min 22 --port-max 22
```
Result
```
Creating rule for security group 2d364f0a-a870-42c3-a554-000001538227 under account Phillip Trent's Account as user msalas@us.ibm.com...

ID                     b597cff2-38e8-4e6e-999d-000005189801
Direction              inbound
IPv*                   ipv4
Protocol               tcp
Min Destination Port   22
Max Destination Port   22
Remote                 0.0.0.0/0
```
**Add an inbound rule to allow all tcp access on port 80 for HTTP access to the web application.**
```
ibmcloud is security-group-rule-add $APP_SG inbound tcp --remote 0.0.0.0/0 --port-min 80 --port-max 80
```
Result
```
Creating rule for security group 2d364f0a-a870-42c3-a554-000001538227 under account Phillip Trent's Account as user msalas@us.ibm.com...

ID                     b597cff2-38e8-4e6e-999d-000005189971
Direction              inbound
IPv*                   ipv4
Protocol               tcp
Min Destination Port   80
Max Destination Port   80
Remote                 0.0.0.0/0
```
**Add an outbound rule to allow all outbound access**
```
ibmcloud is security-group-rule-add $APP_SG outbound all
```
Result
```
Creating rule for security group 2d364f0a-a870-42c3-a554-000001538227 under account Phillip Trent's Account as user msalas@us.ibm.com...

ID          b597cff2-38e8-4e6e-999d-000005190321
Direction   outbound
IPv*        ipv4
Protocol    all
Remote      -
```

**Data Security Group**

**Add an inbound rule to allow all tcp access on port 22 for SSH access to the VSIs.**
```
ibmcloud is security-group-rule-add $DATA_SG inbound tcp --remote 0.0.0.0/0 --port-min 22 --port-max 22
```
Result
```
Creating rule for security group 2d364f0a-a870-42c3-a554-000001538067 under account Phillip Trent's Account as user msalas@us.ibm.com...

ID                     b597cff2-38e8-4e6e-999d-000005190069
Direction              inbound
IPv*                   ipv4
Protocol               tcp
Min Destination Port   22
Max Destination Port   22
Remote                 0.0.0.0/0
```
**Add an inbound rule to allow all tcp access on port 3306 for MySQL (default port for MySQL).**
```
ibmcloud is security-group-rule-add $DATA_SG inbound tcp --remote 0.0.0.0/0 --port-min 3306 --port-max 3306
```
Result
```
Creating rule for security group 2d364f0a-a870-42c3-a554-000001538067 under account Phillip Trent's Account as user msalas@us.ibm.com...

ID                     b597cff2-38e8-4e6e-999d-000005190079
Direction              inbound
IPv*                   ipv4
Protocol               tcp
Min Destination Port   3306
Max Destination Port   3306
Remote                 0.0.0.0/0
```
**Add an outbound rule to allow all outbound access**
```
ibmcloud is security-group-rule-add $DATA_SG outbound all
```
Result
```
Creating rule for security group 2d364f0a-a870-42c3-a554-000001538067 under account Phillip Trent's Account as user msalas@us.ibm.com...

ID          b597cff2-38e8-4e6e-999d-000005190099
Direction   outbound
IPv*        ipv4
Protocol    all
Remote      -
```

## Next Step

At this point the VPC infrastructure components are ready for the next step which is to deploy the application software to the VSIs and test the Load Balancer. Please go to [Install and Configure Application Software](WebApp.md) for the next steps.

## Remove Floating IPs

Once the environment is up and running, you can remove the floating IPs to remove public access on the VSIs.

Syntax: [Disassociate floating IP](https://cloud.ibm.com/docs/vpc?topic=vpc-infrastructure-cli-plugin-vpc-reference#instance-network-interface-floating-ip-remove)

For example, to remove the floating IP on `AppServ1`:
```
ibmcloud is instance-network-interface-floating-ip-remove $APPSERV1 $APPSERV1_NIC $APP1FIP
```

Optionally, you can also release the Floating IPs if there is no longer a need for them.

Syntax: [ibmcloud is floating-ip-release](https://cloud.ibm.com/docs/vpc?topic=vpc-infrastructure-cli-plugin-vpc-reference#floating-ip-release)

For example, to release floating IP `app1fip`:
```
ibmcloud is floating-ip-release $APP1FIP
```
