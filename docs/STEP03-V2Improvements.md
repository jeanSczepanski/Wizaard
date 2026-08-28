# Security Hardening — Veeam CloudFormation Template

This document summarizes the security controls implemented in this template,
beyond the base architecture that was requested (VPC, Subnet, EC2, IAM Role,
Endpoints, Security Groups).

---

## 1. Endpoint policies restricting reachability (not a simple "allow all")

All 4 VPC Endpoints (SSM, SSM Messages, EC2 Messages, and S3 Gateway) carry
an explicit `PolicyDocument` — without it, the endpoint would accept any
request from any principal.

**SSM / SSM Messages / EC2 Messages** — restrict usage to the same AWS account:

```yaml
  SSMEndpoint:
    Type: AWS::EC2::VPCEndpoint
    Properties:
      ...
      PolicyDocument:
        Version: '2012-10-17'
        Statement:
          - Sid: AllowSameAccountOnly
            Effect: Allow
            Principal: "*"
            Action: "*"
            Resource: "*"
            Condition:
              StringEquals:
                aws:PrincipalAccount: !Ref AWS::AccountId
```

**S3 Gateway Endpoint** — even more restrictive, scoped to specific buckets:

```yaml
  S3GatewayEndpoint:
    Type: AWS::EC2::VPCEndpoint
    Properties:
      ...
      PolicyDocument:
        Version: '2012-10-17'
        Statement:
          - Sid: AllowInstallerBucket
            Effect: Allow
            Principal: "*"
            Action:
              - s3:GetObject
              - s3:GetBucketLocation
              - s3:ListBucket
            Resource:
              - !Sub arn:aws:s3:::${S3BucketName}
              - !Sub arn:aws:s3:::${S3BucketName}/*
          - Sid: AllowAwsCliInstallerBucket
            Effect: Allow
            Principal: "*"
            Action:
              - s3:GetObject
            Resource:
              - arn:aws:s3:::aws-cli
              - arn:aws:s3:::aws-cli/*
```

> Result: even if the instance's IAM Role had broader permissions, the
> endpoint itself blocks access to any bucket/resource outside what is
> explicitly allowed.

---

## 2. Explicit removal of the default "allow all" egress rule

By default, every new Security Group created by AWS ships with a
`0.0.0.0/0` egress rule (allow all outbound). This is **not removed** just
by adding extra rules via `AWS::EC2::SecurityGroupEgress` — the default rule
stays active alongside them. To actually remove it, `SecurityGroupEgress`
must be declared inline on the SG resource itself.

**EndpointSecurityGroup:**

```yaml
  EndpointSecurityGroup:
    Type: AWS::EC2::SecurityGroup
    Properties:
      GroupDescription: Allow HTTPS inbound from VeeamServer SG only; no outbound needed.
      VpcId: !Ref VPC
      SecurityGroupEgress:
        - IpProtocol: "-1"
          CidrIp: 127.0.0.1/32
          Description: "Placeholder — removes default allow-all-outbound. No real egress needed (SG is stateful)."
```

**InstanceSecurityGroup:**

```yaml
  InstanceSecurityGroup:
    Type: AWS::EC2::SecurityGroup
    Properties:
      GroupDescription: Veeam server - no inbound, egress restricted to VPC endpoints only
      VpcId: !Ref VPC
      SecurityGroupEgress:
        - IpProtocol: "-1"
          CidrIp: 127.0.0.1/32
          Description: "Placeholder — removes default allow-all-outbound. Real egress defined via explicit rules below."
```

> The actual egress (443 to the endpoints and to the S3 prefix list) is
> defined separately via `AWS::EC2::SecurityGroupEgress`, without relying on
> the default allow-all rule.

---

## 3. SG-to-SG reference (Endpoint SG only accepts 443 from the Instance SG)

Instead of opening HTTPS to the entire VPC CIDR, the `EndpointSecurityGroup`
ingress rule references the `InstanceSecurityGroup` directly. This means
**only** the Veeam instance (and any future resource that reuses that same
SG) can talk to the endpoints — not any host within the VPC's IP range.

```yaml
  EndpointSecurityGroupIngressFromInstance:
    Type: AWS::EC2::SecurityGroupIngress
    Properties:
      GroupId: !Ref EndpointSecurityGroup
      IpProtocol: tcp
      FromPort: 443
      ToPort: 443
      SourceSecurityGroupId: !Ref InstanceSecurityGroup
      Description: HTTPS from Veeam Server security group only
```

> Declared as a separate resource (rather than inline inside the SG) to
> avoid a circular dependency: this rule references `InstanceSecurityGroup`,
> and `InstanceSecurityGroup`'s egress rules reference `EndpointSecurityGroup`
> back.

---

## 4. Enforced IMDSv2 (SSRF mitigation)

Instance Metadata Service v1 has historically been an attack vector for
stealing IAM credentials via SSRF (e.g., the Capital One breach). Enforcing
IMDSv2 requires a session token for any metadata endpoint call, which
prevents this type of exploitation.

```yaml
      MetadataOptions:
        HttpTokens: required
        HttpPutResponseHopLimit: 1
        HttpEndpoint: enabled
```

---

## 5. EBS volume encryption

Both volumes (C: and D:) are created with `Encrypted: true`, protecting
data at rest — including the backup repository volume, which typically
holds sensitive data from the environments protected by Veeam.

```yaml
      BlockDeviceMappings:
        - DeviceName: /dev/sda1 # Root volume → C:
          Ebs:
            VolumeSize: 120
            VolumeType: gp3
            Iops: 3000
            DeleteOnTermination: true
            Encrypted: true

        - DeviceName: /dev/xvdf # Secondary EBS → D:
          Ebs:
            VolumeSize: 450
            VolumeType: gp3
            Iops: 3000
            DeleteOnTermination: true
            Encrypted: true
```

---

## 6. Fully isolated architecture — no IGW/NAT

There is no `AWS::EC2::InternetGateway` or `AWS::EC2::NatGateway` resource
anywhere in the template. All communication with AWS (SSM, ISO/license
download via S3) happens through VPC Endpoints, within the AWS network —
never over the public internet.

**Subnet with no public IP:**

```yaml
  PrivateSubnet:
    Type: AWS::EC2::Subnet
    Properties:
      VpcId: !Ref VPC
      CidrBlock: !Ref PrivateSubnetCIDR
      AvailabilityZone: !Select
        - 0
        - !GetAZs ''
      MapPublicIpOnLaunch: false
```

**Private route table, with no `0.0.0.0/0` route:**

```yaml
  PrivateRouteTable:
    Type: AWS::EC2::RouteTable
    Properties:
      VpcId: !Ref VPC
```

**Administrative access via SSM (no RDP / open port):**

```yaml
      ManagedPolicyArns:
        - arn:aws:iam::aws:policy/AmazonSSMManagedInstanceCore
```

**Architectural intent declared in the template itself:**

```yaml
Description: >
  Veeam Data Platform Premium v13.1 on Windows Server 2025, 100% isolated
  network via SSM (no IGW/NAT). C: 120GB (OS), D: 450GB (repository).
```

---

## Summary

| # | Control | Purpose |
|---|---------|---------|
| 1 | Restrictive endpoint policies | Limits what is reachable through each VPC Endpoint, independent of the IAM Role |
| 2 | Removal of default allow-all egress | Ensures only explicitly permitted traffic leaves the instance |
| 3 | SG-to-SG reference | Reduces lateral attack surface within the VPC |
| 4 | Enforced IMDSv2 | Mitigates credential theft via SSRF |
| 5 | EBS Encrypted | Protects data at rest, including backups |
| 6 | No IGW/NAT | Eliminates exposure to the public internet |
