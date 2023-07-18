# Getting Started with AWS: Creating a VPC with Public and Private Subnets

What you will learn:

- **Virtual Private Cloud (VPC):** You'll learn what a VPC is and how to set one up with custom settings. You'll practice setting specific CIDR blocks, enabling DNS hostnames, and enabling DNS resolution.

- **Subnets:** You'll understand the role of subnets within a VPC, how to create them, and how to set up their IP addressing with CIDR blocks. You'll practice creating both public and private subnets.
- **Subnets:** You'll understand the role of subnets within a VPC, how to create them, and how to set up their IP addressing with CIDR blocks. You'll practice creating both public and private subnets.

- **Internet Gateways:** You'll learn what an Internet Gateway (IGW) is, and how to create and attach one to a VPC to enable communication with the internet.

- **Route Tables:** You'll learn about routing in AWS, specifically how to create and modify a route table. You'll practice setting up a route to an Internet Gateway.

- **Security Groups:** You'll learn about AWS security groups, how they act as a virtual firewall for EC2 instances, and how to create them. You'll practice setting inbound and outbound rules to control traffic to and from instances.

- **IP Addressing:** You'll understand the importance of IP addressing in AWS, how to work with private and public IPv4 addresses, and how to auto-assign public IPv4 addresses to instances in a subnet.

This guide offers a practical introduction to these fundamental AWS networking services, laying the groundwork for more advanced topics like VPC peering, NAT gateways, and Network ACLs.

For Region, I will assume you are using "eu-west-2" throughout. You can use a different Region as long as you are consistent.

First we need to create a VPC, but why do we want this?

A Virtual Private Cloud (VPC) is a virtual network within the Amazon Web Services (AWS) cloud that enables you to create your own isolated network environment with exclusive control and security. With a VPC, you gain the ability to:

1. Define private IP address ranges: You can allocate and customize private IP address ranges for your resources within the VPC, giving you control over your network's addressing scheme.

2. Create subnets: You can divide your VPC into subnets, allowing you to organize and isolate resources based on specific requirements. Subnets help establish boundaries and control traffic flow within your network.

3. Configure routing: You have the flexibility to set up routing tables, allowing you to control how network traffic flows within your VPC and between different subnets.

4. Implement network access control: By utilizing network access control lists (ACLs) and security groups, you can define fine-grained network security policies to control inbound and outbound traffic to your resources within the VPC.

5. Establish private connectivity: A VPC allows you to establish private and secure communication between different resources within your VPC, such as EC2 instances, RDS databases, and Lambda functions, without exposing them to the public internet.

6. Integrate AWS services privately: You can connect and integrate various AWS services within your VPC, enabling private communication and seamless interaction between resources. This allows you to create interconnected architectures and build scalable applications within your own isolated network environment.

In summary, a VPC provides you with the ability to define private IP address ranges, create subnets, configure routing, implement network access control, establish private connectivity, and integrate AWS services privately. These features empower you to build secure, controlled, and interconnected network environments tailored to your specific needs within the AWS cloud.

1. **Create a VPC:**
   - Log into the AWS Management Console, navigate to the VPC Dashboard (you can type VPC into the Search input).
   - Click "Create VPC".
   - Under "VPC settings" select "VPC only"
   - Name your VPC anything you want "MyVPC" for instance, and set the IPv4 CIDR block as `10.10.0.0/16`.
   - Leave the IPv6 CIDR block and tenancy at their default settings, then click "Create".
   - After the VPC is created, select it, under "Your VPCs", and then choose "Actions -> Edit VPC settings".
   - Under "DNS settings" enable both "DNS resolution" and "DNS hostnames".
   - Click "Save"

## CIDR

> This is an in-depth explanation of CIDR notation.  It isn't required for exam purposes, but is essential if you need to create AWS networks.  If you don't then skip this and jump to step 2 below.

A CIDR (Classless Inter-Domain Routing) block is a notation used to specify the network address and the size of a network. It is represented by an IP address followed by a slash ("/") and a number.

A typical CIDR block looks like `10.10.0.0/16`. The `/16` is referred to as the network mask or subnet mask.

CIDR notation is best understood at the binary level. An IP address is composed of four decimal numbers separated by dots (e.g., n.n.n.n). These decimal numbers represent values ranging from 0 to 255. Internally, each of these decimal numbers is converted into its binary representation, which consists of 8 bits, called an octet. Thus we have 4 octets of 8 bits or 32 bits in total.

> In binary, the IP part of the CIDR number above looks like:
> 00001010.00001010.00000000.00000000

The `/16` means that the first 16 bits of the IP address is fixed and represents the network address. The remaining bits, in this case also 16, are the host bits and represent available IP addresses within our VPC.

To construct our network mask of `/16`, we just have to write out 16 bits padded out by 0's:

> 11111111.11111111.00000000.00000000

A different mask of `/24` would be 24 1's followed by 8 0's (32 total IP bits - 24 network mask bits = 8)

If we place this mask above the binary representation of our original IP address of `10.10.0.0`, we can see how they relate:

> 11111111.11111111.00000000.00000000 <- network mask
> 00001010.00001010.00000000.00000000 <- IP address

All you need to remember is that the 1's in the network mask indicate that the corresponding digits in our IP address cannot be changed, and the 0's in our network mask indicate that the corresponding digits in our IP address **can** change. This means that this CIDR range gives us two octets to play with or 16 bits, which is 2^16 or 65,536 hosts.

2. **Create an Internet Gateway (IGW) and attach it to your VPC:**
   - In the VPC Dashboard, click "Internet Gateways" in the left navigation, then "Create internet gateway".
   - Name your IGW and click "Create".
   - After the IGW is created, choose "Actions -> Attach to VPC".
   - Select the VPC you created and click "Attach internet gateway".

## Subnets

A subnet is a smaller, segmented network within a larger network, such as a Virtual Private Cloud (VPC). It allows us to further divide and organize our network infrastructure into more manageable units. Subnets help in implementing network isolation, applying security measures, and efficiently managing IP addresses.

Subdividing a VPC into subnets serves several purposes. Firstly, it enables better organization and logical separation of resources. For example, you might have different subnets for different application tiers (such as web servers, application servers, and database servers), each with its own specific network requirements.

Additionally, subnets enable granular control over network routing and security. By configuring routing tables and network access control lists (ACLs) at the subnet level, you can control the flow of traffic between subnets and apply specific security policies.

We define the size of our subnets using the same CIDR notation as we do for VPC's.  In this case we use a subnet mask of `/24` meaning the first 24 bits are fixed, leaving us 8 bits to define the available hosts within our subnet.

3. **Create subnets:**
   - In the VPC Dashboard, click "Subnets" in the left navigation, then "Create subnet".
   3.1 **Create a public subnet:**
     - Select your VPC from the dropdown.
	 - Name your subnet, for instance: `MyDemoSubnet-Pub`
	 - For Availability Zone you can leave it to "No preference"
	 - For IPv4 CIDR block, use `10.10.0.0/24`
	 - Click "Add new subnet"
   3.2 **Create a private subnet:**
     - Name: `MyDemoSubnet-Pri`
	 - AZ as above
	 - IPv4 CIDR block: `10.10.1.0/24`
	 - Click "Create subnet"

By default both subnets are auto-assigned to the default route table. Even though you previously created an IGW and attached it to the VPC there are no routes defined to this IGW so internet access isn't possible - your subnets are private.

> Exam tip: a subnet can only be associated with a single route table, but a route table can be used by many subnets!

## Route Tables

A route table is like a map that helps network traffic find its way within a Virtual Private Cloud (VPC) in Amazon Web Services (AWS). It contains a set of rules, called routes, that determine how incoming and outgoing traffic is directed.

When network traffic arrives at a VPC, the route table guides it to the appropriate destination based on the IP addresses involved. It specifies where the traffic should go, whether it's within the VPC, outside the VPC (to the internet), or to other connected networks.

By configuring the route table, you can control how traffic flows between subnets, gateways, and other network resources. This allows you to define the paths and connections within your VPC and manage the flow of data between different components.

If we didn't have a route table in a Virtual Private Cloud (VPC), network traffic within the VPC would not know where to go and would effectively be lost. A route table provides instructions on how to direct incoming and outgoing traffic based on the IP addresses involved.

Without a route table, network traffic would not be able to reach its intended destinations. It would be unable to navigate between subnets, gateways, or other connected networks within the VPC. As a result, communication between different resources and services within the VPC would be impossible.

In essence, a route table is essential for establishing the paths and connections within a VPC, allowing network traffic to flow correctly and reach its designated destinations. Without a route table, the network would lack the necessary guidance and coordination, leading to disruptions and an inability to establish connectivity within the VPC.

In summary, a route table serves as a routing guide for network traffic within a VPC, ensuring that it reaches the intended destinations by following the specified routes.

1. **Create a route table for public:**
   - In the VPC Dashboard, click "Route Tables" in the left navigation, then "Create route table".
   - Name your route table, select your VPC and click "Create".
   - After the route table is created, click "Edit routes".
   - Click "Add route" with a destination `0.0.0.0/0` and target, choose "Internet Gateway" then select the IGW you created.
   - Click "Save changes"
   - Click "Actions" > "Edit subnet associations"
   - Select your **public** subnet, click "Save associations"

1. **Create a route table for private:**
   - In the VPC Dashboard, click "Route Tables" in the left navigation, then "Create route table".
   - Name your route table, select your VPC and click "Create".
   - Click "Actions" > "Edit subnet associations"
   - Select your **private** subnet, click "Save associations"

## Enabling Internet Access to a Subnet

1. Select the subnet you want to make public, `MyDemoSubnet-Pub`
2. Click Actions > Edit subnet settings
3. Under "Auto-assign IP settings" select "Enable auto-assign public IPv4 address"

## Viewing What You Built

1. Navigate back to the VPC dashboard (remember you can search for VPC in the header search input)
2. Select "Your VPCs"
3. Click the checkbox next to the VPC you created `MyDemoVPC`
4. Select the "Resource map" tab
5. Mouseover the Subnets to see the routes and associations.

## Logging Network Traffic

**What are VPC Flow Logs?**

It's a feature provided by Amazon VPC that captures information about IP traffic going to and from network interfaces in your VPC. It provides a window into your network traffic, helping you understand traffic patterns, troubleshoot security and network connectivity issues, and ensure compliance with network policies.

Flow logs data can be published to Amazon CloudWatch Logs or Amazon S3, where you can view and retrieve the log data. It's important to note that VPC Flow Logs do not log real-time data, there's a lag of a few minutes.

**How VPC Flow Logs Complement CloudWatch Metrics**

While Amazon CloudWatch provides you with data and actionable insights to monitor your AWS resources, by default, it doesn't offer in-depth insight into your network traffic at the packet level, such as source and destination IP addresses, packet and byte counts, and traffic actions (ACCEPT/REJECT). This is where VPC Flow Logs come in.

VPC Flow Logs provide detailed network flow data about your VPC that CloudWatch doesn't collect by default. This gives you a much more granular view into your network traffic that complements the aggregated and higher-level metrics and alarms provided by CloudWatch. 

By combining VPC Flow Logs with CloudWatch, you can create a comprehensive monitoring solution that gives you visibility into not just the health and performance of your AWS resources, but also the network traffic flowing in and out of your VPC, offering a broader perspective for troubleshooting, optimizing performance, and enhancing security.

1. **Create CloudWatch log group:**
   - First you need to CloudWatch > log groups.
   - Click "Create log group"
   - Enter a name, such as `/aws/vpc/my-demo-VPC-logs`
   - Change retention settings to 1 month.
   - Click "Create"

2. **Create a policy to publish logs to CloudWatch:**
  - Navigate to the IAM console. Click "policies".
  - Click "Create policy"
  - Click the "JSON" button and replace the contents of the "Policy editor" with the content below:
  
```json
{
 "Version": "2012-10-17",
 "Statement": [
	 {
		 "Action": [
			 "logs:CreateLogGroup",
			 "logs:CreateLogStream",
			 "logs:PutLogEvents",
			 "logs:DescribeLogGroups",
			 "logs:DescribeLogStreams"
		 ],
		 "Effect": "Allow",
		 "Resource": "*"
	 }
 ]
}
```
  - Click "Next"
  - Name: "DemoVPCFlowLogAccessCloudWatchPolicy"
  - Click "Create policy"

3. **Create a role:**
  - Select "Roles"
  - Click "Create role"
  - For "Trusted entity type" select "AWS service"
  - For "Common use cases" select "EC2"
  - Click "Next"
  - Select the policy you created above.
  - Click "Next"
  - Enter role name: "DemoVPCFlowLogRole"
  - Click "Create role"
  -  Select the "trust relationships" tab for this Role, click "Edit trust policy" and enter the JSON below:
  
```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Principal": {
                "Service": "ec2.amazonaws.com"
            },
            "Action": "sts:AssumeRole"
        },
        {
            "Effect": "Allow",
            "Principal": {
                "Service": "vpc-flow-logs.amazonaws.com"
            },
            "Action": "sts:AssumeRole"
        }
    ]
}
```
  - Click "Update policy"

2. **Enable Flow Logs in VPC:**
   - Navigate to the VPC dashboard, select the VPC you created.
   - Click the "Flow logs" tab
   - Click "Create flow log"
   - Provide an optional name if you want.
   - Leave the defaults:
     - `Filter` (the type of traffic to log) leave as `All`
     - `Destination`, leave the default `Send to CloudWatch Logs`.
   - For `Destination log group`, select our log group, `/aws/vpc/my-demo-VPC-logs`.
   - For `IAM Role`, select the role you created, "DemoVPCFlowLogRole".
   - Click "Create flow log"

## Verifying Network Traffic

It can take a few minutes for VPC flow log data to be collated. Take a coffee break and come back in 10 minutes or so.

1. Navigate to the CloudWatch console.
2. Select "Log groups"
3. Select your log group "/aws/vpc/my-demo-VPC-logs"
4. Under "Log streams" you should see something named similar to: "eni-05bf222768ed90fc7-all"

> The exact name will depend on the Elastic Network Interface (ENI) that the flow log is associated with.

5. You should see data indicating that your VPC is indeed handling network traffic.
