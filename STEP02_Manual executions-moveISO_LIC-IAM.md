In these scenarios, Amazon S3 can still be used as a secure repository for software, licenses, and installation packages by leveraging VPC Endpoints and IAM permissions.

In my AWS + Veeam lab, I stored a Veeam Data Platform license file in Amazon S3 and retrieved it directly from an EC2 instance running in a private network.

## PowerShell Command

```powershell
Read-S3Object -BucketName "YOUR-S3-BUCKET-NAME" `
              -Key "LICENSE-FILE.lic" `
              -File "C:\VeeamDownloads\LICENSE-FILE.lic" `
              -Region "AWS-REGION"
```

## Required IAM Permissions

The EC2 instance role requires:

- S3 permissions to list and download objects.
- KMS permissions to decrypt objects encrypted with AWS KMS.

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "S3ReadAccess",
      "Effect": "Allow",
      "Action": [
        "s3:GetObject",
        "s3:HeadObject",
        "s3:ListBucket"
      ],
      "Resource": [
        "arn:aws:s3:::YOUR-S3-BUCKET-NAME",
        "arn:aws:s3:::YOUR-S3-BUCKET-NAME/*"
      ]
    },
    {
      "Sid": "KMSDecryptForS3",
      "Effect": "Allow",
      "Action": [
        "kms:Decrypt",
        "kms:GenerateDataKey"
      ],
      "Resource": "arn:aws:kms:AWS-REGION:123456789012:key/00000000-0000-0000-0000-000000000000"
    }
  ]
}
```

## Prerequisites

- AWS Tools for PowerShell installed on the EC2 instance.
- An IAM role attached to the EC2 instance with the permissions listed above.
- Connectivity to Amazon S3 through a VPC Endpoint (Gateway Endpoint) or another private networking path.
- Access to the AWS KMS key used to encrypt the S3 object.

## Benefits

- No Internet access required from the EC2 instance.
- Secure file storage using Amazon S3 and AWS KMS.
- Simplified software and license distribution.
- Alignment with AWS security best practices for private workloads.
