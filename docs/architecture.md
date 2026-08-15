# Wizaard Architecture

## Overview

This document describes the architecture of the Wizaard application, which utilizes AWS components to manage cloud infrastructure.

## Architecture Diagram

![Wizaard Architecture Diagram](./architecture-diagram.png)

## Components

### VeemServer
- **InstanceRole**: Instance role for permissions
- **InstanceProfile**: Instance profile
- **VeemServer**: Main server

### Endpoints
- **S3GatewayEndpoint**: Endpoint for S3 access
- **SSMEndpoint**: Endpoint for Systems Manager
- **SSMMessagesEndpoint**: Endpoint for Systems Manager messages
- **EC2MessagesEndpoint**: Endpoint for EC2 messages

### Network
- **PrivateSubnet**: Private subnet containing:
  - PrivateSubnet
  - PrivateSubnetRouteTableAssociation
  - PrivateRouteTable
- **VPC**: Virtual Private Cloud

### Security
- **InstanceSecurityGroup**: Security group for instances
- **EndpointSecurityGroup**: Security group for endpoints

---

*Last updated: 2026-08-15*
