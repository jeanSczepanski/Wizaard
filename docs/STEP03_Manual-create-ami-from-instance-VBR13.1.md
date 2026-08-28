# Create an AMI from an Existing EC2 Instance (CloudFormation)

This AWS CloudFormation template creates an **AMI** from an already existing EC2 instance, using a Lambda function triggered by a **Custom Resource**.

## How it works

1. Creates an **IAM Role** for the Lambda, with permissions for `CreateImage`, `DescribeImages`, `CreateTags`, and CloudWatch logs.
2. Creates a **Lambda function** (Python 3.12) that runs `ec2.create_image()` against the given instance.
3. A **Custom Resource** triggers the Lambda, passing the `InstanceId` and `AmiName`.
4. The ID of the created AMI is exposed as an **Output** (`AmiId`).

> ⚠️ **Note:** The `NoReboot=False` parameter causes the instance to reboot during AMI creation, ensuring filesystem consistency. If you'd rather avoid the reboot (less safe), set it to `True`.

---

## Template

```yaml
AWSTemplateFormatVersion: '2010-09-09'
Description: Create AMI from existing EC2 instance
Resources:
  # IAM Role for Lambda
  CreateAmiLambdaRole:
    Type: AWS::IAM::Role
    Properties:
      RoleName: <REPLACE_HERE_ROLE_NAME>
      AssumeRolePolicyDocument:
        Version: '2012-10-17'
        Statement:
          - Effect: Allow
            Principal:
              Service: lambda.amazonaws.com
            Action: sts:AssumeRole
      Policies:
        - PolicyName: CreateAmiPolicy
          PolicyDocument:
            Version: '2012-10-17'
            Statement:
              - Effect: Allow
                Action:
                  - ec2:CreateImage
                  - ec2:DescribeImages
                  - ec2:CreateTags
                Resource: '*'
              - Effect: Allow
                Action:
                  - logs:CreateLogGroup
                  - logs:CreateLogStream
                  - logs:PutLogEvents
                Resource: '*'

  # Lambda Function to create the AMI
  CreateAmiFunction:
    Type: AWS::Lambda::Function
    Properties:
      FunctionName: <REPLACE_HERE_LAMBDA_FUNCTION_NAME>
      Runtime: python3.12
      Handler: index.handler
      Role: !GetAtt CreateAmiLambdaRole.Arn
      Timeout: 60
      Code:
        ZipFile: |
          import boto3
          import cfnresponse
          def handler(event, context):
            try:
              if event['RequestType'] == 'Create':
                ec2 = boto3.client('ec2', region_name='<REPLACE_HERE_REGION>')
                instance_id = event['ResourceProperties']['InstanceId']
                ami_name = event['ResourceProperties']['AmiName']
                response = ec2.create_image(
                  InstanceId=instance_id,
                  Name=ami_name,
                  Description='AMI created via CloudFormation',
                  NoReboot=False  # Set to True to skip reboot (less safe for Windows)
                )
                ami_id = response['ImageId']
                cfnresponse.send(event, context, cfnresponse.SUCCESS, {'AmiId': ami_id})
              else:
                cfnresponse.send(event, context, cfnresponse.SUCCESS, {})
            except Exception as e:
              cfnresponse.send(event, context, cfnresponse.FAILED, {'Error': str(e)})

  # Custom Resource that triggers the Lambda
  CreateAmiCustomResource:
    Type: Custom::CreateAmi
    Properties:
      ServiceToken: !GetAtt CreateAmiFunction.Arn
      InstanceId: <REPLACE_HERE_INSTANCE_ID>
      AmiName: <REPLACE_HERE_AMI_NAME>

Outputs:
  AmiId:
    Description: The ID of the created AMI
    Value: !GetAtt CreateAmiCustomResource.AmiId
```

---

## Fields to replace

| Field | Description | Example |
|---|---|---|
| `<REPLACE_HERE_ROLE_NAME>` | Name of the IAM Role used by the Lambda | `CreateAmiLambdaRole` |
| `<REPLACE_HERE_LAMBDA_FUNCTION_NAME>` | Name of the Lambda function | `CreateAmiFromInstance` |
| `<REPLACE_HERE_REGION>` | AWS region where the source instance is located | `us-east-2` |
| `<REPLACE_HERE_INSTANCE_ID>` | ID of the source EC2 instance | `i-0710ba3f3f0922fe4` |
| `<REPLACE_HERE_AMI_NAME>` | Name the created AMI will have | `veeam-prod-server-ami` |

---

## Deploy via AWS CLI

```bash
aws cloudformation create-stack \
  --stack-name create-ami-stack \
  --template-body file://create-ami-from-instance.yaml \
  --capabilities CAPABILITY_NAMED_IAM
```

## Notes

- The IAM Role name (`RoleName`) is hardcoded — if you plan to reuse this template across multiple stacks, consider removing `RoleName` and letting CloudFormation auto-generate one to avoid naming conflicts.
- The region passed to `boto3.client('ec2', region_name=...)` must match the region of the instance specified in `InstanceId`.
- If the stack is deleted, the Custom Resource does not delete the created AMI (the Lambda simply returns success in the `else` branch, with no delete/rollback logic).
