# AWS CloudFormation Nested Stacks

1. Parent Stack (caller)

```yaml
AWSTemplateFormatVersion: 2010-09-09
Resources:
  NestedStackVPC:
    Type: AWS::CloudFormation::Stack
	Properties:
	  TemplateURL: https://s3-bucket-url...
	  TimeoutInMinutes: 60
Outputs:
  StackRef:
    Value: !Ref NestedStackVPC
  OutputFromNestedStack:
    Value: !GetAtt NestedStackVPC.Outputs.BucketName
```

2. Child stack (callee)

```yaml
AWSTemplateFormatVersion: 2010-09-09
Resources:
  S3Bucket:
    Type: AWS::S3::Bucket
Outputs:
  BucketName:
    Value: !Ref 'S3Bucket'
	Description: Name of the S3 bucket
```