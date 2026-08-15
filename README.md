# 🚀 Veeam Data Platform Premium v13 — AWS CloudFormation Template

Deploy automático do **Veeam Data Platform Premium v13** em um **Windows Server 2025** na AWS com rede isolada (SSM only).

---

## 📋 O que é este template?

Um template CloudFormation **100% pronto para produção** que:

✅ Cria uma **VPC privada isolada** (sem Internet Gateway, sem NAT)  
✅ Configura **VPC Endpoints** para acesso SSM Session Manager  
✅ Provisiona um **EC2 Windows Server 2025** com 2 discos:
   - **C:** 120 GB (Sistema Operacional)
   - **D:** 450 GB (Repositório Veeam)  
✅ **Baixa automaticamente** ISO e license do Veeam do S3  
✅ Aplica **IMDSv2 obrigatório** (proteção contra SSRF)  
✅ Restringe egress a apenas S3 e endpoints de SSM  

---

## 📦 Pré-requisitos

Antes de começar, você precisa:

1. **Conta AWS** com permissões para criar VPC, EC2, IAM, VPC Endpoints
2. **AWS CLI v2** instalado ([guia aqui](https://docs.aws.amazon.com/cli/latest/userguide/getting-started-install.html))
3. **Veeam ISO e License** já uploadados para um **S3 bucket** na sua região
4. **Conhecer o Prefix List ID** do seu S3 na região (comando abaixo)

---

## 🔧 Passo a Passo

### 1️⃣ Prepare seus arquivos no S3

Faça upload do **ISO** e **License** do Veeam para seu bucket S3:

```bash
# Exemplo (ajuste os caminhos)
aws s3 cp VeeamDataPlatformPremium_v13.1.iso \
  s3://seu-bucket/ISO-Veeam/ --region us-east-2

aws s3 cp veeam_license.lic \
  s3://seu-bucket/ --region us-east-2
```

### 2️⃣ Encontre o Prefix List ID do S3

Para sua região, execute:

```bash
aws ec2 describe-prefix-lists \
  --filters Name=prefix-list-name,Values=com.amazonaws.[REGION].s3 \
  --query "PrefixLists[0].PrefixListId" \
  --output text \
  --region [REGION]
```

**Exemplo para us-east-2:**
```bash
aws ec2 describe-prefix-lists \
  --filters Name=prefix-list-name,Values=com.amazonaws.us-east-2.s3 \
  --query "PrefixLists[0].PrefixListId" \
  --output text \
  --region us-east-2
```

Você receberá algo como: `pl-12345678` — **guarde este valor!**

### 3️⃣ Baixe o Template

Clone este repositório ou copie o arquivo `VeeamWizzard-V1-Template.yaml`

### 4️⃣ Preencha os Parâmetros

Abra `VeeamWizzard-V1-Template.yaml` e substitua:

| Placeholder | Exemplo | Descrição |
|---|---|---|
| `[ENTER_YOUR_BUCKET_NAME]` | `meu-bucket-veeam` | Nome do seu bucket S3 |
| `[ENTER_YOUR_S3_ISO_PATH_HERE]` | `s3://meu-bucket-veeam/ISO-Veeam/VeeamDataPlatformPremium_v13.1.iso` | Caminho completo do ISO no S3 |
| `[ENTER_YOUR_S3_LICENSE_PATH_HERE]` | `s3://meu-bucket-veeam/veeam_license.lic` | Caminho completo da license no S3 |
| `[ENTER_YOUR_S3_PREFIX_LIST_ID_HERE]` | `pl-12345678` | Prefix List ID que você encontrou no passo 2 |

### 5️⃣ Deploy via AWS Console (Mais Fácil)

1. Vá para [AWS CloudFormation Console](https://console.aws.amazon.com/cloudformation)
2. Clique em **Create Stack** → **With new resources**
3. Cole o conteúdo do arquivo YAML na seção **Template**
4. Clique **Next** e preencha os parâmetros
5. Revise as permissões e clique **Create Stack**

### 6️⃣ Deploy via CLI (Avançado)

```bash
aws cloudformation create-stack \
  --stack-name veeam-prod-stack \
  --template-body file://VeeamWizzard-V1-Template.yaml \
  --parameters \
      ParameterKey=EnvironmentName,ParameterValue=veeam-prod \
      ParameterKey=AWSRegion,ParameterValue=us-east-2 \
      ParameterKey=S3BucketName,ParameterValue=meu-bucket-veeam \
      ParameterKey=VeeamISOUrl,ParameterValue=s3://meu-bucket-veeam/ISO-Veeam/VeeamDataPlatformPremium_v13.1.iso \
      ParameterKey=VeeamLicenseUrl,ParameterValue=s3://meu-bucket-veeam/veeam_license.lic \
      ParameterKey=S3PrefixListId,ParameterValue=pl-12345678 \
  --capabilities CAPABILITY_NAMED_IAM \
  --region us-east-2
```

---

## ⏱️ Tempo de Deploy

Geralmente leva **5-10 minutos**. Você pode acompanhar:

- **Via Console:** CloudFormation → Stacks → Seu Stack → Events
- **Via CLI:**
```bash
aws cloudformation describe-stacks \
  --stack-name veeam-prod-stack \
  --query 'Stacks[0].StackStatus' \
  --region us-east-2
```

---

## 🖥️ Acessar a Instância

Após o deploy, use **AWS Systems Manager Session Manager** (sem SSH, sem RDP aberto):

### Opção 1: Via Console
1. Vá para [EC2 Console](https://console.aws.amazon.com/ec2)
2. Selecione a instância `veeam-prod-server`
3. Clique em **Connect** → **Session Manager**

### Opção 2: Via CLI
```bash
aws ssm start-session --target i-xxxxxxxxx --region us-east-2
```

(Substitua `i-xxxxxxxxx` pelo Instance ID, disponível nos Outputs do Stack)

---

## 📂 Arquivos no Servidor

Após o bootstrap completar, você terá:

```
D:\Veeam\
├── ISO\
│   └── VeeamDataPlatformPremium_v13.1.iso
├── License\
│   └── veeam_license.lic
└── Logs\
    └── userdata.log
```

**Desktop:** Um arquivo `VEEAM_INSTALL_README.txt` com instruções

---

## 🔐 Segurança

✅ **IMDSv2** obrigatório (proteção contra SSRF)  
✅ **Sem acesso à internet** (isolado em subnet privada)  
✅ **Egress restrito** apenas a S3 e VPC Endpoints  
✅ **Sem portas abertas** (acesso apenas via SSM)  
✅ **Discos criptografados** (EBS encryption habilitada)  
✅ **Sem Security Group inbound** (zero exposição)  

---

## 📊 Outputs do Stack

Após deploy, você receberá:

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

## ❌ Deletar o Stack (Cleanup)

Para remover tudo e evitar custos:

```bash
aws cloudformation delete-stack \
  --stack-name veeam-prod-stack \
  --region us-east-2
```

---

## 🐛 Troubleshooting

### **Erro: "S3BucketName is required"**
Você deixou um placeholder `[ENTER_YOUR_...]` sem substituir. Verifique todos os parâmetros.

### **Erro: "Access Denied" ao baixar ISO do S3**
- Verifique se a IAM Role tem permissão `s3:GetObject` no bucket
- Confirme o caminho S3 está correto

### **Disco D: não aparece na instância**
- Acesse via SSM e verifique o log: `D:\Veeam\Logs\userdata.log`
- Pode levar alguns minutos para o disco ser reconhecido

### **VPC Endpoints não conectam**
- Confirme que `S3PrefixListId` está correto
- Verifique a Security Group do endpoint permite HTTPS (porta 443) da VPC CIDR

---

## 💡 Customizações Comuns

### Mudar o tamanho da instância
Edite o parâmetro `InstanceType` para `m5.2xlarge`, `c5.xlarge`, etc.

### Mudar o tamanho dos discos
No `BlockDeviceMappings`, altere `VolumeSize` (em GB)

### Mudar a região
Atualize `AWSRegion` e `S3PrefixListId` conforme sua região

---

## 📚 Documentação Referência

- [AWS CloudFormation](https://docs.aws.amazon.com/cloudformation/)
- [Veeam Data Platform](https://www.veeam.com/data-platform.html)
- [AWS Systems Manager Session Manager](https://docs.aws.amazon.com/systems-manager/latest/userguide/session-manager.html)
- [VPC Endpoints](https://docs.aws.amazon.com/vpc/latest/privatelink/vpc-endpoints.html)

---

## 📝 Licença

Este template é fornecido **as-is** para fins educacionais. Ajuste conforme necessário para sua infraestrutura.

---

## 🤝 Suporte

Dúvidas? Abra uma **Issue** neste repositório! 

Boa sorte com o Veeam! 🚀
