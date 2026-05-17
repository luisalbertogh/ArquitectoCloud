---
post_title: Comandos utiles y ejemplos de uso de AWS CLI
author1: Luis Alberto Garcia Hernandez
post_slug: aws-cli-comandos-utiles-ejemplos
microsoft_alias: luis.garcia
featured_image: ""
categories:
  - Cloud
  - AWS
  - CLI
tags:
  - AWS
  - AWS CLI
  - IAM
  - S3
  - EC2
  - CloudWatch
ai_note: true
summary: >
  Guia rapida con comandos utiles de AWS CLI para empezar, configurar el
  entorno y ejecutar operaciones comunes contra IAM, S3, EC2 y CloudWatch.
post_date: 2026-05-17
---

# Comandos utiles de AWS CLI

Este fichero recoge una serie de comandos de ejemplo para trabajar con AWS CLI
de forma segura y practica. Los ejemplos usan marcadores como `<REGION>` o
`<NOMBRE_BUCKET>` para que puedas adaptarlos a tu entorno.

## 1. Setup inicial

### Instalar AWS CLI v2

En Windows, la instalacion habitual es desde el instalador oficial de AWS CLI
v2. Si prefieres comprobar si ya esta instalada:

```bash
aws --version
```

### Configurar credenciales y region por defecto

```bash
aws configure
```

El asistente te pedira:

- Access Key ID
- Secret Access Key
- Default region name
- Default output format

Si quieres usar un perfil adicional:

```bash
aws configure --profile <NOMBRE_PERFIL>
```

### Verificar la identidad actual

```bash
aws sts get-caller-identity
```

### Ver perfiles configurados

```bash
aws configure list-profiles
```

### Usar un perfil concreto

```bash
aws s3 ls --profile <NOMBRE_PERFIL>
```

## 2. Comandos basicos de comprobacion

### Ver las regiones disponibles

```bash
aws ec2 describe-regions --output table
```

### Listar buckets S3

```bash
aws s3 ls
```

### Ver la ayuda de cualquier comando

```bash
aws ec2 describe-instances help
```

## 3. Ejemplos utiles de IAM

### Listar usuarios IAM

```bash
aws iam list-users
```

### Listar roles IAM

```bash
aws iam list-roles
```

### Ver politicas adjuntas a un usuario

```bash
aws iam list-attached-user-policies --user-name <NOMBRE_USUARIO>
```

### Crear un usuario IAM

```bash
aws iam create-user --user-name <NOMBRE_USUARIO>
```

### Adjuntar una politica administrada a un usuario

```bash
aws iam attach-user-policy \
  --user-name <NOMBRE_USUARIO> \
  --policy-arn arn:aws:iam::aws:policy/ReadOnlyAccess
```

## 4. Ejemplos utiles de S3

### Crear un bucket

```bash
aws s3 mb s3://<NOMBRE_BUCKET>
```

### Subir un fichero

```bash
aws s3 cp ./archivo.txt s3://<NOMBRE_BUCKET>/
```

### Sincronizar una carpeta local con un bucket

```bash
aws s3 sync ./mi-carpeta s3://<NOMBRE_BUCKET>/mi-carpeta
```

### Listar el contenido de un bucket

```bash
aws s3 ls s3://<NOMBRE_BUCKET>/ --recursive
```

### Descargar un objeto

```bash
aws s3 cp s3://<NOMBRE_BUCKET>/archivo.txt ./archivo.txt
```

## 5. Ejemplos utiles de EC2

### Listar instancias EC2

```bash
aws ec2 describe-instances --region <REGION>
```

### Mostrar solo instancias en ejecucion

```bash
aws ec2 describe-instances \
  --filters "Name=instance-state-name,Values=running" \
  --query "Reservations[].Instances[].{ID:InstanceId,Type:InstanceType,State:State.Name}" \
  --output table
```

### Ver grupos de seguridad

```bash
aws ec2 describe-security-groups --region <REGION>
```

### Lanzar una instancia basica

```bash
aws ec2 run-instances \
  --image-id ami-0123456789abcdef0 \
  --instance-type t3.micro \
  --subnet-id subnet-0123456789abcdef0 \
  --security-group-ids sg-0123456789abcdef0 \
  --count 1 \
  --region <REGION>
```

### Detener una instancia

```bash
aws ec2 stop-instances --instance-ids i-0123456789abcdef0 --region <REGION>
```

## 6. Ejemplos utiles de Lambda

### Listar funciones Lambda

```bash
aws lambda list-functions --region <REGION>
```

### Ver una funcion concreta

```bash
aws lambda get-function --function-name <NOMBRE_FUNCION> --region <REGION>
```

### Invocar una funcion Lambda

```bash
aws lambda invoke \
  --function-name <NOMBRE_FUNCION> \
  --payload '{"mensaje":"hola"}' \
  response.json \
  --region <REGION>
```

### Actualizar el codigo de una funcion

```bash
aws lambda update-function-code \
  --function-name <NOMBRE_FUNCION> \
  --zip-file fileb://function.zip \
  --region <REGION>
```

### Ajustar la configuracion de memoria y timeout

```bash
aws lambda update-function-configuration \
  --function-name <NOMBRE_FUNCION> \
  --memory-size 512 \
  --timeout 30 \
  --region <REGION>
```

## 7. Ejemplos utiles de API Gateway

### Listar APIs REST o HTTP

```bash
aws apigateway get-rest-apis --region <REGION>
```

### Ver detalles de una API REST concreta

```bash
aws apigateway get-rest-api \
  --rest-api-id <REST_API_ID> \
  --region <REGION>
```

### Listar recursos de una API REST

```bash
aws apigateway get-resources \
  --rest-api-id <REST_API_ID> \
  --region <REGION>
```

### Ver metodos de un recurso

```bash
aws apigateway get-method \
  --rest-api-id <REST_API_ID> \
  --resource-id <RESOURCE_ID> \
  --http-method GET \
  --region <REGION>
```

### Desplegar una API REST en una etapa

```bash
aws apigateway create-deployment \
  --rest-api-id <REST_API_ID> \
  --stage-name prod \
  --region <REGION>
```

### Invocar una API mediante su URL

```bash
curl https://<API_ID>.execute-api.<REGION>.amazonaws.com/prod/recurso
```

## 8. Ejemplos utiles de Step Functions

### Listar state machines

```bash
aws stepfunctions list-state-machines --region <REGION>
```

### Ver detalles de una state machine

```bash
aws stepfunctions describe-state-machine \
  --state-machine-arn <STATE_MACHINE_ARN> \
  --region <REGION>
```

### Iniciar una ejecucion

```bash
aws stepfunctions start-execution \
  --state-machine-arn <STATE_MACHINE_ARN> \
  --name prueba-ejecucion \
  --input '{"cliente":"demo"}' \
  --region <REGION>
```

### Ver ejecuciones de una state machine

```bash
aws stepfunctions list-executions \
  --state-machine-arn <STATE_MACHINE_ARN> \
  --region <REGION>
```

### Ver el estado de una ejecucion

```bash
aws stepfunctions describe-execution \
  --execution-arn <EXECUTION_ARN> \
  --region <REGION>
```

## 9. Ejemplos utiles de CloudWatch

### Listar metricas

```bash
aws cloudwatch list-metrics --namespace AWS/EC2 --region <REGION>
```

### Ver alarmas

```bash
aws cloudwatch describe-alarms --region <REGION>
```

### Crear una alarma basica de CPU

```bash
aws cloudwatch put-metric-alarm \
  --alarm-name CPUAlta-EC2 \
  --alarm-description "CPU superior al 80%" \
  --namespace AWS/EC2 \
  --metric-name CPUUtilization \
  --dimensions Name=InstanceId,Value=<INSTANCE_ID> \
  --statistic Average \
  --period 300 \
  --evaluation-periods 2 \
  --threshold 80 \
  --comparison-operator GreaterThanThreshold \
  --region <REGION>
```

## 10. Consejos practicos

- Usa `--profile` cuando trabajes con varias cuentas o entornos.
- Añade `--region <REGION>` para evitar ejecutar comandos en la region por
  defecto por accidente.
- Usa `--output table` o `--output json` segun te convenga revisar resultados
  en terminal o procesarlos despues.
- Antes de lanzar recursos, revisa el coste y elimina lo que ya no uses.

## 11. Limpieza de ejemplo

### Borrar un objeto de S3

```bash
aws s3 rm s3://<NOMBRE_BUCKET>/archivo.txt
```

### Vaciar y eliminar un bucket

```bash
aws s3 rm s3://<NOMBRE_BUCKET> --recursive
aws s3 rb s3://<NOMBRE_BUCKET>
```

### Terminar una instancia EC2

```bash
aws ec2 terminate-instances --instance-ids i-0123456789abcdef0 --region <REGION>
```
