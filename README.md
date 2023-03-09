*SECTION I: Warm up*

At the beginning of the lecture, pleas deploy the CloudFormation stacks as guided below:

1) Download the template files from here: https://drive.corp.amazon.com/personal/netd/Demo1
2) Make sure that you have SSH key Pair configured in Singapore and Sydney regions (AWS Console - EC2 - Key Pairs); if you do not, please provision one before deploying the CloudFormation stacks;
3) Create stack in Singapore region from template file Stack1SIN.yaml ; use “demo1” as the stack name
4) Create stack in Sydney region from template file Stack2SYD.yaml ; use your alias (use small letters only) as the stack name


*SECTION II: Exercise description*

Case scenario: 
You were approached by your manager to reconfigure network access to S3 bucket in Sydney region from workloads in Singapore region. Currently, the traffic flows through Internet like below:

[Image: BeforeNetworkUpgrade]Fig 1. Before Network Upgrade



However due to recently imposed changes by regulator, the data from the Client Instance to S3 bucket must use AWS Private Backbone Network to prevent from potential interception by malicious third-parties. The new traffic flow looks like below:
[Image: Image.jpg]Fig 2. After network upgrade



You will utilise VPC Peering feature along with S3 Gateway endpoint to make it happen. S3 Gateway endpoint seems to be a perfect choice as it does not incur provisioning and data transfer charges. However,  S3 Gateway endpoint accepts only requests from resources deployed in the same region as the S3 bucket; hence the Client Instance in Singapore will not be able to connect to S3 bucket in Sydney region without further network modifications. To overcome this limitation, you will use Proxy Instance that will accept the requests from Client Instance coming over VPC peering connection and will forward them privately to S3 service via S3 Gateway endpoint. 

The CloudFormation provisions the majority of the infrastructure, such as VPCs, Client and Proxy Instances (AmazonLinux 2), and VPC Interface endpoints required to initiate the console session with EC2 instance via Session Manager (SSM). All Security Groups and Network Access Control lists allow all traffic in ingress and egress directions. 

*SECTION III: Exercise guide*

1) Testing environment before network upgrade:

a) The S3 bucket has been created for you through CloudFormation; however, it is empty at the moment. Please upload some file into the bucket (the file should not be larger than 5 MB). The bucket name is “awsdesignbckt-<youralias>” and it is created in Sydney region (ap-southeast-2). 

b) Connect via SSM into the Client Instance and use AWS CLI command (aws s3) to download this file from the bucket to home folder. Did it work?

c) Delete the default route 0.0.0.0/0 from the route table associated with the subnet where the Client Instance is deployed. That is to stop Internet connectivity (in result you will not be able to SSH to the instance; use SSM instead). Retry to download the same file. Did it work?  

2) Configuring new network:

a) Connect via SSM into the Proxy Instance and install Squid web proxy software; start and enable the squid service with the below commands:

sudo yum install squid -y
sudo systemctl enable squid.service
sudo systemctl start squid.service

b) Create S3 VPC Gateway endpoint in the VPC SYDVPC and make sure all route tables are updated with the S3 Gateway Endpoint route; please leave VPC endpoint policy set to “Full access”

c) Remove the default route 0.0.0.0/0 from the route table associated with the subnet where the Proxy Instance is deployed (you will no longer be able to use the SSH method to access the Proxy instance; use SSM instead). It is to ensure we no longer use Internet to connect to the S3 service, but rather traffic flows through gateway endpoint. Try to download the file through AWS CLI (you may need to use additional flag --region in the command). The transfer should be successful.

e) Create VPC peering connection between the VPCs: SYDVPC and SINVPC and update route tables so that the Client Instance and Proxy instance can communicate; test the connectivity between the 2 instances using ping tool. The ping must be successful before you proceed with the next step. 

f) SSM into the Client Instance and try to download the file from the S3. It should fail. Why? 

f) Update the proxy settings on the Client Instance so that web traffic to S3 bucket is forwarded to Proxy Instance (ensure you update the HTTP and HTTPS proxy). Try to download the file once again - it should be successful. 

Optional:

To confirm the traffic flows through the Proxy instance, run tcpdump command on the Proxy Instance to capture the traffic from the Client Instance:

sudo tcpdump -nni any host <IPAddressofClientInstance>



Delete resources:

1) Empty the bucket
2) Delete S3 VPC Gateway endpoint
3) Delete VPC peering connection and remove blackhole routes from the route tables
4) Delete CloudFormation Stacks
