# AWS Session Manager desde CLI (sin abrir SSH/RDP)

## Objetivo
Configurar acceso remoto seguro a una instancia EC2 usando **AWS Systems Manager Session Manager** y **AWS CLI**, evitando exponer puertos de entrada 22/3389.

## Arquitectura recomendada
- La instancia EC2 no necesita IP publica para administracion.
- No se abre SSH (22) ni RDP (3389) en el Security Group.
- La conectividad de Systems Manager se hace por HTTPS (443):
  - Con salida a internet (NAT Gateway), o
  - Con VPC Endpoints privados (recomendado):
    - `com.amazonaws.<region>.ssm`
    - `com.amazonaws.<region>.ssmmessages`
    - `com.amazonaws.<region>.ec2messages`

## Prerrequisitos
1. AWS CLI instalado y configurado.
2. Session Manager Plugin instalado en el equipo local.
3. Permisos IAM para el operador (usuario/rol que usa CLI):
   - `ssm:StartSession`
   - `ssm:TerminateSession`
   - `ssm:ResumeSession`
   - `ssm:DescribeSessions`
   - `ssm:GetConnectionStatus`
   - `ssm:DescribeInstanceInformation`
   - `ec2:DescribeInstances`

## Paso 1: validar entorno local
```bash
aws sts get-caller-identity
session-manager-plugin --version
```

## Paso 2: crear rol IAM para la instancia
1. Trust policy para EC2.
2. Crear rol (ejemplo: `EC2RoleSSM`).
3. Adjuntar politica administrada: `AmazonSSMManagedInstanceCore`.
4. Crear instance profile (ejemplo: `EC2ProfileSSM`) y agregar el rol.

Comandos:
```bash
aws iam create-role \
  --role-name EC2RoleSSM \
  --assume-role-policy-document file://trust-ec2.json

aws iam attach-role-policy \
  --role-name EC2RoleSSM \
  --policy-arn arn:aws:iam::aws:policy/AmazonSSMManagedInstanceCore

aws iam create-instance-profile \
  --instance-profile-name EC2ProfileSSM

aws iam add-role-to-instance-profile \
  --instance-profile-name EC2ProfileSSM \
  --role-name EC2RoleSSM
```

## Paso 3: asociar instance profile a la EC2
- Si lanzas una instancia nueva: usa `--iam-instance-profile Name=EC2ProfileSSM`.
- Si la instancia ya existe: asocia el profile en caliente con `associate-iam-instance-profile`.

## Paso 4: preparar conectividad de red para SSM
### Opcion A: NAT Gateway
- La instancia tiene salida HTTPS (443) hacia endpoints de AWS.

### Opcion B (recomendada): VPC Endpoints privados
Crear endpoints de tipo Interface para `ssm`, `ssmmessages`, `ec2messages`.

Ejemplo:
```bash
aws ec2 create-vpc-endpoint \
  --vpc-id vpc-xxxxxxxx \
  --vpc-endpoint-type Interface \
  --service-name com.amazonaws.eu-west-1.ssm \
  --subnet-ids subnet-a subnet-b \
  --security-group-ids sg-endpoint-xxxx \
  --private-dns-enabled
```

Repite para `ssmmessages` y `ec2messages`.

## Paso 5: validar registro en Systems Manager
```bash
aws ssm describe-instance-information \
  --query "InstanceInformationList[].{Id:InstanceId,Ping:PingStatus,Platform:PlatformName}" \
  --output table \
  --region eu-west-1
```

Estado esperado: `PingStatus = Online`.

## Paso 6: abrir sesion shell desde CLI
```bash
aws ssm start-session --target i-0123456789abcdef0 --region eu-west-1
```

## Paso 7: port forwarding (opcional)
### A un puerto local de la propia instancia
```bash
aws ssm start-session \
  --target i-0123456789abcdef0 \
  --document-name AWS-StartPortForwardingSession \
  --parameters '{"portNumber":["3306"],"localPortNumber":["13306"]}' \
  --region eu-west-1
```

### A un host remoto (ejemplo RDS privado)
```bash
aws ssm start-session \
  --target i-0123456789abcdef0 \
  --document-name AWS-StartPortForwardingSessionToRemoteHost \
  --parameters '{"host":["mydb.abcdefghijkl.eu-west-1.rds.amazonaws.com"],"portNumber":["3306"],"localPortNumber":["13306"]}' \
  --region eu-west-1
```

## Troubleshooting rapido
1. No aparece en `describe-instance-information`:
   - Falta rol `AmazonSSMManagedInstanceCore`.
   - Falta salida 443 o endpoints VPC.
   - Agente SSM no disponible/no iniciado.
2. `AccessDenied` en `start-session`:
   - Faltan permisos IAM del operador.
3. `TargetNotConnected`:
   - La instancia no llega a `ssm`, `ssmmessages` o `ec2messages`.
4. Error de plugin:
   - Session Manager Plugin no instalado o desactualizado.

## Automatizacion
Se incluye script PowerShell complementario:
- `aws-session-manager-setup.ps1`

Este script valida entorno, crea/asegura rol y profile, opcionalmente crea endpoints, valida estado SSM y puede iniciar sesion.