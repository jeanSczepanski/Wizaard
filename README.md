# 🚀 Veeam Data Platform Premium v13 — AWS CloudFormation Template

Automated deployment of **Veeam Data Platform Premium v13** on **Windows Server 2025** in AWS with an isolated network (SSM only).

---

## 📋 What is this template?

A **production-ready** CloudFormation template that:

✅ Creates an **isolated private VPC** (no Internet Gateway, no NAT)  
✅ Configures **VPC Endpoints** for SSM Session Manager access  
✅ Provisions an **EC2 Windows Server 2025** with 2 disks:
   - **C:** 120 GB (Operating System)
   - **D:** 450 GB (Veeam Repository)  
✅ **Automatically downloads** Veeam ISO and license from S3  
✅ Enforces **IMDSv2** (protection against SSRF)  
✅ Restricts egress to only S3 and SSM endpoints  

---

## 📦 Prerequisites

Before you start, you need:

1. **AWS Account** with permissions to create VPC, EC2, IAM, VPC Endpoints
2. **AWS CLI v2** installed ([guide here](https://docs.aws.amazon.com/cli/latest/userguide/getting-started-install.html))
3. **Veeam ISO and License** already uploaded to an **S3 bucket** in your region
4. **Know the Prefix List ID** of your S3 in the region (command below)

---

## 🔧 Step by Step

### 1️⃣ Prepare Your Files in S3

Upload the **ISO** and **License** files to your S3 bucket:

```bash
# Example (adjust paths)
aws s3 cp VeeamDataPlatformPremium_v13.1.iso \
  s3://your-bucket/ISO-Veeam/ --region us-east-2

aws s3 cp veeam_license.lic \
  s3://your-bucket/ --region us-east-2
```

### 2️⃣ Find the S3 Prefix List ID

For your region, run:

```bash
aws ec2 describe-prefix-lists \
  --filters Name=prefix-list-name,Values=com.amazonaws.[REGION].s3 \
  --query "PrefixLists[0].PrefixListId" \
  --output text \
  --region [REGION]
```

**Example for us-east-2:**
```bash
aws ec2 describe-prefix-lists \
  --filters Name=prefix-list-name,Values=com.amazonaws.us-east-2.s3 \
  --query "PrefixLists[0].PrefixListId" \
  --output text \
  --region us-east-2
```

You will receive something like: `pl-12345678` — **save this value!**

### 3️⃣ Download the Template

Clone this repository or copy the file `VeeamWizzard-V1-Template.yaml`

### 4️⃣ Fill in the Parameters

Open `VeeamWizzard-V1-Template.yaml` and replace:

| Placeholder | Example | Description |
|---|---|---|
| `[ENTER_YOUR_BUCKET_NAME]` | `my-veeam-bucket` | Name of your S3 bucket |
| `[ENTER_YOUR_S3_ISO_PATH_HERE]` | `s3://my-veeam-bucket/ISO-Veeam/VeeamDataPlatformPremium_v13.1.iso` | Full path of ISO in S3 |
| `[ENTER_YOUR_S3_LICENSE_PATH_HERE]` | `s3://my-veeam-bucket/veeam_license.lic` | Full path of license in S3 |
| `[ENTER_YOUR_S3_PREFIX_LIST_ID_HERE]` | `pl-12345678` | Prefix List ID found in step 2 |

### 5️⃣ Deploy via AWS Console (Easier)

1. Go to [AWS CloudFormation Console](https://console.aws.amazon.com/cloudformation)
2. Click **Create Stack** → **With new resources**
3. Paste the YAML file content in the **Template** section
4. Click **Next** and fill in the parameters
5. Review permissions and click **Create Stack**

### 6️⃣ Deploy via CLI (Advanced)

```bash
aws cloudformation create-stack \
  --stack-name veeam-prod-stack \
  --template-body file://VeeamWizzard-V1-Template.yaml \
  --parameters \
      ParameterKey=EnvironmentName,ParameterValue=veeam-prod \
      ParameterKey=AWSRegion,ParameterValue=us-east-2 \
      ParameterKey=S3BucketName,ParameterValue=my-veeam-bucket \
      ParameterKey=VeeamISOUrl,ParameterValue=s3://my-veeam-bucket/ISO-Veeam/VeeamDataPlatformPremium_v13.1.iso \
      ParameterKey=VeeamLicenseUrl,ParameterValue=s3://my-veeam-bucket/veeam_license.lic \
      ParameterKey=S3PrefixListId,ParameterValue=pl-12345678 \
  --capabilities CAPABILITY_NAMED_IAM \
  --region us-east-2
```

---

## ⏱️ Deployment Time

Usually takes **5-10 minutes**. You can monitor via:

- **Console:** CloudFormation → Stacks → Your Stack → Events
- **CLI:**
```bash
aws cloudformation describe-stacks \
  --stack-name veeam-prod-stack \
  --query 'Stacks[0].StackStatus' \
  --region us-east-2
```

---

## 🖥️ Access the Instance

After deployment, use **AWS Systems Manager Session Manager** (no SSH, no RDP open):

### Option 1: Via Console
1. Go to [EC2 Console](https://console.aws.amazon.com/ec2)
2. Select the instance `veeam-prod-server`
3. Click **Connect** → **Session Manager**

### Option 2: Via CLI
```bash
aws ssm start-session --target i-xxxxxxxxx --region us-east-2
```

(Replace `i-xxxxxxxxx` with the Instance ID from Stack Outputs)

---

## 📂 Files on the Server

After bootstrap completes, you will have:

```
D:\Veeam\
├── ISO\
│   └── VeeamDataPlatformPremium_v13.1.iso
├── License\
│   └── veeam_license.lic
└── Logs\
    └── userdata.log
```

**Desktop:** A `VEEAM_INSTALL_README.txt` file with instructions

---

## 🔐 Security

✅ **IMDSv2** enforced (SSRF protection)  
✅ **No internet access** (isolated in private subnet)  
✅ **Restricted egress** only to S3 and VPC Endpoints  
✅ **No open ports** (access only via SSM)  
✅ **Encrypted disks** (EBS encryption enabled)  
✅ **No Security Group inbound rules** (zero exposure)  

---

## 📊 Stack Outputs

After deployment, you will receive:

```
VpcId: vpc-xxxxx
PrivateSubnetId: subnet-xxxxx
InstanceId: i-xxxxx
InstancePrivateIP: 10.10.1.xxx
SSMConnectCommand: aws ssm start-session --target i-xxxxx --region us-east-2
ISOPath: D:\Veeam\ISO\VeeamDataPlatformPremium_v13.1.iso
LicensePath: D:\Veeam\License\veeam_license.lic
```

---

## ❌ Delete the Stack (Cleanup)

To remove everything and avoid costs:

```bash
aws cloudformation delete-stack \
  --stack-name veeam-prod-stack \
  --region us-east-2
```

---

## 🐛 Troubleshooting

### **Error: "S3BucketName is required"**
You left a placeholder `[ENTER_YOUR_...]` without replacing. Check all parameters.

### **Error: "Access Denied" when downloading ISO from S3**
- Verify that the IAM Role has `s3:GetObject` permission on the bucket
- Confirm the S3 path is correct

### **D: drive not appearing in the instance**
- Access via SSM and check the log: `D:\Veeam\Logs\userdata.log`
- The disk may take a few minutes to be recognized

### **VPC Endpoints not connecting**
- Confirm that `S3PrefixListId` is correct
- Check that the endpoint Security Group allows HTTPS (port 443) from VPC CIDR

---

## 💡 Common Customizations

### Change the instance size
Edit the `InstanceType` parameter to `m5.2xlarge`, `c5.xlarge`, etc.

### Change disk sizes
In `BlockDeviceMappings`, change `VolumeSize` (in GB)

### Change the region
Update `AWSRegion` and `S3PrefixListId` according to your region

---

## 📚 Reference Documentation

- [AWS CloudFormation](https://docs.aws.amazon.com/cloudformation/)
- [Veeam Data Platform](https://www.veeam.com/data-platform.html)
- [AWS Systems Manager Session Manager](https://docs.aws.amazon.com/systems-manager/latest/userguide/session-manager.html)
- [VPC Endpoints](https://docs.aws.amazon.com/vpc/latest/privatelink/vpc-endpoints.html)

---

## 📝 License

This template is provided **as-is** for educational purposes. Adjust as needed for your infrastructure.

---

## 🤝 Support

Have questions? Open an **Issue** in this repository! 

Good luck with Veeam! 🚀
