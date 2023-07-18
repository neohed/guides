# 03 - Setting Up and Connecting to a PostgreSQL Database on Amazon RDS via a Bastion Host

In this guide, you'll learn about:

1. **Amazon RDS:** Understand what Amazon Relational Database Service (RDS) is, and how to use it to deploy a PostgreSQL database.

2. **Creating a PostgreSQL Database:** Learn how to create a PostgreSQL database instance in RDS, with a focus on details such as instance identifier, username, password, and connectivity settings.

3. **Understanding Virtual Private Cloud (VPC):** Deepen your understanding of VPCs in AWS, particularly in terms of their role in setting up a database.

4. **Database Connectivity:** Understand how to configure the database to be private and accessible only via a specific EC2 instance (acting as a bastion host).

5. **Security Groups in AWS:** Learn about AWS Security Groups and how they can control access to your DB instance.

6. **PostgreSQL Client:** Learn how to install and use the PostgreSQL client on your EC2 instance, allowing you to interact with your database.

7. **Connecting to RDS from EC2:** Understand how to use an EC2 instance as a bastion host to connect to your RDS instance.

8. **Performing Basic SQL Operations:** Once connected, learn how to perform basic SQL operations like listing databases and checking the server's current date and time.

9. **Managing Database Access:** Learn how security group rules can enable continuous access to the database even when the IP address of your EC2 instance changes.

10. **Writing an SQL Script for RDS** - Writing an SQL script, secure file transfer over SSH, executing the SQL script on the PostgreSQL database and verifying the table creation.

## Create the DB

1. **Access the RDS Dashboard:**
   - Log into the AWS Management Console and navigate to the RDS Dashboard.

2. **Create a new PostgreSQL database:**
   - Click "Databases" in the left navigation, then "Create database".
   - Select "Standard Create" for more detailed settings.
   - For "Engine options", select "PostgreSQL".
   - For "Template" choose "Free tier".

3. **Settings:**
   - Choose the "DB instance identifier", "Master username", and password.
   - Remember the master username and password as you'll need them to connect to the database.
   - Leave the other settings at their defaults unless you have specific needs. 

4. **Connectivity:**
   - Select "Don’t connect to an EC2 compute resource". We will manually configure this.
   - For "Virtual Private Cloud (VPC)", choose the VPC you created earlier. For "Subnet group", leave it as default.
   - For "Publicly accessible", choose "No" because you'll access the DB through a bastion host.
   - In the "VPC security group" section, choose "Create new" and then provide a name for the new security group.
   - Leave the other settings at their defaults unless you have specific needs.
     - "Database authentication" should be left at default of: "Password authentication"

5. **Monitoring:**
   - Disable "Turn on Performance Insights"
   - Click "Create database".

> If you see an error like this:
> ERROR: "DB Subnet Group doesn't meet availability zone coverage requirement. Please add subnets to cover at least 2 availability zones."
> It's because the subnets in your VPC are all in the same AZ (Availabillity Zone).
> FIX: create another subnet for your VPC in a different AZ, if you followed this guide, you can use a CIDR range of: `10.10.2.0/24`
> Exam tip: remember that Amazon RDS requires at least two subnets in two different Availability Zones to support Multi-AZ DB instance deployments!

6. **Create a Bastion Host Security Group:**
   - Navigate to EC2 console.
   - Click "Create security group"
   - Name: "EC2-bastion-host-RDS-SG"
   - Description: "A minimal SG for EC2 instances that act as bastion hosts for RDS"
   - VPC: Change this to the VPC you created.
   - Inbound rule: Type: SSH, Source: MyIP
   - Click "Create security group"

7. **Modify the RDS Security Group:**
   - Once your RDS instance is created, select the RDS instance in the RDS Dashboard.
   - Scroll down to "Connectivity & security". Find the "VPC security groups" link and open that in a new tab.
   - In the inbound rules, select "Edit inbound rules".
   - Add a new rule with type "PostgreSQL". For "Source", select "Custom" and then select the security group created above: "EC2-bastion-host-RDS-SG"
   - Save the rule.

8. **Add EC2 Bastion Host to Security Group**
   - In EC2 console, select your EC2 instance, Click "Actions" > "Security" > "Change security groups"
   - Under "Associated security groups" select the security group created above, click "Add security group"
   - Click "Save"
   
Now, you have an RDS PostgreSQL instance that your EC2 instance can access. Anytime your EC2 instance's IP changes, the connection won't be affected because the security group rule references the EC2's security group rather than its IP. You can SSH into your EC2 instance and then connect to the RDS instance using the `psql` command.

## Storing Database Credentials in AWS Secrets Manager

For security reasons, it's advisable not to hard code the database credentials in your Lambda function. Instead, these credentials can be stored in AWS Secrets Manager, allowing your Lambda function to retrieve them when necessary.

> Note: Secrets Manager costs $0.40 per secret per month.

1. **Access AWS Secrets Manager**
   - In the AWS console, navigate to the Secrets Manager service.

2. **Store a New Secret**
   - Click on "Store a new secret". In the "Select secret type" section, choose "Credentials for RDS database".

3. **Credentials**
   - Input the master username and password in the respective fields.

4. **Encryption key**
   - Leave the default

5. **Database**
   - Select your DB instance.
   - Click "Next".

6. **Secret name**
   - In the "Secret name and description" section:
      - For "Secret name", enter `dev/my-lambda/postgres`.
      - Optionally, add a description and tags for better resource management.
   - Click "Next". On the following screen, you don't need to change anything. Simply click "Next" again.

7. **Review and Store the Secret**
   - Review your settings and the provided sample code snippets, which can come in handy when you want to retrieve the secret. Once everything looks good, click "Store".

You've now securely stored your database credentials in AWS Secrets Manager, ready for your Lambda function to use.

## Installing a DB Client and Creating a Database Table

1. **Connect to Your EC2 Instance via SSH**
   - Connect to your EC2 instance using SSH.

2. **Install the PostgreSQL Client**
   - Execute the following command to install the PostgreSQL client:

```bash
sudo yum install -y postgresql
```

> Note: If you see an error like: `No match for argument: postgresql Error: Unable to find a match: postgresql`, it means that the package is not available. You can check if this repository is enabled with the command `sudo yum repolist`. If it's not listed, you can install it with `sudo dnf install postgresql15`.

3. **Connect to PostgreSQL Database**
   - Run the CLI command below, replacing `<DB-ENDPOINT>` and `<USER-NAME>` with your actual RDS endpoint and database username, respectively. When prompted, enter the password you input when you created the RDS instance.

```bash
psql --host=<DB-ENDPOINT> --dbname=postgres --username=<USER-NAME> --password
```

> Tip: You can disconnect from your RDS DB by running the command `\q`.

4. **Verify Your DB Instance is Working**
   - After you've connected to your PostgreSQL database with the `psql` command, list all databases in your PostgreSQL RDS instance with `\l`. You can also check the current date and time according to the database server with `SELECT current_date, current_time;`.

```bash
\l
```

```sql
SELECT current_date, current_time;
```

> Reminder: All commands in `psql` have to end with a semicolon (;).

5. **Create a DB Table**

Create a file locally on your computer called: `create-table.sql`, with the content below:

```sql
CREATE EXTENSION IF NOT EXISTS "uuid-ossp";
CREATE TABLE demo_table (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    message TEXT,
    created_date TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
```

6. **Upload SQL Script to EC2 Instance**
   - Run the command below, replacing `{ec2-ip-address}` with the Public IPv4 address of your EC2 instance, and `{/path/to/your/key.pem}` with the path to your key file.

```bash
scp -i {/path/to/your/key.pem} create-table.sql ec2-user@{ec2-ip-address}:/home/ec2-user/
```

SSH into your EC2 instance:

```sh
ssh -i "/path/to/your/key-file.pem" ec2-user@your-ec2-public-dns-name
```

> Tip: If you get a `port 22: Connection timed out` error, it's likely because the IP address you're connecting from has changed since you created the security group rule for your EC2 instance. Simply update the IP address associated with that rule.

7. **Connect to DB**
   - Connect to your database with:

```bash
psql --host=<DB-ENDPOINT> --dbname=postgres --username=<USER-NAME> --password
```

8. **Run SQL Script**
   - Execute the script you uploaded with:

```bash
\i create-table.sql
```

9. **Verify Table Creation**

Verify that your table has been created by listing tables and selecting data from your table:

```bash
\dt
```

```sql
SELECT * FROM demo_table;
```

Finally, disconnect from your DB:

```bash
\q
```
