# AWS CloudFormation Nested Stacks

[Nested Stack Parameters - Output vs Export](https://stackoverflow.com/questions/67605906/nested-cloudformation-child-template-not-picking-up-parameter)
[Output vs Export](https://stackoverflow.com/questions/62441152/difference-between-an-output-an-export)

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