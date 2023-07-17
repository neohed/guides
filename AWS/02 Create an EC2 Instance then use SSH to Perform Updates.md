# Create an EC2 Instance then use SSH to Perform Updates

What you will learn:

1. **Accessing the AWS Management Console and EC2 Dashboard:** You'll learn how to navigate AWS's web-based interface to manage your resources.

2. **Creating an EC2 Instance:** This is a fundamental task in AWS. You'll learn how to choose an Amazon Machine Image (AMI), select an instance type, and configure network settings.

3. **Managing Key Pairs:** AWS uses public-key cryptography to secure the login information for your instances. You'll learn how to create, download, and securely store a .pem file for SSH connections to your instances.

4. **Understanding Instance Metadata and User Data:** Instance metadata is data about your instance, while user data is optional data you can provide when launching your instance. You'll learn how to enable the metadata service and restrict it to version 2 only (token required).

5. **Connecting to an EC2 Instance using SSH:** You'll learn how to use your key pair to establish a secure connection to your instance.

6. **Working with EC2 Instance Metadata:** You'll understand how to use the instance metadata service to retrieve information about your instance.

7. **IAM Roles and Policies:** You'll learn how AWS's identity and access management system works and how to create and assign a role with specific permissions to an EC2 instance. 

8. **Running AWS CLI commands from an EC2 Instance:** You'll understand how to run AWS CLI commands from within an instance, using the instance's IAM role for authorization.

9. **Creating and Associating Security Groups:** Security groups act as virtual firewalls for your instances. You'll learn how to create a new security group and associate it with your instance.

10. **Updating an EC2 Instance:** You'll learn how to update the software packages on your EC2 instance. This is a crucial task for maintaining the security and performance of your instances.

## Prerequisites

You have created the networking resources from guide: `Getting Started with AWS: Creating a VPC with Public and Private Subnets.md`

## Web traffic to EC2

We're going to create a web server for our EC2 instance. To do this easily we will run a script in user data that will install packages using `yum`. This requires web access and we will need web access to view the website one EC2 is running.

1. **Create security group for web access:**
   - In the EC2 console, select "Security Groups" from the menu under "Network & Security"
   - Click "Create security group"
   - Provide a name.
   - Under "VPC" make sure the VPC you created is selected.
   - Under "Inbound rules" click "Add rule"
   - Add rules:
     * Type: HTTP, Source: Anywhere-IPv4
	 * Type: HTTPS, Source: Anywhere-IPv4
   - Click "Create security group"

2. **Create security group for SSH:**
   - Click "Create security group"
   - Provide a name.
   - Under "VPC" make sure the VPC you created is selected.
   - Under "Inbound rules" click "Add rule"
   - Add rule: Type: SSH, Source: `My IP`
   - Click "Create security group"

> NB., Be aware that, while locking this rule down to your IP is more secure, it will cause problems if your IP address changes, which can happen as home ISP providers use dynamic IP addresses. If you get a `port 22: Connection timed out` error when trying to SSH this is likely the reason. Simply go into this rule and update the IP address.

## Creating and Connecting to an EC2 Instance

1. **Access the EC2 Dashboard:**
   - Log into the AWS Management Console and navigate to the EC2 Dashboard.
   - Click "Instances" in the left navigation, then "Launch Instance".
   - Choose an Amazon Machine Image (AMI). For a general-purpose instance, you might select the latest Amazon Linux 2023 AMI. This should be free tier eligible.
   - Choose an instance type. For general purposes, 't2.micro' is a suitable starting point.
 
2. **Set Up Key Pair**
   - Click "Create a new key pair".
   - Leave the default type.
   - Name your key pair and download the `.pem` file. Securely store this file, as you can't download it again and it's required for SSH access to your instance.

## Configure Network Settings

1. Under "Network settings" click "Edit".
2. Select the VPC you created.
3. For "Subnet", select the public subnet you created.
4. Ensure "Auto-assign Public IP" is set to "Enable".
5. Under "Firewall (security group)" Click "Select existing security group" and select both of the security groups you created for SSH and HTTP(S).
6. Leave the other settings at their defaults and scroll down to the "Advanced Details" section.
7. Select "Select an existing security group", then choose the SSH security group you created.

### Step 5: Adjust Advanced Details

1. Open the "Advanced details" section.
2. For the "Metadata accessible" dropdown, set it to "Enabled".
3. Set "Metadata version" to "V2 only (token required)".
4. Paste the script below into the "User data" section.

```sh
#!/bin/bash
yum update -y
yum install -y httpd
systemctl start httpd
systemctl enable httpd
echo "<html><body><h1>Welcome to my Web Server!</h1></body></html>" > /var/www/html/index.html
```

5. Click "Launch instance"

### Step 6: Launch Instance

Your instance is now launching. It may take a few minutes for it to be ready. In the meantime, you can view its status on the Instances screen. Once the "Instance State" is "running" and the "Status Checks" column reads "2/2 checks passed", your instance is ready to use.

## Web traffic to EC2

1. In your EC2 console, select the EC2 instance you have created.
2. Under "Public IPv4 address" click the "open address" link to open a new browser tab with your EC2 instance's public IP address.
  - After a delay, you should see an error page: "This site can’t be reached" or similar.
  - This is because you are trying to access the server over HTTPS and, even though we allowed HTTPS traffic in our  security group, we did not provide our instance with an SSL certificate.
  - Edit the URL in the browser address bar to HTTP://{"Public IPv4 address"} (you can copy this IP address from your EC2 console)
  - You should see a web page with the message "Welcome to my Web Server!".

## SSL

To enable HTTPS traffic you will need to create and install an SSL certificate.

## Connecting to the Instance via SSH

### Step 1: Set the Correct Permissions

For the .pem file you downloaded, set the correct permissions:

```sh
chmod 400 /path/to/your/key-file.pem
```

### Step 2: SSH into Your EC2 Instance

To SSH into your EC2 instance, use the following command:

```sh
ssh -i "/path/to/your/key-file.pem" ec2-user@{your-ec2-public-dns-name}
```

Replace "/path/to/your/key-file.pem" with the path to your .pem file and "{your-ec2-public-dns-name}" with the Public DNS (IPv4) of your EC2 instance.

> Note: You can exit from your SSH connection by typing `exit`. Keep it open for now, as we'll be using it in the following steps.

## Verifying Your Setup

### Step 1: Test Instance Metadata Access

To verify your setup, you can attempt to access the instance metadata. The following command should return a `401 Unauthorized` error:

```sh
curl http://169.254.169.254/latest/meta-data/ami-id
```

The `401 Unauthorized` error confirms that we've configured the instance to use Instance Metadata Service Version 2 (IMDSv2).

### Step 2: Retrieve Session Token

To use IMDSv2, first retrieve a session token:

```sh
TOKEN=`curl -X PUT "http://169.254.169.254/latest/api/token" -H "X-aws-ec2-metadata-token-ttl-seconds: 21600"`
```

### Step 3: Access Instance Metadata

Use the session token to access the metadata:

```sh
curl -H "X-aws-ec2-metadata-token: $TOKEN" -v http://169.254.169.254/latest/meta-data/
```

This command lists all available metadata categories. 

To retrieve a specific attribute, such as the instance's private IP, use:

```sh
curl -H "X-aws-ec2-metadata-token: $TOKEN" http://169.254.169.254/latest/meta-data/instance-id && echo ""
```

> Note: appending ` && echo ""` to the `curl` command is for formatting in the console, otherwise the CLI prompt appears on the same line as the attribute.
> If you're using the `curl` command to store the attribute in a variable then omit the `echo` suffix.

## Accessing the AWS API from Your Instance

### Step 1: Test AWS API Access

Run the following command:

```sh
aws ec2 describe-instances --output table
```

You should encounter an error: `Unable to locate credentials. You can configure credentials by running "aws configure"`.

To run this command, your instance needs permissions from IAM.

### Step 2: Understanding IAM Policies

In AWS, every operation ultimately translates to an API call. IAM policies dictate who (the 'principal') can make which API calls (the 'actions') and on which resources.

The 'principal' refers to the AWS entity (a user, group, service, or role) permitted or denied access to a resource. It can be thought of as the 'who' in the policy.

An IAM policy is analogous to a rulebook for a sports game. For instance, in football, not everyone can touch the football with their hands. Similarly, in AWS, not all principals can perform all actions on all resources. The IAM policy is the rulebook clearly stating who can do what.

Imagine an IAM policy attached to a user (the principal) that allows the user to start or stop EC2 instances (the actions) but only in a specific region and only for a specific set of instances (the resources).

An IAM policy not only lists allowed API calls but also those explicitly denied. If an action isn't explicitly allowed, it is implicitly denied. This means the user can't perform that action by default. It's an allow-list of things you can do, and everything else is forbidden unless stated otherwise.

## Adding a Role to Your Instance

### Step 1: Create a JSON Policy Document

Start by creating a JSON policy document that grants permissions to run the `describe-instances` command. 

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": "ec2:DescribeInstances",
            "Resource": "*"
        }
    ]
}
```

### Step 2: Create the Policy

1. Open the IAM console at https://console.aws.amazon.com/iam/.
2. Click "Policies" in the navigation pane.
3. Click "Create policy".
4. Switch to the JSON tab.
5. Copy the JSON document above into the policy document box.
6. Click "Review policy".
7. Name the policy (e.g., "DescribeInstancesPolicy") and click "Create policy".

### Step 3: Create the Role

1. Navigate to "Roles" in the IAM console.
2. Click "Create role".
3. Select "AWS service" for the trusted entity type, choose "EC2", and click "Next: Permissions".
4. Search for the name of the policy you just created ("DescribeInstancesPolicy").
5. Check the box next to the policy you just created and click "Next: Tags".
6. Optionally add tags to the role and click "Next: Review".
7. Name your role (e.g., "DescribeInstancesRole") and click "Create role".

### Step 4: Attach the Role to Your EC2 Instance

1. Open the Amazon EC2 console at https://console.aws.amazon.com/ec2/.
2. Choose "Instances" in the navigation pane.
3. Select the instance to which you want to attach the role.
4. Click "Actions", then "Security", and finally "Modify IAM role".
5. Select your newly created IAM role in the "IAM role" list, and click "Save".

Now, you should be able to use the `describe-instances` command from the AWS CLI on your EC2 instance without manually configuring credentials. You should be able to run `aws ec2 describe-instances` within your SSH session.

## Updating Your Amazon EC2 instance

### Step 1: Create a new Security Group with HTTP/HTTPS access

1. Open the Amazon EC2 console in your web browser at https://console.aws.amazon.com/ec2/.
2. Click on "Security Groups" under "Network & Security" in the navigation pane.
3. Click "Create security group".
4. Enter a name and description for the new security group.
5. Select the ID of your VPC from the VPC dropdown list.
6. Click "Add rule" under "Inbound rules". Add two rules for HTTP and HTTPS access:
   - Type: HTTP, Protocol: TCP, Port Range: 80, Destination: Anywhere-IPv4 (This sets the IP to: 0.0.0.0/0)
   - Type: HTTPS, Protocol: TCP, Port Range: 443, Destination: Anywhere-IPv4 (This sets the IP to: 0.0.0.0/0)
7. Click "Create security group".

### Step 2: Associate the new Security Group with your EC2 Instance

1. Click on "Instances" under "Instances" in the navigation pane.
2. Select the instance you want to update.
3. Navigate to "Security -> Change Security Groups" in the "Actions" dropdown menu.
4. In the "Change security groups" dialog box, select the ID of the new security group you created, then click "Add security group".
5

. Click "Save"

### Step 3: Update Your EC2 Instance

1. Connect to your EC2 instance via SSH.

```bash
ssh -i /path/my-key-pair.pem ec2-user@ec2-xxx-xxx-xxx-xxx.compute-1.amazonaws.com
```

Replace `/path/my-key-pair.pem` with the path to your private key file, and `ec2-xxx-xxx-xxx-xxx.compute-1.amazonaws.com` with the public DNS of your instance.

2. Perform an update:

```bash
sudo yum update -y
```

Now, your EC2 instance is updated with the latest packages.
