# 04 Create a Lambda Function to Perform CRUD Operations against RDS

## Creating a VPC Endpoint for Secrets Manager

### Introduction

Some AWS services, such as Secrets Manager, are primarily accessed over the internet. Therefore, even if you're using them from within your private network (VPC), the traffic to these services travels over the internet. 

A VPC Endpoint allows you to privately connect your VPC to supported AWS services without needing to access the internet, akin to a secure, private tunnel from your VPC directly to the service. For Secrets Manager, creating a VPC Endpoint equates to establishing a private and direct connection from your VPC to Secrets Manager, bypassing the need to use the public internet. Once set up, any traffic to Secrets Manager from your VPC will flow through this private connection.

1. **Create Endpoint:**
   - Open the Amazon VPC console at `https://console.aws.amazon.com/vpc/`.
   - In the navigation pane, select "Endpoints", then "Create Endpoint".
   - For "Service category", choose "AWS services".
   - For "Services", search for "secretsmanager" then select the one that appears. It should be "com.amazonaws.eu-west-2.secretsmanager".
   - For "VPC", select your VPC.
   - For "Subnets", select your private subnet. You can check the Availability Zones then check the dropdowns to figure out what AZ this subnet is in.
   - Security groups: SecretsManager only allows HTTPS access. Select the SG you created to provide web access to your EC2 instance or create a new SG that allows inbound HTTPS traffic only.
   - Leave the policy as Full access for now.
   - Click "Create Endpoint"

Your VPC Endpoint for Secrets Manager is now set up. This ensures a private connection between your VPC and Secrets Manager, enhancing the security of your AWS setup.

## Create an IAM Role for Your Lambda Function

Before creating your Lambda function, you need an IAM role that the Lambda function will use. This role should have:

  - An inline policy that allows `secretsmanager:GetSecretValue` action (as your Lambda function will need to retrieve the database credentials stored in Secrets Manager)

In the AWS console, go to IAM, create a new role for Lambda

  - select Trusted entity type: AWS Service
  - Use case: Lambda
  - attach these policies:
	- AWSLambdaVPCAccessExecutionRole policy attached (so that your Lambda function can access resources in your VPC)
	- AWSLambdaBasicExecutionRole policy attached (so that your Lambda function can write logs to CloudWatch)
  - Click Next
  - Name the Role: LambdaCRUDRole
  - Add a description and Tags if you wish.
  - Click Create Role

  - Click on your Role name to view it.
  - Under Permissions, Click the Add permissions button and select Create inline policy.
  - Click the JSON tab.
  - Replace <SECRET-ARN> with the ARN for your secret "dev/my-lambda/postgres" which you can copy from SecretsManager.
  - Paste in the policy below:

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": [
                "secretsmanager:GetSecretValue",
                "secretsmanager:GetResourcePolicy",
                "secretsmanager:DescribeSecret",
                "secretsmanager:ListSecretVersionIds"
            ],
            "Resource": "<SECRET-ARN>"
        },
        {
            "Effect": "Allow",
            "Action": "secretsmanager:ListSecrets",
            "Resource": "*"
        }
    ]
}
```

  - Click Review Policy
  - Name the policy: ReadSecretPolicy
  - Click Create policy

### Setup a Node.js Project

1. **Initialize a New Node.js Project:**
   - Create a project folder on your local machine called "my-CRUD-lambda"
   - Navigate inside that directory.
   - Run `npm init -y` in your terminal to create a new `package.json` file.
  
2. **Install Dependencies:**
   - Run `npm i -S pg @aws-sdk/client-secrets-manager`.

3. **Prepare the Package for the Layer:**
   - Create a directory named `nodejs`.
   - Move the `node_modules` directory and `package*.json` files into the `nodejs` directory. Your structure should look like this:
     ```
     nodejs
     ├── node_modules
     ├── package.json
     └── package-lock.json
     ```

4. **Create a `.zip` file:**
   - Zip the `nodejs` directory (including the directory itself).

> If you decide to npm install any other packages, ensure you run the command in the same folder as your package.json file.

## Create a Layer

  - If your zip file is larger than 10MB you need to upload to S3 first and not directly to Lambda. Even though this zip file is less than 10MB, we will use the S3 upload option as practice.
  - In the console navigate to S3
  - Click Create Bucket
  - Provide a bucket name, remember that this name must be globally unique!
  - Click "Create Bucket"
  - From the bucket list, click on your bucket.
  - Upload the .zip file containing your code.
  - After the upload completes, click on the file in S3, then copy the "Object URL" to your clipboard.

  - Navigate back to the lambda console
  - Click "Layers" from the left-hand navigation under "Additional resoures".
  - Click "Create layer"
  - Name: my-aws-pg-layer
  - Select "Upload a file from Amazon S3" and paste in the Object URL from your clipboard.
  - Under "runtimes" select Node.js
  - Click "Create".

## Create a Lambda Function

In the AWS console, go to Lambda and create a new function. Select the following configuration:

  - "Author from scratch" should be selected.
  - Function name: 'myCRUDLambda'
  - Runtime: Node.js
  - Click Create function

### Set Role

  - under the Configuration tab, select Permisions:
    - Click Edit for Execution role
    - For `existing role` select the role you created in the previously (should be called LambdaCRUDRole)
  - Click Save

### Add VPC Configuration to the Lambda Function

  - Under the Configuration tab click VPC on the left.
  - Click Edit
  - Select the VPC where your PostgreSQL database resides.
  - Select the private subnet.
  - Select the security group for your database, I called mine "my-RDS-psql-SG" (if you're not sure, look in the RDS console, select your database and look for "VPC security groups" under "Connectivity & security")
    - It should have a rule on port 5432, this is the group you want to use.
  - Click Save

### Change Security Group for Your Lambda Function

 - Under Configuration, select VPC
 - In VPC, click on the link for your Security Group
 - Edit inbound rules
 - Add a rule of type PostgreSQL
 - Click the source search box and select this same security group (this group needs to accept traffic from itself...)
 - Click "Save rules"

> This might seem strange, but as both the source of the DB traffic, your Lambda function, and the destination, your DB, are in the same security group, the group must allow DB traffic from itself.  In production you might choose to use different security groups for your DB and Lambda functions and then the DB group would instead allow DB traffic from the Lambda security group.

### Setting an Environment Variable in AWS Lambda

**Add an Environment Variable:**
   - Under "Configuration" select the "Environment variables" section.
   - Click on "Edit" (on the top-right corner of the section).
   - Click on "Add environment variable".
   - In the "Key" field, input "region".
   - In the "Value" field, input "eu-west-2" (or whatever Region you are using).
   - Click on "Add environment variable".
   - Add another entry for: Key = "secretId" and Value = "dev/my-lambda/postgres"
   - Click on "Save" at the bottom of the page to apply your changes.

Now that you've set the environment variable in AWS Lambda, you can use this variable in your function's Node.js code.

### Add the layer

  - In the Lambda console, select your function.
  - In the "Function overview" click "Layers"
  - In the "Layers" section click "Add a layer"
  - Under "Choose a layer" select "Custom layers"
  - From the dropdown select your layer.
  - From "Version" select 1.
  - Click "Add"

## Add the code

   - In the Code tab, under "Code source" right-click on the myCRUDKambda folder and select "New File" call it "index.js"
   - Double-click index.js to edit it.
   - Paste in the code below:
```javascript
const {
  SecretsManagerClient,
  GetSecretValueCommand,
} = require("@aws-sdk/client-secrets-manager");
const { Client } = require('pg');

const region = process.env.region;
const secretId = process.env.secretId;

const secretsManager = new SecretsManagerClient({
  region,
});

let cachedDbCredentials;

async function getDbCredentials() {
	if (cachedDbCredentials) return cachedDbCredentials;

	let secretData;
	try {
	  secretData = await secretsManager.send(
	    new GetSecretValueCommand({
	      SecretId: secretId,
	      VersionStage: "AWSCURRENT", // VersionStage defaults to AWSCURRENT if unspecified
	    })
	  );
	} catch (error) {
	  // For a list of exceptions thrown, see
	  // https://docs.aws.amazon.com/secretsmanager/latest/apireference/API_GetSecretValue.html
	  throw error;
	}

	const { username, password, host, port, dbname } = JSON.parse(secretData.SecretString);

	cachedDbCredentials = {
		host,
		database: dbname,
		port,
		user: username,
		password
	};

	return cachedDbCredentials
}

exports.handler = async (event, context) => {
	const { operation, id, message } = event;
	console.log({ operation, id, message })

	const dbCredentials = await getDbCredentials();
	
	const client = new Client(dbCredentials);
	await client.connect();

	let result;
	switch (operation) {
		case 'create':
			result = await client.query('INSERT INTO demo_table (id, message, created_date) VALUES (uuid_generate_v4(), $1, NOW()) RETURNING *', [message]);
			break;
		case 'read':
			result = await client.query('SELECT * FROM demo_table WHERE id = $1', [id]);
			break;
		case 'update':
			result = await client.query('UPDATE demo_table SET message = $1 WHERE id = $2 RETURNING *', [message, id]);
			break;
		case 'delete':
			result = await client.query('DELETE FROM demo_table WHERE id = $1 RETURNING *', [id]);
			break;
		default:
			return 'Invalid operation';
	}

	await client.end();

	return result.rows
};
```

  - Click "Deploy"

## Test Your Lambda Function

Time to test the function.

In the Test tab, enter an "Event name", Add the JSON below to the "Event JSON" area:
```json
{ 
    "operation": "create",
    "message": "First create"
}
```
Then click "Test"

Try running it a few more times with different values.

## Verify Data in RDS

1. ssh to your EC2
2. Connect to DB
3. Run: 

```sql
SELECT * FROM demo_table;
```

You should see the data you added from your Lambda tests in your RDS DB.
