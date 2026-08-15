# Arquitetura Wizaard

## Visão Geral

Este documento descreve a arquitetura da aplicação Wizaard, que utiliza componentes AWS para gerenciar infraestrutura em nuvem.

## Diagrama de Arquitetura

![Diagrama de Arquitetura Wizaard](./architecture-diagram.png)

## Componentes

### VeemServer
- **InstanceRole**: Papel de instância para permissões
- **InstanceProfile**: Perfil de instância
- **VeemServer**: Servidor principal

### Endpoints
- **S3GatewayEndpoint**: Endpoint para acesso ao S3
- **SSMEndpoint**: Endpoint para Systems Manager
- **SSMMessagesEndpoint**: Endpoint para mensagens do Systems Manager
- **EC2MessagesEndpoint**: Endpoint para mensagens do EC2

### Rede
- **PrivateSubnet**: Sub-rede privada contendo:
  - PrivateSubnet
  - PrivateSubnetRouteTableAssociation
  - PrivateRouteTable
- **VPC**: Virtual Private Cloud

### Segurança
- **InstanceSecurityGroup**: Grupo de segurança para instâncias
- **EndpointSecurityGroup**: Grupo de segurança para endpoints

---

*Última atualização: 2026-08-15*
