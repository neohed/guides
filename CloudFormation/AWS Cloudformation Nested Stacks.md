# AWS CloudFormation Nested Stacks

[Nested Stack Parameters - Output vs Export](https://stackoverflow.com/questions/67605906/nested-cloudformation-child-template-not-picking-up-parameter)
[Output vs Export](https://stackoverflow.com/questions/62441152/difference-between-an-output-an-export)

## Conditions

https://saturncloud.io/blog/cloudformation-conditional-parameters/#:~:text=Conditional%20parameters%20are%20a%20powerful,another%20parameter%20or%20a%20condition.
https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/intrinsic-function-reference-conditions.html
https://aws.amazon.com/blogs/infrastructure-and-automation/conditionally-launch-aws-cloudformation-resources-based-on-user-input/

1. Parent Stack (caller)

```yaml
AWSTemplateFormatVersion: 2010-09-09
Resources:
  NestedStackCreateS3:
    Type: AWS::CloudFormation::Stack
	Properties:
	  TemplateURL: https://s3-bucket-url-to-nested-stack-template...
	  TimeoutInMinutes: 60
Outputs:
  StackRef:
    Value: !Ref NestedStackCreateS3
  OutputFromNestedStack:
    Value: !GetAtt NestedStackCreateS3.Outputs.BucketName
```

2. Nested Stack (callee)

Upload this nested stack to an S3 bucket, e.g: https://s3-bucket-url-to-nested-stack-template... so it can be referenced by the parent stack template.

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