# AUTO-VPS-OCI

Guía paso a paso para crear una VM `VM.Standard.A1.Flex` (ARM, free tier) en Oracle Cloud Infrastructure usando `oci-arm-catcher`.

## Requisitos

- Cuenta OCI (Always Free)
- OCI CLI instalado
- Una VM existente 24/7 donde correr el catcher (p.ej. sao-1a, sao-1b)
- Git

## 1. Preparar el catcher en la VM

SSH a la VM que esté encendida (ej: `sao-1b`):

```bash
ssh -i ruta/a/tu-key ubuntu@<IP_VM>
```

### 1.1 Instalar OCI CLI

```bash
# Crear virtualenv e instalar
rm -rf ~/lib/oracle-cli
python3 -m venv ~/lib/oracle-cli
~/lib/oracle-cli/bin/pip install --upgrade pip
~/lib/oracle-cli/bin/pip install oci-cli
```

Agregar al `~/.bashrc`:

```bash
echo 'export PATH=$HOME/lib/oracle-cli/bin:$PATH' >> ~/.bashrc
echo 'export OCI_CLI_SUPPRESS_FILE_PERMISSIONS_WARNING=True' >> ~/.bashrc
source ~/.bashrc
```

### 1.2 Configurar OCI

```bash
mkdir -p ~/.oci
```

Crear `~/.oci/config`:

```ini
[DEFAULT]
user=ocid1.user.oc1..<USER_OCID>
fingerprint=<FINGERPRINT>
tenancy=ocid1.tenancy.oc1..<TENANCY_OCID>
region=<REGION>
key_file=~/.oci/<api-key>.pem
```

Copiar la API key desde tu PC:

```bash
scp -i tu-key ruta/local/api-key.pem ubuntu@<IP_VM>:~/.oci/api-key.pem
```

### 1.3 Clonar oci-arm-catcher

```bash
git clone https://github.com/alexpua/oci-arm-catcher.git ~/oci-arm-catcher
```

### 1.4 Configurar .env

Crear `~/oci-arm-catcher/.env`:

```bash
COMPARTMENT_ID=ocid1.tenancy.oc1..<TENANCY_OCID>
AVAILABILITY_DOMAIN=<AD>  # ej: PrTR:SA-SAOPAULO-1-AD-1
SUBNET_ID=ocid1.subnet.oc1..<SUBNET_OCID>
IMAGE_ID=ocid1.image.oc1..<IMAGE_OCID>
DISPLAY_NAME=<NOMBRE_VM>
OCPUS=1
MEMORY_GB=6
SSH_KEY_FILE=~/.ssh/authorized_keys
RETRY_INTERVAL=120
```

### 1.5 Parchear el script (si oci no está en PATH)

```bash
sed -i '172s|.*|export PATH=$HOME/lib/oracle-cli/bin:$PATH|' \
  ~/oci-arm-catcher/oci-arm-catcher.sh
```

### 1.6 Iniciar el catcher

```bash
cd ~/oci-arm-catcher
nohup bash ./oci-arm-catcher.sh > catcher.log 2>&1 &
```

### 1.7 Monitorear

```bash
tail -f ~/oci-arm-catcher/catcher.log
```

## 2. Obtener los parámetros necesarios

### Tenancy OCID

```bash
cat ~/.oci/config | grep tenancy
```

### Availability Domain

```bash
oci iam availability-domain list --region <REGION>
```

### Subnet OCID

```bash
oci network subnet list --compartment-id <TENANCY_OCID> --region <REGION> --all
```

### Image OCID (Ubuntu 24.04 ARM)

```bash
oci compute image list \
  --compartment-id <TENANCY_OCID> \
  --region <REGION> \
  --operating-system "Canonical Ubuntu" \
  --all \
  --query "data[?contains(\"display-name\",'aarch64-2026.04')].{Name:\"display-name\", ID:id}"
```

## 3. Verificar que el catcher está corriendo

```bash
ps aux | grep oci-arm-catcher | grep -v grep
```

Log esperado:

```
=== oci-arm-catcher started ===
Shape: VM.Standard.A1.Flex  |  OCPU: 1  |  RAM: 6GB
[2026-06-23 05:16:45] Attempt #1
  -> InternalError: Out of host capacity.
  ⏳ 1 min remaining...
```

## 4. Posibles errores

| Error | Causa | Solución |
|-------|-------|----------|
| `Out of host capacity` | Sin capacidad A1 | Esperar, el catcher reintenta solo |
| `The connection to endpoint timed out` | Timeout de red | El catcher reintenta solo |
| `oci: command not found` | OCI no está en PATH | Verificar `export PATH` en `.bashrc` |
| `config file not found` | `.env` no existe | Copiar `.env.example` a `.env` |
| `NotAuthorizedOrNotFound` | API key/OCID incorrecto | Verificar `~/.oci/config` |
| `LimitExceeded` | Límite free tier alcanzado | Verificar OCPUs existentes |

## 5. Estrategias para aumentar probabilidad

- **1 OCPU / 6 GB** en lugar de 4/24 (más probable)
- **Rotar Fault Domains** (el catcher no lo hace por defecto, pero se puede parchar)
- **Probar en múltiples regiones** (stgo-1, valpo-1, etc.)
- **Dejar corriendo 24/7** — la capacidad se libera sin horario fijo

## 6. Detener el catcher (cuando se cree la VM)

```bash
pkill -f oci-arm-catcher
```

## Referencias

- [oci-arm-catcher](https://github.com/alexpua/oci-arm-catcher)
- [oracle-arm-grabber](https://github.com/zerodeps-dev/oracle-arm-grabber)
- [oci-free-arm-instance (GitHub Actions)](https://github.com/tammukul/oci-free-arm-instance)
