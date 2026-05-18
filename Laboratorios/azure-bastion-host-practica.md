# Azure Bastion Host - Guia practica (CLI de referencia)

## Objetivo
Implementar acceso remoto seguro a maquinas virtuales Azure (Linux/Windows) usando **Azure Bastion**, evitando exponer puertos SSH (22) y RDP (3389) a internet.

## Cuando usar Azure Bastion
- Cuando quieres administrar VMs sin IP publica en cada VM.
- Cuando quieres evitar jumpboxes autogestionados.
- Cuando necesitas acceso seguro desde portal o cliente nativo.

## Requisitos previos
1. Azure CLI instalado y autenticado.
2. Suscripcion activa y permisos para crear recursos.
3. Grupo de recursos y red virtual (nueva o existente).

Comandos base:
```bash
az login
az account show --output table
az account set --subscription "<subscription-id-o-nombre>"
```

## Arquitectura recomendada
- VNet con subred dedicada **AzureBastionSubnet** (nombre obligatorio).
- Azure Bastion con IP publica propia.
- VMs en subredes privadas (sin IP publica, recomendado).
- NSG en VMs sin reglas inbound 22/3389 desde internet.

## Paso 1: variables de trabajo
```bash
LOCATION="westeurope"
RG="rg-bastion-demo"
VNET="vnet-hub-demo"
BASTION_SUBNET_NAME="AzureBastionSubnet"
BASTION_SUBNET_PREFIX="10.0.0.0/26"
VM_SUBNET_NAME="snet-workload"
VM_SUBNET_PREFIX="10.0.1.0/24"
PIP_BASTION="pip-bastion-demo"
BASTION_NAME="bas-hub-demo"
VM_NAME="vm-linux-demo"
ADMIN_USER="azureuser"
NSG_NAME="nsg-workload"
```

## Paso 2: crear grupo de recursos y red
```bash
az group create \
  --name $RG \
  --location $LOCATION

az network vnet create \
  --resource-group $RG \
  --name $VNET \
  --location $LOCATION \
  --address-prefixes 10.0.0.0/16 \
  --subnet-name $BASTION_SUBNET_NAME \
  --subnet-prefixes $BASTION_SUBNET_PREFIX

az network vnet subnet create \
  --resource-group $RG \
  --vnet-name $VNET \
  --name $VM_SUBNET_NAME \
  --address-prefixes $VM_SUBNET_PREFIX
```

> Nota: el nombre `AzureBastionSubnet` es obligatorio para desplegar Bastion.

## Paso 3: crear IP publica para Bastion
```bash
az network public-ip create \
  --resource-group $RG \
  --name $PIP_BASTION \
  --sku Standard \
  --zone 1 2 3
```

## Paso 4: crear Azure Bastion
```bash
az network bastion create \
  --resource-group $RG \
  --name $BASTION_NAME \
  --public-ip-address $PIP_BASTION \
  --vnet-name $VNET \
  --location $LOCATION
```

## Paso 5: crear NSG y VM de prueba (sin IP publica)
### 5.1 NSG sin abrir 22/3389 a internet
```bash
az network nsg create \
  --resource-group $RG \
  --name $NSG_NAME \
  --location $LOCATION
```

Asociar NSG a la subred de workload:
```bash
az network vnet subnet update \
  --resource-group $RG \
  --vnet-name $VNET \
  --name $VM_SUBNET_NAME \
  --network-security-group $NSG_NAME
```

### 5.2 Crear VM Linux sin IP publica
```bash
az vm create \
  --resource-group $RG \
  --name $VM_NAME \
  --image Ubuntu2204 \
  --admin-username $ADMIN_USER \
  --generate-ssh-keys \
  --vnet-name $VNET \
  --subnet $VM_SUBNET_NAME \
  --public-ip-address "" \
  --nsg ""
```

> Nota: aqui se evita NSG administrado automatico en la NIC y se usa el NSG de subred.

## Paso 6: conectar a la VM usando Bastion
### Opcion A: Portal Azure (recomendada para operacion diaria)
1. Abrir la VM en el portal.
2. Ir a **Connect -> Bastion**.
3. Elegir autenticacion (SSH key o usuario/password para Windows).
4. Iniciar sesion.

### Opcion B: Cliente nativo (CLI) para tunel
Puedes crear un tunel local con Bastion y luego usar tu cliente SSH/RDP.

Ejemplo (Linux SSH):
```bash
az network bastion tunnel \
  --resource-group $RG \
  --name $BASTION_NAME \
  --target-resource-id $(az vm show -g $RG -n $VM_NAME --query id -o tsv) \
  --resource-port 22 \
  --port 50022

ssh -i ~/.ssh/id_rsa $ADMIN_USER@127.0.0.1 -p 50022
```

Ejemplo (Windows RDP, referencia):
```bash
az network bastion tunnel \
  --resource-group $RG \
  --name $BASTION_NAME \
  --target-resource-id <vm-windows-resource-id> \
  --resource-port 3389 \
  --port 53389

mstsc /v:127.0.0.1:53389
```

## Verificaciones utiles
```bash
az network bastion show -g $RG -n $BASTION_NAME --output table
az vm show -g $RG -n $VM_NAME --show-details --query "{name:name,privateIps:privateIps,publicIps:publicIps}" -o table
az network vnet subnet show -g $RG --vnet-name $VNET -n $BASTION_SUBNET_NAME --output table
```

## Buenas practicas de seguridad
- No asignar IP publica a VMs de administracion interna.
- No abrir inbound 22/3389 desde internet.
- Usar RBAC minimo para acceso a Bastion y VMs.
- Activar logs/diagnostico y auditoria.
- Usar MFA para cuentas con privilegios.

## Limpieza de recursos (demo)
```bash
az group delete --name $RG --yes --no-wait
```

## Troubleshooting rapido
1. Error al crear Bastion por subred:
   - Verifica que la subred se llame exactamente `AzureBastionSubnet`.
2. No conecta por tunnel:
   - Revisa que Bastion y VM esten en VNets conectadas correctamente.
   - Revisa permisos RBAC sobre VM y Bastion.
3. Tiempo de espera en acceso:
   - Revisa NSG/rutas internas entre Bastion y la subred de la VM.
