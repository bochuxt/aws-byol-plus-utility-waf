# Utility + BYOL Deployment of BIG-IP in AWS

This directory contains CloudFormation templates which provide an example of how to combine BYOL and Utility instances of BIG-IP in AWS. BIG-IP is used as a load balancer and web application firewall.  Using this code requires acess to the 12.0 Hourly image which should be available shortly in the AWS Marketplace. Until then, this code is provided to show the use of CloudInit and autoscale features of BIG-IP. 

## Description

### CloudFormation Templates

There are four CloudFormation templates in this directory:

* common.json - This template deploys common EC2/VPC resources which will be used by the other templates.  In particular, this template creates a VPC, subnets, a routing table, common security groups, and an ElasticLoadBalancer group to which all BIG-IPs will be added. 
* application.json - This template deploys all components to support the application, with the exception of those related to BIG-IP.  An autoscaling group for the application is created, but we leave the creation of CloudWatch alarms and scaling policies as an exercise for the future. 
* autoscale-bigip.json - This template deploys an autoscaling group for utility instances BIG-IP. Example scaling policies and CloudWatch alarms are associated with the BIG-IP autoscaling group.
* byol-bigip.json - Deploys a single BYOL BIG-IP instance.

With the exception of the byol-bigip.json template, each of the other templates should only be deployed once for a given application deployment. 

### BIG-IP deployment and configuration

* All BIG-IPs are deployed with a single interface attached to a public subnet in one availability zone.
* Advanced traffic management and web application firewall functionality are provided through use of BIG-IP Local Traffic Manager (LTM) and Application Security Manager (ASM) modules.  We use the BEST image available in the AWS marketplace to license these modules.
* All BIG-IP configuration is performed at device bootup using CloudInit.  This can be seen in both the BIG-IP CFTs.  Note the slight differences in the CloudInit scripts for the two different deployment models (utility and BYOL).  In particular, the CFT BYOL template includes a licensing step.  In general, CloudInit is used to :
  * set BIG-IP hostname, NTP, and DNS settings
  * add aws-access-key and aws-secret-key to BIG-IP, allowing BIG-IP to make authenticated calls to AWS HTTPS endpoints. 
  * move management GUI on a high port number (443 -> 8443) 
  * create a virtual server with an attached ASM policy
  * deploy integration with EC2 CloudWatch for scaling of BIG-IP tier.
  * deploy integration with EC2 Autoscale service for pool member management. 

## Usage

### Prerequisites

1) Access to 12.0 'BEST' BYOL and Hourly images in the Amazon region within which you are working. <br>
- The 12.0 Hourly image should be available shortly, please contact F5 sales personal for details on its availability<br>
- Make sure that you have accepted the EULA for both images in the AWS marketplace.<br>
2) Set of AWS Access Keys for use by BIG-IP, as described here:<br>
- https://support.f5.com/kb/en-us/products/big-ip_ltm/manuals/product/bigip-ve-setup-amazon-ec2-12-0-0/4.html#unique_1903231220<br>
3) OPTIONAL - Required if you use the deploy_stacks.py script per instructions below:<br>
- You will need an AWS Access Key and Secret Access key configured in ~/.aws/credentials<br>
- Install boto3<br>
```
pip install boto3
```

### Two options 
To use this example code, you may either launch the CloudFormation templates directly or use the deploy_stacks.py script. These are mMthod 1 and Method 2 below

### Method 1 (recommend for first time) - Manually deploy the CloudFormation Templates

1) Launch the following CFTs in the following order:
   1) common.json
   2) application.json
   3) autoscale-bigip.json
   4) byol-bigip.json

This ordering is necessary because outputs values from previous templates are used as parameters to provision later templates.  Note that variables names are the same for all matching input parameters and outputs for all templates.  For example, the outputs from the common.json template include Vpc, Subnet, AvailabilityZone, and BigipSecurityGroup.  The values of these outputs should be used for the parameters with the same variable names in the other templates.

2) After you have deployed all the CloudFormation templates, add the BYOL BIG-IP to the BigipElasticLoadBalancer ELB Group. 

### Method 2 - deploy_stacks.py

This script will launch each of the CloudFormation templates in the correct order.

To use this script:

1) Find config.ini.example in this directory, copy this script to config.ini.  In this config.ini file, edit the variables for your scenario. The last flag in the script 'deploy_jmeter_host' should be given a value of 'true' if you wish to test scale out using JMeter as documented below. 

2) Then run the script:
```
python ./deploy_stacks.py
```

### Triggering scale out

For convenience, you can use the JMeter script included with these examples to trigger a scale out event.

To trigger a scale out event:

1) Find and replace all instances of '<** elb dns name**>' in the simple_jmeter_load.xml script with the DNS name associated with your BIG-IP ELB group.

i.e. <** elb dns name**> -> BigipElasticLoadBalancer-1263232202.us-east-1.elb.amazonaws.com

2) SSH to the ubuntu instance:
```
ssh -i <path to key pair you provided in config.ini> ubuntu@<ip address of the ubuntu instance> 
```

3) Copy and paste the simple_jmeter_load.xml file to the ubuntu host (possibly using vi/vim).

4) Run the script from the Ubuntu host:
```
nohup jmeter -n -t simple_jmeter_load.xml &
```

5) A CloudWatch alarm will be triggered, and EC2 Autoscale will launch another BIG-IP instance. 


## Building on these examples
The following elements might be improved in these examples to create a more robust solution:
* Adding additional subnets for various resources in the application stack in order to better leverage security constructs in EC2 Virtual Private Clouds such as security groups, network ACLs, and routing.
* Updating the deployment to span multiple availability zones.
* Combining the autoscale-bigip.json and byol-bigip.json CFTs into a single template. We have split them into two to make the relationship between various EC2, VPC, and CloudWatch resources easier to understand.

For further docmentation of the solution see the pdf in /docs 
